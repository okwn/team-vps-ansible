# team-vps-ansible

Ansible + Terraform for provisioning a fleet of per-developer Linux VPSs with:

- Zero Trust SSH (Cloudflare Tunnel + Access)
- Browser-based VS Code (code-server) per user
- Identity-aware logging (auditd + S3 via AWS Roles Anywhere — no static keys on hosts)
- WARP private network so all VPSs share a 10.200.0.0/24 fabric
- Supply-chain attack early detection for npm + VS Code extensions
- Hardened SSH, fail2ban, unattended-upgrades, sysctl tweaks

The pattern is opinionated: one VPS per member, member is sudoer on their own
box, ops team manages everything via Ansible from a control workstation.

## Why this exists

Onboarding ~25 developers, each with an isolated dev environment that:

- Doesn't share state across team members (no shared bastion / shared shell)
- Is reachable from mobile (phone, tablet) without a VPN client
- Has identity-tagged audit trail (which user ran what, when, where)
- Costs predictable money (~$5-15/mo per VPS on Contabo)
- Can be re-provisioned from scratch in <30 min if anything breaks

If your situation matches the above, this should adapt cleanly. If you have
1-2 devs, this is overkill — use Tailscale SSH directly.

## Stack

- **Compute**: Contabo Cloud VPS (cheap, multi-account capable)
- **Network access**: Cloudflare Zero Trust (Tunnel + Access policies)
- **Editor**: code-server (browser VS Code), Workspace Trust enforced
- **Logging**: auditd → /var/log/audit/ → S3 via AWS IAM Roles Anywhere
- **Secrets**: ansible-vault for at-rest, AWS Secrets Manager + Roles Anywhere
  for runtime cred broker (no static AWS keys on VPSs)
- **CA**: step-ca for TLS cert issuance (Roles Anywhere trust anchor)
- **PKI**: per-host SSH ed25519 keypairs generated locally, distributed via
  welcome-kit tgz

## Repo layout

```
inventory/         # hosts.yml.example + per-host vars (gitignored)
group_vars/        # cluster-wide vars (vault-encrypted)
roles/             # ansible roles, see below
playbooks/         # numbered phase playbooks (00-bootstrap, 10-harden, ...)
terraform/
  contabo/         # VPS provisioning + firewall
  cloudflare/      # Zero Trust tunnels, Access policies, DNS
  aws/             # SES, IAM, Roles Anywhere, S3 log bucket
scripts/           # bootstrap + maintenance helpers
docs/              # operator runbooks
site.yml           # aggregate playbook (most days you'd run individual phases)
```

### Roles

| Role | Purpose |
|------|---------|
| `harden` | sysctl, unattended-upgrades, auditd base rules, fail2ban |
| `users` | per-VPS owner user, sudoers, SSH key install |
| `dev-tools` | git, build-essentials, mise, common CLI |
| `chrome-playwright` | headless browser for automation |
| `claude-autoupdate` | Anthropic Claude CLI auto-updater |
| `claude-config` | per-user Claude config, hooks, scripts |
| `cloudflare-tunnel` | cloudflared install + token wiring |
| `code-server` | browser VS Code, bind 127.0.0.1:8080, exposed via CF Tunnel |
| `common` | base packages |
| `disk-quota` | per-user disk quotas |
| `docker-firewall` | block docker from publishing on eth0 (only 127.0.0.1) |
| `docker-limits` | cgroup limits for docker containers |
| `log-shipping` | auditd → S3 via Roles Anywhere, hourly timer |
| `metadata` | host facts collection for ops dashboard |
| `monitoring` | system metrics + postfix for outbound alerts |
| `warp-vip` | bind 10.200.0.X/32 on lo for WARP private routing |
| `supply-chain-monitor` | auditd watches for npm/VS Code supply-chain attacks |
| `aws-broker` | wrapper script — issues short-lived STS creds via Roles Anywhere |
| `purple-agent-qwen-env` | (optional) env for self-hosted Qwen LLM agent service |

## Quick start

### Prerequisites

