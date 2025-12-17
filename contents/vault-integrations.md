# Vault 連携
Vault Enterprise では、Vault のポリシー制御の機構として ACL Policy の他に、Sentinel ポリシーを利用することができます。 \
HCL で記述される ACL Policy では `path` および `capabilities` を軸としてシンプルかつ強力な RBAC を Vault で実現することができますが、Sentinel と Vault とを連携させることによりより高度なポリシーロジックを実装することができます。

Vault では Sentinel は完全にサポートされており、Sentinel から Vault の情報にアクセス可能で、`hard-mandatory`, `soft-mandatory`, `advisory` すべての Enforcement Level を利用可能です。\
Enforcement Level を指定し Sentinel ポリシーを Vault に設定することにより、Vault のユーザからのリクエストのタイミングで内容に応じたポリシーの評価を実現することができるようになります。

## Vault における Sentinel Policy
Vault で Sentinel ポリシーを利用する際には、ポリシーコードが評価する対象やタイミングにより、以下の２つのタイプに分類されます。

**RGPs: Role Governing Policies**
- Vault が発行する Token や Entity, Group に紐づく Sentinel ポリシーを RGPs と呼びます

**EGPs: Endpoint Governing Policies**
- Vault は管理コンポーネントも含めすべてパスベースでの操作・アクセスが可能ですが、Vault におけるパスに紐づく Sentinel ポリシーを EGPs と呼びます

上記のようにポリシーの評価対象が異なるだけでなく、RGPs と EGPs とは評価されるタイミングについても異なります。\
Sentinel ポリシーは Vault が標準的にもつ ACL Policy を拡張する形で連携するため、ACL Policy と Sentinel Policy との択一ではなく併用することが可能です。\
(ACL Policy は Vault が発行する Token に紐づいたアクセス制御を行います)

ACL Policy は Vault においてなんらかの Auth Method により認証成功した後（＝Token を取得した後）の認可の役割を持つため、Vault に対する認証有無によりクライアントからのリクエストは、ACL Policy を含め以下の順番で評価される点に注意が必要です。

- Token なしのリクエストの場合は、EGPs のみが評価される
- Token あり（Vault への認証済み）のリクエストの場合は、以下の順で評価が行われます
  1. ACL Policy により特定パスへのアクセス権限チェックが行われる
  2. ACL Policy 評価が PASS の場合、RGPs が評価される
  3. RGPs 評価が PASS の場合、EGPs が評価される
  4. EGPs 評価が PASS の場合、Vault はレスポンスを返す

また、Vault では最も強力な権限である Root Token を利用することもできますが、EGPs, RGPs ともに Root Token に対しては評価が行われない点についても留意するようにしてください。\
（Root Token は原則的に初回の構築時に利用後に失効させ、必要となったタイミングで適切なプロセスを経た上で再発行することが推奨です）

## EGPs
まずは Vault への認証有無を問わず利用することができる EGPs から実際に試してみましょう。\
Vault は数多くの API を持ちますが、その一部は認証なしで利用することができるものも含まれます。\
ここでは、Vault のヘルスチェックに該当する `sys/health` API に対するリクエストを例に、EGPs を実装してみましょう。

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


## RGPs


## 参考リンク
- [Manage Vault policies with Sentinel](https://developer.hashicorp.com/vault/docs/enterprise/sentinel)
- [Sentinel properties for Vault](https://developer.hashicorp.com/vault/docs/enterprise/sentinel/properties)
- [Sentinel and Vault](https://developer.hashicorp.com/sentinel/docs/vault)
