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
license_path = "consul.hclic"

datacenter = "dc1"
data_dir = ".consul-data/"
node_name = "consul-server"
server = true

addresses {
  http = "0.0.0.0"
}
bootstrap = true
bootstrap_expect = 1

acl {
  enabled = true
  default_policy = "deny"
}
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

次に、KVS へデータを投入するためにクライアントの Token を発行し、付与する ACL Policy を作成したのちに Token に対して ACL Policy を付与します。\
ここで付与するポリシーは Consul が標準で持つ ACL Policy であり、KVS への書き込み権限を与えるためのものになる点に注意してください。 \
(このクライアントが、特定 KVS パスへの書き込み時に評価されるポリシーについては、Sentinel で後続のステップで実装します)

```shell
# redis/config/* に write 操作を許可する ACL Policy を作成
% cat >> kvs-redis-config.hcl <<EOF
key_prefix "redis/config/" {
	policy = "write"
}

partition_prefix "" {
	namespace "default" {
		node_prefix "" {
			policy = "write"
		}
		agent_prefix "" {
			policy = "write"
		}
	}
}
EOF

# 上記の HCL ファイルを Consul にポリシーとして適用します
% consul acl policy create -name="redis-config-writer" \
> -description="Allow write operations under redis/config/" \
> -rules=@kvs-redis-config.hcl

ID:           7383416d-d7e6-e2ab-cd5f-4a84ad483442
Name:         redis-config-writer
Partition:    default
Namespace:    default
Description:  Allow write operations under redis/config/
Datacenters:
Rules:
key_prefix "redis/config/" {
	policy = "write"
}
# ...
```

作成した `kvs-redis-writer` ポリシーがアタッチされた Consul クライアント用の Token を新規に生成し、この Token を用いてポリシー通りに `redis/config/*` のパスに書き込みができることを確認します。\
ACL Policy 作成時にポリシーに付与された UUID（`policy-id`）を指定し、クライアントアクセス用の Token を作成します。\
また、Consul KVS に登録した内容を確認できるよう、デフォルトで作成されている `builtin/global-read-only` ポリシーも合わせて付与します。

```shell
% consul acl token create -description="Limited client access of Consul KVS" \
-policy-id=15c7cdf1-d0b0-582d-2d6e-dd6b7ef572ba \
-policy-id=00000000-0000-0000-0000-000000000002

AccessorID:       9267d017-94dd-5b72-9c91-083c651c48aa
SecretID:         2ac79c47-fb43-e7f5-2371-c7950c79d594
Partition:        default
Namespace:        default
Description:      Limited client access of Consul KVS
Local:            false
Create Time:      2025-12-08 23:55:20.129244 +0900 JST
Policies:
   7383416d-d7e6-e2ab-cd5f-4a84ad483442 - redis-config-writer
   00000000-0000-0000-0000-000000000002 - builtin/global-read-only
```

Consul クライアント用に作成された Token の SecretID に切り替え、Consul KVS へ書き込みを行います。\
ここでは ACL Policy が適用され、write/read（put/get）がいずれもできることを確認します。

```shell
# クライアント用に作成した Token に切り替える
% export CONSUL_HTTP_TOKEN='2ac79c47-fb43-e7f5-2371-c7950c79d594'

# Consul KVS にデータを投入し、確認する
% consul kv put redis/config/version "7.4.2"
% consul kv get -detailed redis/config/version
```

併せて、ACL Policy で許可されていない KVS パスへの書き込みには失敗することも確認できるかと思います。

```shell
% consul kv put mysql/config/version 8.0
```

そして、再び Consul 管理者用の Bootstrap Token に切り替え、Sentinel による Policy を Consul サーバに適用します。\
`key` により特定 KVS パスへの `write` 時の処理という点については同様ですが、`sentinel` ブロックが追加されていることがわかるかと思います。

