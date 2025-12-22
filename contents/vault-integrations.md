# Vault 連携
Vault Enterprise では、Vault のポリシー制御の機構として ACL Policy の他に、Sentinel ポリシーを利用することができます。 \
HCL で記述される ACL Policy では `path` および `capabilities` を軸としてシンプルかつ強力な RBAC を Vault で実現することができますが、Sentinel と Vault とを連携させることによりより高度なポリシーロジックを実装することができます。

Vault では Sentinel は完全にサポートされており、Sentinel から Vault の情報にアクセス可能で、`hard-mandatory`, `soft-mandatory`, `advisory` すべての Enforcement Level を利用可能です。\
Enforcement Level を指定し Sentinel ポリシーを Vault に設定することにより、Vault のユーザからのリクエストのタイミングで内容に応じたポリシーの評価を実現することができるようになります。

## Vault における Sentinel Policy
Vault で Sentinel ポリシーを利用する際には、ポリシーコードが評価する対象やタイミングにより、以下の２つのタイプに分類されます。

**RGPs: Role Governing Policies**
- Vault が発行する Token や Entity, Group に紐づく Sentinel ポリシーを RGPs と呼びます
- RGPs がサポートする上記のオブジェクトを作成する際に、Vault に RGPs として適用済みの Sentinel ポリシーを指定することで、Vault のオブジェクトに対するポリシー評価が行われるようになります
  - Vault が標準でもつ ACL Policy は、Token に紐づき同じような形でポリシーを利用することができるため、RGPs は ACL Policy よりもより高度な RBAC を実現したい場合に利用することができます

**EGPs: Endpoint Governing Policies**
- Vault は管理コンポーネントも含めすべてパスベースでの操作・アクセスが可能ですが、Vault におけるパスに紐づく Sentinel ポリシーを EGPs と呼びます
- 実装した Sentinel ポリシーを適用する際に、Vault 上のパスを指定するものが EGPs とも考えることができます
- RGPs とは異なり、Vault の特定パスへのリクエスト時のペイロードや属性に応じてポリシー評価が行われる形になります

