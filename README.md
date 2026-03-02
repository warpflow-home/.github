# Docker Swarm HA クラスター（Keepalived + GlusterFS + macvlan/overlay）

このリポジトリは、Ubuntu 24.04 上での 4 ノード構成 Docker Swarm クラスターを構築するための要点まとめです。
目的は Keepalived による VIP（192.168.1.240）を入口として使い、共通サービスは Swarm Ingress（overlay）で、Asterisk は macvlan で物理 LAN に直接接続する構成を提供することです。

**主な構成要素**
- Docker Swarm (4 ノード: 全ノードを manager + worker として利用)
- Keepalived (VIP: 192.168.1.240)
- GlusterFS (Replica 3 の分散ストレージ)
- macvlan ネットワーク (Asterisk 用固定 IP: 192.168.1.200)
- overlay ネットワーク (サービス間通信用: 10.10.0.0/16)

**設計のポイント（要約）**
- Asterisk は SIP/RTP の NAT 問題を避けるために物理 LAN IP (`192.168.1.200`) を利用。
- その他のサービスは VIP `192.168.1.240` を共有し、ポートで振り分ける（例: Portainer 9443）。
- GlusterFS を使ってデータを全ノードに複製し、サービスが他ノードへ移動してもデータを維持。
- Swarm の Raft クォーラムに注意（4 ノードは完全冗長ではない。可能なら 5 ノードを検討）。

**前提（各ノードで実行）**
- Ubuntu 24.04
- root または sudo 権限
- 物理ネットワーク: `192.168.1.0/24`（ゲートウェイ `192.168.1.1`）

簡易 IP 割当（例）:
- node01: `192.168.1.241`
- node02: `192.168.1.242`
- node03: `192.168.1.243`
- node04: `192.168.1.244`
- VIP (Keepalived): `192.168.1.240`
- Asterisk (macvlan): `192.168.1.200`

**クイックスタート（概略コマンド）**

1) 全ノード共通の事前準備（パッケージ、カーネルモジュール、sysctl）

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl gnupg lsb-release keepalived glusterfs-server glusterfs-client ufw
sudo modprobe overlay br_netfilter 8021q
sudo tee /etc/sysctl.d/99-docker-swarm.conf <<'EOF'
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```

2) Docker Engine のインストール（各ノード）

```bash
# Docker 公式セットアップ（省略: 公式手順に従ってください）
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```

3) GlusterFS ボリューム作成（node01 から）

```bash
sudo gluster peer probe node02
sudo gluster peer probe node03
sudo gluster peer probe node04
sudo gluster volume create gfs-appdata replica 3 \
  node01:/data/glusterfs/bricks/appdata \
  node02:/data/glusterfs/bricks/appdata \
  node03:/data/glusterfs/bricks/appdata force
sudo gluster volume start gfs-appdata
```

4) Swarm 初期化（node01）

```bash
sudo docker swarm init --advertise-addr 192.168.1.241 --listen-addr 192.168.1.241:2377 \
  --default-addr-pool 10.10.0.0/16 --default-addr-pool-mask-length 24
```

5) ネットワーク作成

```bash
# macvlan（全ノードで同一コマンド）
IFACE=$(ip route | grep default | awk '{print $5}')
sudo docker network create --driver macvlan --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 --ip-range=192.168.1.200/32 --opt parent=$IFACE swarm-macvlan

# overlay（node01）
sudo docker network create --driver overlay --attachable --subnet=10.10.0.0/16 --gateway=10.10.0.1 swarm-overlay
```

6) Keepalived の設定（各ノードで `/etc/keepalived/keepalived.conf` を編集）

```ini
vrrp_instance VI_1 {
  state MASTER
  interface eth0
  virtual_router_id 51
  priority 100
  advert_int 1
  authentication { auth_type PASS; auth_pass SecretPass123 }
  virtual_ipaddress { 192.168.1.240/24 }
}
```

7) サービスデプロイ例

- Portainer（VIP 経由、Ingress ポートを公開）
- Asterisk（macvlan、`192.168.1.200` を固定）

詳しいスタックファイルと設定例は元ガイドを参照してください（Asterisk の `bindaddr`/`externip` は `192.168.1.200` を指定すること）。

**運用上の注意**
- Swarm の manager 数と Raft クォーラム: 4 ノードは 2 台障害に耐えられません。可能なら manager を奇数にする（例: 5 ノード）を検討してください。
- macvlan を使う際はホスト⇄コンテナ間のルーティングを明示的に作成し、IP の二重割当を避ける。
- Keepalived のテスト: VIP を持つノードで `sudo systemctl stop keepalived` を実行し、VIP が別ノードに移ることを確認する。

**トラブルシュートのヒント**
- GlusterFS: `gluster peer status`, `gluster volume status` を確認
- Swarm: `docker node ls`, `docker service ps <service>`
- Keepalived: `ip addr show` で VIP の配置確認

**デプロイチェックリスト（簡易）**
- 全ノード: ホスト名、/etc/hosts、必要パッケージ、sysctl、UFW
- GlusterFS: ボリューム作成・起動・マウント
- Swarm: 全ノードが manager として参加
- ネットワーク: `swarm-overlay`, `swarm-macvlan` の作成
- サービス: Portainer, Asterisk 等のスタックデプロイ

---

詳細な手順・設定例は添付のドキュメント（Docker Swarm HA クラスター構築ガイド）を参照してください。

もし README の修正や追加で「デプロイ手順を自動化するスクリプト」や「スタックファイルのテンプレート」を作成したい場合は、次にやることを教えてください。