- [Terraform](https://terraform.io) >= 1.5
- [Ansible](https://ansible.com) >= 2.15
- Python 3.11+ with `pip install ansible-core hvac`
- Contabo account with API access
- Cloudflare account with a zone configured
- AWS account for Roles Anywhere and S3 log bucket

### Bootstrap a new VPS

```bash
# 1. Provision infrastructure
cd terraform/contabo
terraform init
terraform plan -var-file=../../inventory/prod.tfvars
terraform apply

# 2. Collect the new host IP from terraform output

# 3. Add the host to inventory
cp inventory/hosts.yml.example inventory/hosts.yml
# Edit hosts.yml with the new IP

# 4. Run the phase playbooks in order
ansible-playbook playbooks/00-bootstrap.yml -i inventory/hosts.yml --limit new-host
ansible-playbook playbooks/10-harden.yml -i inventory/hosts.yml --limit new-host
ansible-playbook playbooks/20-dev-tools.yml -i inventory/hosts.yml --limit new-host
# ... continue with remaining phase playbooks

# 5. Distribute welcome-kit to the new user
# The ops team sends the tgz from scripts/welcome-kit.sh to the new developer
```

### Prerequisites

- Cloudflare account with Zero Trust enabled (free tier OK up to 50 users)
- One AWS account for SES + IAM Roles Anywhere + S3 log bucket
- Domain name (you'll create `vps-<name>.<your-domain>` per member)
- Contabo account (or any provider — non-Contabo needs a different terraform module)
- Operator workstation (Ubuntu/Mac) with `ansible`, `terraform`, `gh`, `aws`
- Initial VPSs (one per developer) provisioned with Ubuntu 22.04 / 24.04 LTS

### Setup

1. **Fill in `members.yml`** from `members.yml.example` (real personal data —
   gitignored, never commit)

2. **Generate inventory**:

   ```bash
   scripts/generate-inventory.py
   ```

   produces `inventory/hosts.yml` + `inventory/host_vars/*.yml` (also gitignored).

3. **Bootstrap each VPS** (one-time, runs locally on the VPS via `curl | sh`):

   ```bash
   scripts/pki-bootstrap.sh <vps-slug>
   ```

   installs step-ca client cert so Ansible can auth via mutual TLS afterward.

4. **Apply terraform** to create Cloudflare tunnels + DNS + Access policies:

   ```bash
   cd terraform/cloudflare
   cp terraform.tfvars.example terraform.tfvars  # fill in real values
   terraform init && terraform apply
   ```

5. **Run Ansible**:

   ```bash
   ansible-playbook site.yml --limit vps-yourname
   ```

   site.yml runs all phases. For first-run, you'd typically run them in order:

   ```bash
   for p in playbooks/*.yml; do
     ansible-playbook "$p" --limit vps-yourname
   done
   ```

6. **Generate welcome kit** (tgz with onboarding docs, SSH config, etc.):

   ```bash
   scripts/generate-welcome-kits.sh <vps-slug>
   ```

   Send the kit to the developer; they follow `docs/mobile-ssh-guide.md`.

## Security notes

- **Never commit** `members.yml`, `inventory/hosts.yml`, `inventory/host_vars/*.yml`,
  `*.tfvars`, `*.tfstate*`, `.vault_pass`, `keys/`, `secrets/`, `dist/` —
  all gitignored by default.

- **Supply-chain hardening** is in `roles/supply-chain-monitor` —
  detect-don't-block by design (no impact on dev experience).

- **Credentials are never static** on VPSs after bootstrap. The `aws-broker`
  role issues short-lived STS credentials via Roles Anywhere on each call.

- **Per-user audit trail**: each member has a real Linux user (no shared
  `ubuntu` account). auditd tags every syscall with the uid, which ships to
  S3 hourly.

## Adapting to your environment

Things you'll definitely need to change:

- `inventory/group_vars/all/public.yml` — your domain, CF account, AWS account
- `terraform/*/variables.tf` defaults — your org info
- `members.yml.example` — your team
- VPS provider: this repo has `terraform/contabo/` modules. For Hetzner / DO /
  Vultr, write a sibling module in `terraform/<provider>/` and adjust
  `scripts/fetch-*-instance-ids.py` to use that provider's API.

## License

MIT — see [LICENSE](LICENSE).

Contributions welcome. This was built for one specific team's workflow; if
your edge case is different but the core pattern fits, open an issue.
