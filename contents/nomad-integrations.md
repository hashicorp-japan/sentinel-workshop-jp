# Nomad 連携
Nomad Enterprise では ACL Policy の機能に対して Sentinel を用いることが可能です。\
Nomad においてワークロードやその関連リソースをスケジューリングする際に、Sentinel によるポリシー評価を連動させることにより、Nomad によりスケジュールされる Job やストレージにガバナンスを効かせることが可能になります。

また、Nomad において Sentinel の Enforcement Level は完全にサポートされています。\
Job のスケジューリング時に Sentinel 同様に `advisory`, `soft-mandatory`, `hard-mandatory` を指定することでポリシーコードを Nomad サーバに適用することにより、スケジューリング時にポリシー評価が行われます。

ここでは Nomad Enterprise 機能で利用可能な Namespace 機能において、default namespace への適用を禁止するようなポリシーコードを Nomad サーバに適用して、実際に挙動を確認してみましょう。

まずは、Nomad サーバを起動します。 \
Nomad で Sentinel 連携を行うためには、[ACL Policy](https://developer.hashicorp.com/nomad/docs/secure/acl) の機能を利用可能にしておく必要があります。\
そのため、Nomad サーバ起動時には、`-acl-enabled` のフラグを渡す必要がある点に気をつけてください。 \
（商用環境においては、`nomad.hcl` などの設定ファイルでフラグを有効化することが推奨とされます）

```shell
% export NOMAD_LICENSE_PATH="$HOME/nomad.hclic"
% nomad agent -dev -acl-enabled &
# ...
```

ACL Policy を有効化した Nomad サーバでは、`NOMAD_TOKEN`（HTTP Header の場合 `X-Nomad-Token`）が指定されない限り、原則として API を利用することはできないため、\
初期セットアップとして Bootstrap Token を発行します。

```shell
% nomad acl bootstrap
Accessor ID  = c25cff01-4ae1-5274-1ad6-eeca93b3af15
Secret ID    = dfd0134b-1ced-b4d6-4a81-50f49ee7f602
Name         = Bootstrap Token
Type         = management
Global       = true
Create Time  = 2025-12-08 04:59:10.156229 +0000 UTC
Expiry Time  = <none>
Create Index = 14
Modify Index = 14
Policies     = n/a
Roles        = n/a

% export NOMAD_TOKEN='dfd0134b-1ced-b4d6-4a81-50f49ee7f602'
```

以降のオペレーションでは `NOMAD_TOKEN` として Bootstrap Token がセットされているものとして進めます。
Nomad の Sentinel 連携には Enterprise ライセンスが必要となるため、機能が有効になっていることを確認します。

```shell
# Licensed Features に Sentinel Policies が含まれていることを確認
% nomad license get
Product        = nomad
License Status = valid
# ...
Datacenter     = *
Modules:
	multicluster-and-efficiency
	governance-policy
Licensed Features:
	Automated Upgrades
	Enhanced Read Scalability
	Redundancy Zones
	Namespaces
	Resource Quotas
	Audit Logging
	Sentinel Policies
	Multiregion Deployments
	Automated Backups
	Multi-Vault Namespaces
	Dynamic Application Sizing
	Node Pools Governance
	Multiple Vault Clusters
	Multiple Consul Clusters
```

続いて、Nomad Enterprise では Namespace の指定を行わない場合には、`default` namespace にスケジュールされるため、`generic` という別の namespace を作成します。

```shell
% nomad namespace apply -description="Namespace for validated workload" generic
Successfully applied namespace "generic"!
```

次に、Sentinel のポリシーコードを作成し、Nomad サーバに適用します。 \
ポリシーコードでは、[Nomad の `Job` オブジェクトのスキーマ](https://developer.hashicorp.com/nomad/docs/reference/sentinel-policy?page=enterprise&page=sentinel#sentinel-job-objects)で Sentinel から容易に必要な情報にアクセスすることが可能です。

また、Nomad に対して Sentinel ポリシーを適用する際には、ポリシーの `Scope` を定義する点に注意が必要です。\
Nomad では Job のスケジューリングに関連して、ワークロードが必要とする動的なボリュームなどを払い出すことが可能です。\
Scope は Sentinel ポリシーが評価されるタイミングについて定義したものとなり、以下の３つから選択する必要があります。
- `submit-job`
  - Job が Submit されたタイミングでのポリシー評価が行われる
- `submit-host-volume`
  - [Host Volume](https://developer.hashicorp.com/nomad/docs/architecture/storage/host-volumes) がプロビジョニングされたタイミングでのポリシー評価が行われる
- `submit-csi-volume`
  - [CSI Plugin](https://developer.hashicorp.com/nomad/docs/architecture/storage/csi) により Dynamic Host Volume がプロビジョニングされたタイミングでのポリシー評価が行われる

```shell
% cat >> restrict-default-namespace.sentinel <<EOF
# This policy would restrict schedule into Nomad default namespace

main = rule {
    job.namespace != "default"
}
EOF

# submit-job scope で hard-mandatory で Sentinel Policy を適用する
% nomad sentinel apply -level=hard-mandatory \
-scope=submit-job \
-description="Restrict scheduling to default namespace" \
restrict-default-namespace \
restrict-default-namespace.sentinel

Successfully wrote "restrict-default-namespace" Sentinel policy!

# Sentinel policy が追加されていることを確認
% nomad sentinel list
Name                        Scope       Enforcement Level  Description
restrict-default-namespace  submit-job  hard-mandatory     Restrict scheduling to default namespace
```

それでは、実際に Job のスケジューリングを行ってみましょう。

まずは、ポリシーを準拠するような Job Spec、つまり、`default` namespace ではない、作成した `generic` namespace にデプロイされる Job Spec を作成し、スケジュールします。 \
ポリシーに準拠しているため、`nomad job run` コマンド実行時には特段エラーや警告なども生じず、通常通りスケジューリングが行われることが確認できます。

```shell
% cat >> generic.nomad.hcl <<EOF
job "generic" {
  namespace = "generic"
  group "web" {
    network {
      port "web" {
        to = 8080
      }
    }
    task "nginx" {
      driver = "docker"
      config {
        image          = "bitnamilegacy/nginx:latest"
        ports          = ["web"]
        auth_soft_fail = true
      }
    }
  }
}
EOF

% nomad job run generic.nomad.hcl
```

次に、`default` namespace にスケジュールされるような Job Spec を作成して、スケジューリングを行います。 \
ここでは簡便のため、`nomad job init` コマンドを利用して、Job Spec を自動生成します。

```shell
% nomad job init -short
Example job file written to example.nomad.hcl

% nomad job run example.nomad.hcl
    2025-12-08T14:18:23.182+0900 [ERROR] http: request failed: method=PUT path=/v1/jobs
  error=
  | 1 error occurred:
  | \t* restrict-default-namespace : Result: false
  |
  | restrict-default-namespace:1:1 - Rule "main"
  |   Value:
  |     false
  |
  |
   code=500
    2025-12-08T14:18:23.182+0900 [DEBUG] http: request complete: method=PUT path=/v1/jobs duration="359.209µs"
Error submitting job: Unexpected response code: 500 (1 error occurred:
	* restrict-default-namespace : Result: false

restrict-default-namespace:1:1 - Rule "main"
  Value:
    false)
```

今度は、`nomad job run` 実行時に 500 エラーが返却されたことが確認できるかと思います。\
これは `nomad job init -short` コマンドで自動生成した Job Spec では namespace の指定がないために、デフォルトの挙動として `default` namespace にスケジューリングが行われようとしたためになります。\
このことからも、Nomad に対して適用した `restrict-default-namespace.sentinel` のポリシー評価が Job スケジュールのタイミングで実施され、ブロックされたことを確認することができます。

実際に、Job Spec の中で指定している Docker driver により作成された Docker コンテナを確認すると、以下のように、ポリシーを準拠している `generic.nomad.hcl` で記載の `bitnamilegacy/nginx:latest` イメージのコンテナのみがスケジュールされていることが確認できるかと思います。

```shell
% nomad job status -namespace=generic
ID       Type     Priority  Status   Submit Date
generic  service  50        running  2025-12-08T14:13:35+09:00

==> View and manage Nomad jobs in the Web UI: http://127.0.0.1:4646/ui/jobs

% docker container ls
CONTAINER ID   IMAGE                        COMMAND                  CREATED          STATUS          PORTS                                                  NAMES
393085e50ecb   bitnamilegacy/nginx:latest   "/opt/bitnami/script…"   10 minutes ago   Up 10 minutes   127.0.0.1:29212->8080/tcp, 127.0.0.1:29212->8080/udp   nginx-2834ce26-2f19-2d31-3358-e2e79c670c22
```

## 参考リンク
- [Create and manage Sentinel policies](https://developer.hashicorp.com/nomad/docs/govern/sentinel)
- [ACL System Overview](https://developer.hashicorp.com/nomad/docs/secure/acl)
- [Bootstrap the ACL system](https://developer.hashicorp.com/nomad/docs/secure/acl/bootstrap)
- [Sentiel Policy Reference](https://developer.hashicorp.com/nomad/docs/reference/sentinel-policy?page=enterprise&page=sentinel)
