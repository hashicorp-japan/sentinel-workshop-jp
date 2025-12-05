# Terraform Contexts
この章では、Sentinel と Terraform との連携を行う際に重要となる mock を扱うための概念を学習します。

Sentinel は HashiCorp プロダクトとの親和性を念頭に開発が行われているため、Terraform Community Edition や HCP Terraform との連携を容易に行うための[ライブラリ（imports）](https://developer.hashicorp.com/terraform/cloud-docs/policy-enforcement/import-reference#functions-for-terraform) が用意されています。

代表的なものは以下となります。
- `tfstate/v2`
  - Sentinel ポリシーから State ファイル内の情報にアクセスすることができるもの
  - <https://developer.hashicorp.com/terraform/cloud-docs/policy-enforcement/import-reference/tfstate-v2>
- `tfplan/v2`
  - Sentinel ポリシーから `terraform plan` 時の情報にアクセスすることができるもの
  - <https://developer.hashicorp.com/terraform/cloud-docs/policy-enforcement/import-reference/tfplan-v2>
- `tfconfig/v2`
  - Sentinel ポリシーから provider の設定など、`*.tf` ファイル内で定義された情報にアクセスすることができるもの
  - <https://developer.hashicorp.com/terraform/cloud-docs/policy-enforcement/import-reference/tfconfig-v2>
- `tfrun/v2`
  - HCP Terraform の Run 情報（plan/apply）にアクセスすることができるもの
  - HCP Terraform でのみ利用可能
  - <https://developer.hashicorp.com/terraform/cloud-docs/policy-enforcement/import-reference/tfrun>

これらは特定の条件を満たすことで利用できるライブラリ（plugin import）と考えることもできます。\
(HCP Terraform との連携の際には、これらの import はシームレスに行うため、ポリシー開発時に開発者が意識することはあまりありません)

そのため、ポリシー開発者は `terraform plan` や `terraform apply` 時に評価を行うようなポリシーを開発する際には、\
上記のライブラリの import を行うだけで plan/apply などの Terraform 固有の情報にアクセスすることができるようになります。

ここでは、実際に `terraform plan` や `terraform apply` により作成される plan ファイルや state ファイルを用いて、それらが Sentinel でどのように扱うことができるかを学習していきましょう。

まず、terraform-provider-random を用いて、ローカル環境に plan ファイルおよび state ファイルを生成します。

```shell
% cat >> main.tf <<EOF
terraform {
  required_providers {
    random = {
      source = "hashicorp/random"
    }
  }
}

resource "random_id" "this" {
  byte_length = 8
}

output "this" {
  value = random_id.this
}
EOF
```

`terraform plan` および `terraform apply` コマンドを実行し、state ファイルが作成されたことを確認しましょう。

```shell
% terraform init
% terraform plan
# ...
% terraform apply -auto-approve
# ...
% ls -al terraform.tfstate
```

先ほどの apply により生成された `random_id` を確認し、tfstate ファイルの中にも作成済みのリソースとして記録されていることを確認しましょう。
- tfstate ファイルは JSON ファイルであるため、`cat` コマンドなどで値を確認することも可能です

```shell
% terraform state show random_id.this
```

出力結果から、`random_id.this.id` の値を確認します。（`k10nJGyIaTQ` など）

再度 `terraform plan` を実行し、今度は tfplan ファイルとして出力しておきます。
- plan ファイルは gzip 圧縮ファイルのため、`unzip` コマンドなどで中身を確認することも可能です

```shell
% terraform plan -out terraform.tfplan
% ls -al terraform.tfplan

# plan ファイルから plan 時に読み込まれる Terraform の情報を抜き出し、JSON ファイルとして保存します
% terraform show -json terraform.tfplan |jq > terraform.tfplan.json
```

以降は、state に記録されたデータを Sentinel ポリシーの中で扱ってみましょう。\
まず、plan ファイルと state ファイルをポリシーコードから扱うことができるように、import の設定を追加します。\
この時、Terraform との連携を行うために、`sentinel` stanza の設定を有効化します。 \
（HCP Terraform 連携時には自動的にこのフラグは有効化されます）

```shell
% cat > sentinel.hcl <<EOF
sentinel {
  features = {
    terraform = true
  }
}

import "plugin" "tfstate/v2" {
  config = {
    path = "./terraform.tfplan.json"
  }
}

policy "ensure-random-id-in-state" {
    source            = "./ensure-random-id-in-state.sentinel"
    enforcement_level = "advisory"
}
EOF
```

続いて、State の情報を扱うようなポリシーコードを作成していきます。\
`terraform.tfstate` ファイルの中身を確認してみるとわかりますが、先ほど確認した `random_id.this.id` の値が `k10nJGyIaTQ` であることを確認するポリシーを書いてみましょう。\
`tfstate/v2` のリファレンスである [こちら](https://developer.hashicorp.com/sentinel/docs/features/terraform/tfstate-v2) を参考に state 内のデータへアクセスしてみます。

```shell
% cat >> ensure-random-id-in-state.sentinel <<EOF
import "tfstate/v2" as tfstate

randomId = tfstate.resources["random_id.this"].values.id

main = rule {
    randomId == "k10nJGyIaTQ"
}
EOF
```

このポリシー評価してみると、PASS の出力となることがわかるかと思います。

```shell
% sentinel apply -trace ensure-random-id-in-state.sentinel
```

これは、実装した Sentinel ポリシーの中で `tfstate/v2` import を利用することにより、state ファイルにアクセスが行われたことを意味しています。

`tfstate/v2`, `tfplan/v2`, `tfconfig/v2`, `tfrun` といった Terraform のコンテキストにアクセスすることができる imports は非常に強力で、ポリシーでの評価対象を拡張するだけでなく、\
ポリシーそのものをシンプルに保つことができるというメリットもあります。

なお、今回はローカル環境での実施であるため、tfplan ファイルを用意しましたが、HCP Terraform においてはこれらの Terraform コンテキストはシームレスに連携されるため、\
ポリシーの開発者側で tfplan ファイルの用意などは不要となります。