ここでは、`main` ルールとして `value == "1gb"` が定義されています。\
これは、Sentinel コード内で、Consul KVS アクセス時にクライアントが指定した `value` の値を評価し、その値が `1gb` であれば PASS, そうでない場合には FAIL とするポリシーとなります。 \
Sentinel と Consul の連携により、Sentinel コードから、Consul KVS のコンテキストにアクセスすることが可能になっています。\
（例では Consul KVS への write 時の `value` を Sentinel により評価しましたが、他にも[ドキュメント記載](https://developer.hashicorp.com/consul/docs/secure/acl/sentinel#injected-variables)の通り、`key` や `flags` の値をポリシー評価に利用することも可能です）

```shell
# Bootstrap Token に切り替える
% export CONSUL_HTTP_TOKEN='821bc899-1acc-90ec-8923-f0552da482f6'

% cat >> ensure-kvs-redis-maxmemory.hcl <<EOF
key "redis/config/maxmemory" {
  policy = "write"
  sentinel {
    enforcementlevel = "hard-mandatory"
    code = <<EOC
      main = rule {
        value = "1gb"
      }
    EOC
  }
}
EOF

# 上記の Sentinel Policy を適用する
% consul acl policy create -name="ensure-redis-maxmemory" \
-description="Ensure Redis maxmemory under redis/config/maxmemory" \
-rules=@ensure-kvs-redis-maxmemory.hcl

ID:           15c7cdf1-d0b0-582d-2d6e-dd6b7ef572ba
Name:         ensure-redis-maxmemory
Partition:    default
Namespace:    default
Description:  Ensure Redis maxmemory under redis/config/maxmemory
Datacenters:
Rules:
key "redis/config/maxmemory" {
  policy = "write"
  sentinel {
    enforcementlevel = "hard-mandatory"
    code = <<EOC
      main = rule {
        value == "1gb"
      }
    EOC
  }
}
```

そして、先ほど払い出したクライアントアクセス用の Token にこの Sentinel Policy を追加で紐付けます。\
払い出し済みの Token に対する操作を行う場合には、Token 作成時に出力された AccessorID を利用します。

```shell
% consul acl token update -accessor-id=9267d017-94dd-5b72-9c91-083c651c48aa \
-description="Limited client access of Consul KVS" \
-policy-id=00000000-0000-0000-0000-000000000002 \
-policy-id=7383416d-d7e6-e2ab-cd5f-4a84ad483442 \
-policy-id=15c7cdf1-d0b0-582d-2d6e-dd6b7ef572ba

AccessorID:       9267d017-94dd-5b72-9c91-083c651c48aa
SecretID:         2ac79c47-fb43-e7f5-2371-c7950c79d594
Partition:        default
Namespace:        default
Description:      Limited client access of Consul KVS
Local:            false
Create Time:      2025-12-08 23:55:20.129244 +0900 JST
Policies:
   15c7cdf1-d0b0-582d-2d6e-dd6b7ef572ba - ensure-redis-maxmemory
   00000000-0000-0000-0000-000000000002 - builtin/global-read-only
   7383416d-d7e6-e2ab-cd5f-4a84ad483442 - redis-config-writer
```

再びクライアントアクセス用の Token に切り替え、`redis/config/maxmemory` に Key-Value データを put し、値に応じて挙動を確認してみましょう。\
最初に、Sentinel ポリシーが PASS となるケース、つまり `redis/config/maxmemory="1gb"` を write してみます。

```shell
% export CONSUL_HTTP_TOKEN='2ac79c47-fb43-e7f5-2371-c7950c79d594'
% consul kv put redis/config/maxmemory 1gb
Success! Data written to: redis/config/maxmemory
```

上記の様に、問題なく put することができたかと思います。\
つづいて、この `1gb` という値を別の値に変更する場合を考えます。\
この時、Consul KVS の `redis/config/maxmemory` に write される値は `1gb` 以外のものとなるため、Sentinel ポリシー違反となることが予測されます。\
実際にデータを write（put による更新）をしてみましょう。

```shell
% consul kv put redis/config/maxmemory 256mb
Error! Failed writing data: Unexpected response code: 403 (Permission denied: token with AccessorID '9267d017-94dd-5b72-9c91-083c651c48aa' lacks permission 'key:write' on "redis/config/maxmemory" in partition "default" in namespace "default")
```

上記の通り、Permission Error となることが確認できるかと思います。\
これは、クライアントアクセス用の Token `2ac79c47-fb43-e7f5-2371-c7950c79d594` に対して、Senitnel コードを含む ACL Policy `ensure-redis-maxmemory` が適用され、\
Sentinel のポリシー評価が行われた結果として、`"256mb" == "1gb"` により false, 即ち FAIL になったということを意味しています。

このように、Sentinel は Consul との連携において、ACL Policy を拡張する役割を持ち、ACL Policy が制御するクライアントの Token 単位で決め細やかにガードレールを実装することが可能です。\
Consul では ACL Policy を利用することで、Consul の各種機能への詳細なアクセス制御を実現していますが、\
Consul KVS は Terraform や Vault, Nomad などとも併用されるケースも多く、格納される Key-Value データの取り扱いはセキュアに行う必要があります。

Sentinel と併用することにより Consul の ACL Policy を拡張し、Consul KVS へのより高度なガバナンスを実現できることが理解いただけたかと思います。


## 参考リンク
- [Enforce ACL Policies with Sentinel](https://developer.hashicorp.com/consul/docs/secure/acl/sentinel)
- [Store and access key/value store](https://developer.hashicorp.com/consul/docs/automate/kv/store)
