# Codens VPS — Network Access & Defense

Codens VPS (24 台) は **インターネットから直接 port を叩けない** 設計。メンバーが
`docker run -p` や `docker compose` で port を公開しても外部到達不能。

これは複数層の defense-in-depth で実現していて、1 層が drift / 誤設定されても
他の層が遮断するように作ってあります。

## 3 層防御

```
                    [ Internet / 外部攻撃者 ]
                              │
                              ▼
        ┌─────────────────────────────────────────┐
        │  Layer 1: Contabo Cloud Firewall        │
        │  (network 層、VM の外側で動く)           │
        │  default deny + tcp/22 from operator IP │
        └─────────────────────────────────────────┘
                              │
                              ▼ (もし drift / 誤設定で抜けた場合)
                              │
        ┌─────────────────────────────────────────┐
        │  Layer 2: iptables DOCKER-USER chain    │
        │  (host 層、systemd で永続化)             │
        │  external NIC → container を DROP       │
        │  (compose の 0.0.0.0 bind も block)     │
        └─────────────────────────────────────────┘
                              │
                              ▼ (もし iptables が flush された場合)
                              │
        ┌─────────────────────────────────────────┐
        │  Layer 3: Docker daemon "ip"            │
        │  (Docker 設定層)                         │
        │  docker run -p 8080:8080 → 127.0.0.1   │
        │  (compose は bypass する点に注意)        │
        └─────────────────────────────────────────┘
                              │
                              ▼
                    [ コンテナ port (127.0.0.1) ]
```

### Layer 1: Contabo Cloud Firewall

- 場所: VM の外側 (Contabo の hypervisor / ネットワーク perimeter)
- 設定: `cntb get firewall <id>` または Contabo Web UI
- ルール:
  - `accept tcp/22 from <ops-server-ip>/32` (= operator VPS)
  - `drop all` (default-deny)
- 影響範囲: **すべての port (22 以外)、すべての NIC**

**既知のリスク:** instance 割り当てが Contabo 側で wipe された前例がある (memory
`project_contabo_firewall_drift.md`、2026-04-23 `noOfInstances=0`)。Layer 1 単独
では fragile なので Layer 2/3 を併設している。

### Layer 2: iptables DOCKER-USER chain

- 場所: VPS host (Linux iptables の `filter` table、`FORWARD` chain 内)
- 実装: `roles/docker-firewall/` (`codens-docker-firewall.service` が systemd
  oneshot で rule 投入)
- ルール (`/usr/local/bin/codens-docker-firewall start` が投入する):
  ```
  iptables -I DOCKER-USER -i eth0 -m conntrack --ctstate RELATED,ESTABLISHED -j RETURN
  iptables -I DOCKER-USER -i eth0 -j DROP
  ```
- 効果:
  - 外部 → container 新規接続: **DROP** (`docker run -p 0.0.0.0:8080:8080` でも
    `docker compose` の `"8080:8080"` でも block)
  - 外部 → container ESTABLISHED/RELATED: **RETURN** (コンテナ outbound の応答
    packet を許可)
  - **loopback (127.0.0.1)・bridge interface (br-xxx)・SSH tunnel: 影響なし**

**なぜ Layer 3 だけでは足りないか:** docker-compose は `ports:` field を解釈して
**明示的に `0.0.0.0` を Docker API に送る** ため、daemon の `"ip"` 設定 (Layer 3)
を bypass する。実機検証 (3 patterns) で全部 0.0.0.0 bind になることを確認済。
Layer 2 が必要。

**永続化:** systemd unit が `After=docker.service` で起動 → reboot 時も
docker.service の後に rule を再投入。Docker daemon は `DOCKER-USER` chain を
触らない (それが Docker 公式の reserved chain) ので daemon 再起動でも rule は
維持される。

### Layer 3: Docker daemon `"ip": "127.0.0.1"`

- 場所: `/etc/docker/daemon.json`
- 実装: `roles/docker-limits/`
- 効果: `docker run -p 8080:8080` の bind を `127.0.0.1:8080` に強制 (デフォルト
  `0.0.0.0` を localhost に変更)
- 限界: docker-compose が **明示的に 0.0.0.0 を送る** ので compose 経由には効かない
  (Layer 2 で補完)

このレイヤーの主目的は「メンバーが事故で `-p 8080:8080` 書いた時に external bind
される事故を防ぐ」こと (compose ではない通常の `docker run` はカバーされる)。

## メンバーがサービスにアクセスする方法 (closed-by-default に対する正規ルート)

外部到達はすべて遮断されているので、メンバーは以下のいずれかでアクセスする:

| 経路 | 仕組み | 主な用途 |
|---|---|---|
| **`ssh -L`** | OpenSSH の TCP forwarding | dev server を laptop ブラウザで開く |
| **VS Code Remote-SSH の Forwarded Ports** | SSH tunnel + VS Code の自動検出 | dev 中の port を 1-click forward |
| **Termius / Blink の Port Forwarding** | SSH tunnel (GUI) | スマホから dev server を見る |
| **code-server (`https://vps-<slug>.<domain>`)** | Cloudflare Tunnel + browser VS Code | ブラウザだけで完結する dev 環境 |
| **Cloudflare Tunnel ingress (operator が追加)** | Cloudflare Tunnel + CF Access auth | webhook receiver 等の長期公開 URL |

