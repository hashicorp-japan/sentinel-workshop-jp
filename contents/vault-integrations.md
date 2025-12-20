# Vault 連携
Vault Enterprise では、Vault のポリシー制御の機構として ACL Policy の他に、Sentinel ポリシーを利用することができます。 \
HCL で記述される ACL Policy では `path` および `capabilities` を軸としてシンプルかつ強力な RBAC を Vault で実現することができますが、Sentinel と Vault とを連携させることによりより高度なポリシーロジックを実装することができます。

Vault では Sentinel は完全にサポートされており、Sentinel から Vault の情報にアクセス可能で、`hard-mandatory`, `soft-mandatory`, `advisory` すべての Enforcement Level を利用可能です。\
Enforcement Level を指定し Sentinel ポリシーを Vault に設定することにより、Vault のユーザからのリクエストのタイミングで内容に応じたポリシーの評価を実現することができるようになります。

## Vault における Sentinel Policy
Vault で Sentinel ポリシーを利用する際には、ポリシーコードが評価する対象やタイミングにより、以下の２つのタイプに分類されます。

**RGPs: Role Governing Policies**
- Vault が発行する Token や Entity, Group に紐づく Sentinel ポリシーを RGPs と呼びます
- RGPs でサポートされる Vault 上のオブジェクトに対するポリシー制御を Sentinel ポリシーコードの中で記述するため、ポリシーを登録する際にはパスの指定は不要となります

**EGPs: Endpoint Governing Policies**
- Vault は管理コンポーネントも含めすべてパスベースでの操作・アクセスが可能ですが、Vault におけるパスに紐づく Sentinel ポリシーを EGPs と呼びます
- 実装した Sentinel ポリシーを適用する際には、Vault 上のパスを指定するものが EGPs とも考えることができます

上記のようにポリシーの評価対象が異なるだけでなく、RGPs と EGPs とは評価されるタイミングについても異なります。\
Sentinel ポリシーは Vault が標準的にもつ ACL Policy を拡張する形で連携するため、ACL Policy と Sentinel Policy との択一ではなく併用することが可能です。\
(ACL Policy は Vault が発行する Token に紐づいたアクセス制御を行います)

また、Vault では最も強力な権限である Root Token を利用することもできますが、EGPs, RGPs ともに Root Token に対しては評価が行われない点についても留意するようにしてください。\
（Root Token は原則的に初回の構築時に利用後に失効させ、必要となったタイミングで適切なプロセスを経た上で再発行することが推奨です）

## EGPs
まずは Vault への認証有無を問わず利用することができる EGPs から実際に試してみましょう。 \
Vault サーバを起動後、Sentinel Policy と連携する Enterprise ライセンスが適用されているかを確認します。

```shell
% export VAULT_LICENSE_PATH="$HOME/vault.hclic"
% export VAULT_ADDR='http://localhost:8200'
% vault server -dev &
# ...

# 出力の features に Sentinel が含まれていることを確認する
% vault license get

Key                          Value
---                          -----
expiration_time              2025-12-31T00:00:00Z
features                     [HSM Performance Replication DR Replication MFA Sentinel Seal Wrapping Control Groups Performance Standby Namespaces KMIP Entropy Augmentation Transform Secrets Engine Lease Count Quotas Key Management Secrets Engine Automated Snapshots Key Management Transparent Data Encryption Secrets Sync Secrets Import Oracle Database Secrets Engine]
license_id                   6a7f22a5-211c-61f6-53ce-5681b0b66513
performance_standby_count    9999
start_time                   2025-09-25T00:00:00Z
termination_time             2025-12-31T00:00:00Z
```

