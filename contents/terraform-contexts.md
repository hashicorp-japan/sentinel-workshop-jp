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

`terraform plan` コマンドを実行し、plan ファイルとして出力しておき、対象を確認します
- plan ファイルは gzip 圧縮ファイルのため、`unzip` コマンドなどで中身を確認することも可能です

```shell
% terraform plan -out terraform.tfplan
% ls -al terraform.tfplan

# plan ファイルから plan 時に読み込まれる Terraform の情報を抜き出し、JSON ファイルとして保存します
% terraform show -json terraform.tfplan |jq > terraform.tfplan.json
```

また、`terraform apply` コマンドを実行し、state ファイルが作成されたことを確認しましょう。

```shell
% terraform apply -auto-approve
# ...
% ls -al terraform.tfstate
```

以降は、これらの plan ファイルや state ファイルを Sentinel ポリシーの中で扱ってみましょう。\
まず、plan ファイルと state ファイルをポリシーコードから扱うことができるように、import の設定を追加します。\
この時、Terraform との連携を行うために、`sentinel` stanza の設定を有効化します。 \
（HCP Terraform 連携時には自動的にこのフラグは有効化されます）

```hcl
sentinel {
  features = {
    terraform = true
  }
}

import "plugin" "tfplan/v2" {
  config = {
    plan_path = "./terraform.tfplan.json"
  }
}

import "plugin" "tfconfig/v2" {
  config = {
    path = "./terraform.tfplan.json"
  }
}

import "plugin" "tfstate/v2" {
  config = {
    path = "./terraform.tfstate"
  }
}
```