上記のようにポリシーが想定している評価対象が異なるだけでなく、RGPs と EGPs とは評価されるタイミングについても異なります。（[参考](https://developer.hashicorp.com/vault/docs/enterprise/sentinel#policy-evaluation)）\
EGPs/RGPs ともに、Sentinel ポリシーは Vault が標準的にもつ ACL Policy を拡張する形で連携するため、ACL Policy と Sentinel Policy との択一ではなく併用することが可能です。\
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
# ...
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
続いて RGPs を実装してみましょう。 \
RGPs ではパスに対するポリシー指定を行う EGPs とは異なり、ACL Policy のように Vault への認証が完了した Vault クライアントの属性などに応じたポリシーの実装が可能です。

早速、RGPs として利用する Sentinel ポリシーを実装してみましょう。\
ここでは、`token` ブロパティの `metadata` の値に応じた Senitnel ポリシーを実装し、`application_type` という特定の metadata を持つ Token に対してのみ、作成時の TTL をチェックするような RGPs を実装してみます。\
(Token 発行時の `token_duration` が 86400s = 24h 以下の場合には FAIL となる)

```shell
% cat >> enforce-token-ttl-batch-application-type.sentinel <<EOF
# this policy enforce TTL shorter than 24h with token having application_type fields
precond = token.meta.application_type is defined

main = rule when precond {
    token.creation_ttl_seconds < 86400
}
EOF
```

続いて、作成した RPGs を Vault に登録します。\
EGPs を登録する際同様に、base64 する点については共通ですが、`paths` の指定がない点がポイントとなります。

```shell
# 実装したポリシーを hard-mandatory レベルで RGPs として登録する
% vault write sys/policies/rgp/enforce-token-ttl-application-type \
policy=$(base64 -i enforce-token-ttl-application-type.sentinel) \
enforcement_level="hard-mandatory"
```

次に、RGPs の評価対象となる Token のうち、Root Token（或いは `root` ポリシーが適用されている Token）以外の２つの Token を作成しておきます。\
作成する２つの Token に対して、ポリシーはいずれも `default` ACL Policy および、登録した RGPs Sentinel ポリシー `enforce-token-ttl-application-type` を指定しますが、片方には `application_type` をキーに持つ metadata を付与しておきます。 \

```shell
# metadata なしの Token
% vault token create -policy='default' -policy='enforce-token-ttl-application-type'
Key                  Value
---                  -----
token                hvs.CAESIHwUB2dw7MuLIEohLdilLtUUhE1pQ7mbXaD4RmyvlZC1Gh4KHGh2cy5lMEtnakgxdUJqdTJ3N2I2NXF1emgyZ2g
token_accessor       L1ms1cOTWnJyKFSASvHVKSuL
token_duration       768h
token_renewable      true
token_policies       ["default" "enforce-token-ttl-application-type"]
identity_policies    []
policies             ["default" "enforce-token-ttl-application-type"]

# application_type=batch の metadata を持つ Token
% vault token create -policy='default' -policy='enforce-token-ttl-application-type' -metadata='application_type=batch'
Key                            Value
---                            -----
token                          hvs.CAESICJQP3gdWZISnXGS1PZZNlxfz_eYNrBNbo68cLHzhzfDGh4KHGh2cy5jSkwwTHROS2NWT1lqNU1pU0JleVBIY2k
token_accessor                 WnVaxMSQ9ZLu7AtkQqw51o0N
token_duration                 768h
token_renewable                true
token_policies                 ["default" "enforce-token-ttl-application-type"]
identity_policies              []
policies                       ["default" "enforce-token-ttl-application-type"]
token_meta_application_type    batch
```

Token に `-policy` オプションで RPGs の名前を指定していることからも、ACL Policy と同じような形で Token に付与することができることがわかるかと思います。

それでは、実際に適用した `enforce-token-ttl-application-type` RGPs が評価されるか確認してみましょう。\
発行した２つの Token はそれぞれ token_accessor `L1ms1cOTWnJyKFSASvHVKSuL` および `WnVaxMSQ9ZLu7AtkQqw51o0N` で一意に特定することが可能であり、片方は `application_type=batch` という Token metadata を持っています。\
また、いずれも RPGs の他に、`default` ACL Policy も付与されているため、ACL Policy の世界においては `vault token lookup` コマンドが実行可能であることが期待されます。

いずれも Token 作成時の有効期限はデフォルトの 768h（32日間）であり、token_accessor `WnVaxMSQ9ZLu7AtkQqw51o0N` を持つ Token については `enforce-token-ttl-application-type` で実装した Sentinel ポリシーに準拠していない状態になっています。\
このため、token_accessor `WnVaxMSQ9ZLu7AtkQqw51o0N` を持つ Token で `vault token lookup`（`read auth/token/lookup`）を実行した際には、
- ACL Policy `default` には準拠する
- Sentinel Policy（RGPs）`enforce-token-ttl-application-type` には準拠しない

形となり、リクエストはエラーとなることが期待されます。\
Token の値を切り替えながら実際に試してみましょう。

```shell
# metadata を持たない Token（token_accessor L1ms1cOTWnJyKFSASvHVKSuL）
% VAULT_TOKEN='hvs.CAESIHwUB2dw7MuLIEohLdilLtUUhE1pQ7mbXaD4RmyvlZC1Gh4KHGh2cy5lMEtnakgxdUJqdTJ3N2I2NXF1emgyZ2g' vault token lookup
Key                 Value
---                 -----
accessor            L1ms1cOTWnJyKFSASvHVKSuL
creation_time       1766414580
creation_ttl        768h
display_name        token
entity_id           n/a
expire_time         2026-01-23T23:43:00.567025+09:00
explicit_max_ttl    0s
id                  hvs.CAESIHwUB2dw7MuLIEohLdilLtUUhE1pQ7mbXaD4RmyvlZC1Gh4KHGh2cy5lMEtnakgxdUJqdTJ3N2I2NXF1emgyZ2g
issue_time          2025-12-22T23:43:00.567027+09:00
meta                <nil>
num_uses            0
orphan              false
path                auth/token/create
policies            [default enforce-token-ttl-application-type]
renewable           true
ttl                 767h52m7s
type                service

# application_type の metadata を持つ Token（token_accessor WnVaxMSQ9ZLu7AtkQqw51o0N）
% VAULT_TOKEN='hvs.CAESICJQP3gdWZISnXGS1PZZNlxfz_eYNrBNbo68cLHzhzfDGh4KHGh2cy5jSkwwTHROS2NWT1lqNU1pU0JleVBIY2k' vault token lookup
Error looking up token: Error making API request.

URL: GET http://localhost:8200/v1/auth/token/lookup-self
Code: 403. Errors:

* 2 errors occurred:
	* rgp standard policy "root/enforce-token-ttl-application-type" evaluation resulted in denial.

The specific error was:
<nil>

A trace of the execution for policy "root/enforce-token-ttl-application-type" is available:

Result: false

Description: this policy enforce TTL shorter than 24h with token having application_type fields

Rule "main" (root/enforce-token-ttl-application-type:5:1) = false
	* permission denied
```

上記のように、いずれも Token の有効期限（token_duration）は 768h と RGPs には準拠していないものの、RGPs の条件に合致する場合には ACL Policy に加えて Token 有効期限に関する評価が行われることが確認できたかと思います。\
このように、RGPs では Vault への認証が完了したのちに、ACL Policy ではアクセスすることができないような詳細な Vault のプロパティをポリシーの評価対象とすることができ、ACL Policy を拡張するような形でより高度な Vault の RBAC を実現することが可能です。

最後に、RPGs が適用されているこの状態から、実装した RGPs を満たすような形を試してみます。\
Sentinel で記述した RGPs は Token 作成時の有効期限が 24h 以下である場合に PASS するため、`application_type` キーの metadata を持つ Token 作成時には `-ttl` で 24h 以下の値を指定すればよいことがわかります。

```shell
% vault token create -policy=default -policy=enforce-token-ttl-application-type -metadata='application_type=batch' -ttl=12h -format=json | jq .auth.client_token
"hvs.CAESIH7tmrmKT-MbkirHxPjnBWK-lFGnvUe2CFzfNOT5MkqsGh4KHGh2cy4wTkhnUjFkaERGcjR3RHIyZnRSQ0NsUVA"

% VAULT_TOKEN="hvs.CAESIH7tmrmKT-MbkirHxPjnBWK-lFGnvUe2CFzfNOT5MkqsGh4KHGh2cy4wTkhnUjFkaERGcjR3RHIyZnRSQ0NsUVA" vault token lookup
Key                 Value
---                 -----
accessor            4JVIrOfTlcR7wRYWM7NUIktS
creation_time       1766415901
creation_ttl        12h
display_name        token
entity_id           n/a
expire_time         2025-12-23T12:05:01.553452+09:00
explicit_max_ttl    0s
id                  hvs.CAESIH7tmrmKT-MbkirHxPjnBWK-lFGnvUe2CFzfNOT5MkqsGh4KHGh2cy4wTkhnUjFkaERGcjR3RHIyZnRSQ0NsUVA
issue_time          2025-12-23T00:05:01.553452+09:00
meta                map[application_type:batch]
num_uses            0
orphan              false
path                auth/token/create
policies            [default enforce-token-ttl-application-type]
renewable           true
ttl                 11h59m46s
type                service
```

上記のように、`-ttl=12h` を指定して作成された Token の場合には、`-policy=enforce-token-ttl-application-type` RGPs が付与されていてもポリシー評価条件に準拠していれば、レスポンスが得られることが確認できたかと思います。

今回は、RGPs を紐づけることが可能な Vault の identity（token, entity, group）のうち、最も簡単な token に紐づく RGPs を実装しましたが、RGPs は internal/external group のプロパティや、より細かい entity のプロパティなどに紐づけることで、Vault クライアントの属性に応じたポリシー制御を行うことが可能となります。


## EGPs と RGPs、ACL Policy との使い分けについて
ここまで、EGPs および RGPs をそれぞれ実装し動作確認をすることで、Vault において Sentinel を利用する際のポリシーの種類について学習してきました。\
前述の通り、Vault が従来から持つ ACL Policy だけであっても、シンプルかつ強力なポリシー制御を行うことができますが、Sentinel と連携することでより高度なポリシーを実装可能であることが理解できたかと思います。

Vault における Sentinel ポリシーは、EGPs および RGPs に種類としては分かれますが、Sentinel の観点で考えた際にはいずれも Vault のプロパティにアクセスしポリシー評価を行なっている、という点がポイントとなります。\
これはすなわち、（言語としての）Sentinel で記述されたポリシーのロジックを、Vault において「パスベースでの制御として適用するのか（EGPs）」「ロールベースでの制御として適用するのか（RGPs）」の違いにすぎず、\
「Sentinel のある文法を利用したから EGPs（或いは RPGs）になる」ということを意味していません。

ACL Policy, EGPs, RGPs はすべて併用することが可能であるため、Vault においてポリシーを適用したい対象やタイミングに応じて使い分けていくことが望ましいと言えます。


## 参考リンク
- [Manage Vault policies with Sentinel](https://developer.hashicorp.com/vault/docs/enterprise/sentinel)
- [Sentinel properties for Vault](https://developer.hashicorp.com/vault/docs/enterprise/sentinel/properties)
- [Sentinel and Vault](https://developer.hashicorp.com/sentinel/docs/vault)
- [Create/Update EGP policy](https://developer.hashicorp.com/vault/api-docs/system/policies#create-update-egp-policy)
- [Create/Update RGP policy](https://developer.hashicorp.com/vault/api-docs/system/policies#create-update-acl-policy)
