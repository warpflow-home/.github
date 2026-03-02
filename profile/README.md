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


**サービスデプロイ例**

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