それでは早速 Sentinel ポリシーを作成してみましょう。\
ここでは、[time](https://developer.hashicorp.com/sentinel/docs/imports/time) import を利用して、ある時刻以前に発行された Token 以外はすべて利用不可とするような Sentinel ポリシーを記述してみましょう。\
（これは Break-glass と呼ばれる Vault への侵害が疑われた際の対処としても利用することができます）

なお、今回は `token` プロパティを参照していますが、 Sentinel コードからは様々な Vault のプロパティにアクセスすることが可能です。

```shell
% cat >> disable-staled-token.sentinel <<EOF
import "time"

main = rule {
    # 2025/12/20 19:00:00 以前に発行された Token は false となりポリシー違反となる
    time.load("2025-12-20T19:00:00+09:00").unix < time.load(token.creation_time).unix
}
EOF
```

次に、この記述した EPGs をすべての Vault のパスに対して `hard-mandatory` のレベルで適用する設定を行います。 \
記述した EGPs を Vault に適用する際には、ポリシーコード（`.sentinel` ファイル）を base64 で難読化する必要がある点に注意してください。

```shell
% vault write sys/policies/egp/disable-staled-token \
policy=$(base64 -i disable-staled-token.sentinel) \
paths="*" \
enforcement_level="hard-mandatory"
Success! Data written to: sys/policies/egp/disable-staled-token
```

早速 EGPs が有効になっているかを確認していきましょう。
前述の通り、Root Token は Vault においてポリシー評価の対象とならないため、`root` ボリシーではなく `default` ポリシーを持つ Token を発行して動作確認を行います。

```shell
# default ポリシーを持つ Token の発行
% vault token create -policy=default
Key                  Value
---                  -----
token                hvs.CAESIBDLhrX6I9fE1XTL3-wSMllm2Ug0TSC7bNX2Wq8NZUumGh4KHGh2cy5nT1dlRmV5NDVnN1VEeGlqdWhMQUMwYUw
token_accessor       LzLx2KzJqAj0scwpvYlVX7qR
token_duration       768h
token_renewable      true
token_policies       ["default"]
identity_policies    []
policies             ["default"]

# Token が発行された時刻を確認する
% vault token lookup -accessor LzLx2KzJqAj0scwpvYlVX7qR
Key                 Value
---                 -----
accessor            LzLx2KzJqAj0scwpvYlVX7qR
creation_time       1766222289
creation_ttl        768h
display_name        token
entity_id           n/a
expire_time         2026-01-21T18:18:09.360352+09:00
explicit_max_ttl    0s
id                  n/a
issue_time          2025-12-20T18:18:09.360353+09:00
meta                <nil>
num_uses            0
orphan              false
path                auth/token/create
policies            [default]
renewable           true
ttl                 767h23m18s
type                service
```

`default` ポリシーで許可されている内容を確認し、ACL Policy の他に EGPs が適用されることを確認します。

```shell
$ vault policy read default
# Allow tokens to look up their own properties
path "auth/token/lookup-self" {
    capabilities = ["read"]
}
# ...
```

出力から分かるように、Vault の `default` ACL Policy では `auth/token/lookup-self` に対する read 操作は **ACL Policy の範疇においては** 許可されています。\
`vault token lookup` コマンドは内部的にはこの `auth/token/lookup-self` のパスを呼ぶため、Accessor `LzLx2KzJqAj0scwpvYlVX7qR` を持つ Token は `vault token lookup` コマンドの実行が許可されています。

一方、先ほど適用した EGPs では、すべての Vault パスに対して、Token の発行時刻が 2025/12/20 19:00:00 以前のものはすべて禁止する形になっています。\
先ほどの `vault token lookup` コマンドの出力から、Token は発行時刻が `1766222289`（2025-12-20 18:18:09+09:00）となっているため、ポリシーに違反することが期待されます。

`default` ポリシーが当たっている Token では Sentinel ポリシーが適用されていることを確認します。

```shell
% VAULT_TOKEN='hvs.CAESIBDLhrX6I9fE1XTL3-wSMllm2Ug0TSC7bNX2Wq8NZUumGh4KHGh2cy5nT1dlRmV5NDVnN1VEeGlqdWhMQUMwYUw' vault token lookup
Error looking up token: Error making API request.

URL: GET http://localhost:8200/v1/auth/token/lookup-self
Code: 403. Errors:

* 2 errors occurred:
	* egp standard policy "root/disable-staled-token" evaluation resulted in denial.

The specific error was:
<nil>

A trace of the execution for policy "root/disable-staled-token" is available:

Result: false

Description: <none>

Rule "main" (root/disable-staled-token:4:1) = false
	* permission denied
```

これは、本来 `default` ACL Policy が適用されている、`LzLx2KzJqAj0scwpvYlVX7qR` Accessor を持つ Token は `auth/token/lookup-self` への read 処理が許可されているにもかかわらず、\
その後の EPGs の評価（Token 発行時刻による Sentinel ポリシー評価）によりポリシー違反となり、アクセスが拒否されたことを意味しています。

今回は簡便のために、すべての Vault パスに対しての EGPs 適用を行いましたが、実際の運用の中では Vault のパスベースできめ細やかにポリシー制御を行うことが可能です。

## RGPs
続いて RGPs を実装して行ってみましょう。


## 参考リンク
- [Manage Vault policies with Sentinel](https://developer.hashicorp.com/vault/docs/enterprise/sentinel)
- [Sentinel properties for Vault](https://developer.hashicorp.com/vault/docs/enterprise/sentinel/properties)
- [Sentinel and Vault](https://developer.hashicorp.com/sentinel/docs/vault)
