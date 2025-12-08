# Consul 連携
Consul Enterprise では、Sentinel と連携することにより ACL Policy を拡張し、より高度なガバナンスを実現することが可能です。 \
Consul はサービスメッシュを中心として数多くの機能を持ちますが、そのうちの一つである KVS（Key-Value Store）に対する ACL へのアクセスにおいて、Sentinel は強力に機能します。

Consul KVS では Sentinel の Enforcement Level を完全にサポートしており、ACL Policy 定義時に、従来の HCL ベースのポリシーだけでなく、\
Sentinel をベースとしたポリシーコードを適用することで、KVS に対するより高度なポリシー制御を行うことが可能となります。

ここでは、実際に Consul を KVS として起動し、特定パスへの Key-Value データ書き込み時に Sentinel ポリシーによる評価が行われることを実際に試してみましょう。

まず、Consul サーバを起動します。\
Consul における Sentinel 連携は、Consul がデフォルトで持つ ACL Policy 制御を前提としているため、Consul 起動時に ACL Policy を利用できる形で起動しておく必要があります。 \
Consul サーバの設定ファイルを作成し、Consul Agent をサーバとして起動します。

```shell
% mkdir -p .consul-data
% cat >> consul.hcl <<EOF

EOF

% consul agent -config-file=consul.hcl
# ...
```

ACL Policy を有効化した Consul サーバでは `CONSUL_HTTP_TOKEN`（HTTP Header の場合 `X-Consul-Token`）によってクライアントからのリクエストが Token ベースで制御される形となります。\
管理者としてのアクセスのために、Bootstrap を行うことにより初期トークンを払い出します。

```shell
% consul acl bootstrap
AccessorID:       ee3d777e-b8cb-fbd8-c6cd-df7f7172131f
SecretID:         b0ac54cb-26a4-1966-aebc-27a8af66a388
Partition:        default
Namespace:        default
Description:      Bootstrap Token (Global Management)
Local:            false
Create Time:      2025-12-08 22:00:32.926766 +0900 JST
Policies:
   00000000-0000-0000-0000-000000000001 - global-management

# 払い出された SecretID を Token としてセットします
% export CONSUL_HTTP_TOKEN='b0ac54cb-26a4-1966-aebc-27a8af66a388'
```

以降、Sentinel ポリシーを Consul サーバへ適用するオペレーションまでは `CONSUL_HTTP_TOKEN` として Bootstrap Token がセットされているものとして進めます。\
（実際に、Sentinel ポリシーコードの挙動を確認する際には、異なる Token を払い出し、Consul KVS へのアクセス確認を行います） \
また、Consul の Sentinel 連携には Enterprise ライセンスが必要となるため、機能が有効になっていることを確認します。

```shell
# Governance and Policy が Module として含まれていることを確認する
% consul license get
License is valid
# ...
Datacenter: *
Modules:
	Global Visibility, Routing and Scale
	Governance and Policy
Licensed Features:
	Automated Backups
	Automated Upgrades
	Enhanced Read Scalability
	Network Segments
	Redundancy Zone
	Advanced Network Federation
	Namespaces
	SSO
	Audit Logging
	Admin Partitions
	Service Mesh
```

次に、KVS へデータを投入するクライアントの Token を発行し、付与する ACL Policy を作成したのちに Token に対して ACL Policy を付与します。\
ここで付与するポリシーは Consul が標準で持つ ACL Policy であり、KVS への書き込み権限を与えるためのものになる点に注意してください。 \
(このクライアントが、特定 KVS パスへの書き込み時に評価されるポリシーについては、Sentinel で後続のステップで実装します)

```shell


```

## 参考リンク
- [Enforce ACL Policies with Sentinel](https://developer.hashicorp.com/consul/docs/secure/acl/sentinel)