**SSH tunnel と Cloudflare Tunnel は別物**:
- SSH tunnel = OpenSSH の TCP forwarding。member が ad-hoc で張る。
- Cloudflare Tunnel = `cloudflared` daemon が Cloudflare edge に常時接続している
  outbound-only の tunnel。`vps-<slug>.<domain>` / `ssh-<slug>.<domain>` も
  全部これ越し。CF Access で email 認証付き。

どちらも **VPS の外部 NIC を一切経由しない** (cloudflared は outbound-only
connection で edge と繋がってる、ssh は operator IP から allow されてる port 22)
ので、Layer 1/2/3 とは関係なく動く。

## "Closed by default" は何が closing しているか

「sshトンネル以外でポートが閉じる仕組み」の問いに対する答え:

- ポートを **「閉じている」のは Layer 1/2/3** (上記の 3 段) であって、SSH tunnel は
  「閉じる仕組み」ではなく「閉じた状態のままアクセスする経路」
- Linux / Docker のデフォルト挙動では **port は開いてしまう** (Docker は
  `0.0.0.0` に bind するし、Linux 側に external 遮断の自動機構は無い)
- なので 3 段防御を **明示的に設定する** ことで closed-by-default を実現している

「SSH tunnel 以外の閉じる仕組み」を増やしたい場合は以下が候補:
- ufw default-deny を有効化 (現在 inactive、`11-sshd-lockdown.yml` 実装時に対応予定)
- sshd を `127.0.0.1:22` bind に変更 + Cloud Firewall で 22 も close
  (`11-sshd-lockdown.yml` 実装時に対応予定)
- AppArmor profile による network 制限 (現在未導入)

ただし上記は「**閉じる**」を増やすもので、現在も既に外部到達は不能。追加した
くなる動機としては:
1. internal lateral movement の防止 (一台落ちても他台に攻撃が伝播しない)
2. 万が一の supply chain 攻撃 (悪意ある package が beacon してくる) を抑える

これらは現在の Codens VPS の脅威モデル外なので、優先度は低い (要検討は OQ レベル)。

## 検証コマンド

```bash
# Layer 1: Cloud Firewall ルール
python3 -c "
import yaml, subprocess, json
acc = yaml.safe_load(open('~/.contabo-accounts.yaml'))['accounts']['account-a']
flags = [f'--oauth2-clientid={acc[\"client_id\"]}', f'--oauth2-client-secret={acc[\"client_secret\"]}', f'--oauth2-user={acc[\"user\"]}', f'--oauth2-password={acc[\"password\"]}']
res = subprocess.run(['cntb', 'get', 'firewall', '<your-firewall-uuid>', '--output', 'json', *flags], capture_output=True, text=True)
print(json.loads(res.stdout)[0]['rules'])
"

# Layer 2: iptables DOCKER-USER chain (sample VPS)
ssh ansible@<vps> 'sudo iptables -L DOCKER-USER -n -v --line-numbers'

# Layer 3: docker daemon ip
ssh ansible@<vps> 'sudo grep "ip" /etc/docker/daemon.json'

# 動作テスト: compose で公開試行 → 外部から到達不能
ssh ansible@<vps> 'cd /tmp && cat > test.yml <<YAML
services:
  web:
    image: nginx:alpine
    ports: ["18099:80"]
YAML
sudo docker compose -f test.yml up -d
sudo ss -tlnp | grep 18099  # → 0.0.0.0:18099 (compose は bypass)
'
# operator ホストから:
curl -m 5 http://<vps-public-ip>:18099   # → timeout (Layer 1/2 が block)
# clean up:
ssh ansible@<vps> 'sudo docker compose -f /tmp/test.yml down'
```

## 関連ファイル

| Path | 内容 |
|---|---|
| `roles/docker-limits/tasks/main.yml` | Layer 3 (`daemon.json` `"ip": "127.0.0.1"`) |
| `roles/docker-firewall/files/codens-docker-firewall` | Layer 2 (iptables DOCKER-USER 投入スクリプト) |
| `roles/docker-firewall/files/codens-docker-firewall.service` | Layer 2 (systemd unit、reboot 時の永続化) |
| `roles/docker-firewall/tasks/main.yml` | Layer 2 (Ansible task) |
| `playbooks/55-security-roundup.yml` | Layer 2/3 を全 VPS に push する entrypoint |
| `terraform/cloudflare/tunnels.tf` | Cloudflare Tunnel ingress 定義 (公開時の追加先) |
| `roles/cloudflare-tunnel/` | cloudflared daemon の設置 (= 内向き tunnel の実体) |

## トラブルシューティング

| 症状 | 原因と対処 |
|---|---|
| メンバーが "service にアクセスできない" と言う | 外部 IP を直接叩いてる可能性。SSH tunnel か code-server を案内 (welcome kit / `docs/mobile-ssh-guide.md`) |
| `iptables -L DOCKER-USER` が空になっている | `systemctl restart codens-docker-firewall` で再投入。docker daemon が完全停止していると失敗するので docker.service 確認 |
| docker daemon が起動失敗 | `daemon.json` syntax error 可能性。`docker daemon --validate` または `systemctl status docker -l` |
| compose で 0.0.0.0 でも block されない | `iptables -L DOCKER-USER -v` で rule 順序確認。`-i eth0 -j DROP` が一番上にあるはず。なければ `codens-docker-firewall start` を再実行 |
| `ssh -L` で port forward できない | sshd の `AllowTcpForwarding` 設定が `no` になっていないか確認 (default は `yes`) |
| 一時的に external 公開したい | Cloudflare Tunnel ingress を `terraform/cloudflare/tunnels.tf` に追加 → `terraform apply` |
