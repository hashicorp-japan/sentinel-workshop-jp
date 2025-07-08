# Sentinel 設定ファイル

Sentinel では、ポリシーコードの挙動を定義するために、設定ファイルを利用することが可能で、慣例的には `sentinel.hcl` と命名されます。 \
Sentinel の設定ファイルでは、`HCL（HashiCorp Configuration Language）` または `JSON` での記述がサポートされています。

`sentinel.hcl` は `sentinel apply` または `sentinel test` コマンド実行時に自動的に読み込まれますが、 \
`-config-path` フラグを利用することで読み込み先のパスを変更することが可能です。　\
そのため、以下のように設定ファイルを環境ごとに分離することも可能です。

```tree
.
├── development
│   └── sentinel.hcl
├── production
│   └── sentinel.hcl
└── README.md
```

上記のディレクトリ構成の場合、例えば development 環境用の Sentinel のみを評価したい場合には、以下のように指定を行うことが可能です。

```shell
$ sentinel apply -config-path development/sentinel.hcl
```

`sentinel.hcl` の役割は、**どのポリシーコードをどのように実行するか** を定義するもので、主に以下のブロックから構成されます。
- `policy`
  - 評価対象となるポリシーコードを指定し、その挙動をポリシーごとに定義する
  - `source` でポリシーコードを参照し、`enforcement_level` により強制レベルを指定します
- `import`
  - ポリシーコードの再利用性を高めるための仕組みやライブラリの設定を行う
- `params`
  - 変数定義やその値を指定する

また、`sentinel.hcl` は `policy` などのブロックを複数回記述することで、ポリシー設定を束ねることが可能です。

```hcl
policy "enforce-camel" {
  source            = "./enforce-camel-case.sentinel"
  enforcement_level = "hard-mandatory"
}

policy "require-description" {
  source            = "./require-description-in-variables.sentinel"
  enforcement_level = "soft-mandatory"
}
```

## Policy Libraries & Pre-written policies

sentinel.hcl の policy では、`source` により評価するポリシーコードを指定することが可能ですが、これは URL によるリモートのポリシーコードもサポートされます。 \
これにより、Sentinel では既に実装したポリシーコードの再利用性を高めることが可能です。

ここでは、このリポジトリをリモートとして、[`sentinel-workshop-jp/assets/sample-policies/remote-policy.sentinel`](../assets/sample-policies/remote-policy.sentinel) を取り込んでみましょう。

```shell

```

また、HashiCorp がメンテナンスを行う GitHub リポジトリでは、主要クラウド環境向けの事前定義済みポリシーが用意されています。 \
これらは、Terraform の Module や Provider などと同様に、[Terraform Registry](https://registry.terraform.io/) で [Policy Libraries](https://registry.terraform.io/browse/policies) として公開されています。

主要クラウド環境向けの定義済みポリシーコードも併せて GitHub リポジトリで公開されています。

<details><summary>AWS の場合はこちら</summary>

</details>

<details><summary>Azure の場合はこちら</summary>

Computes: <https://github.com/hashicorp/policy-library-azure-compute-terraform> \
Networks: <https://github.com/hashicorp/policy-library-azure-networking-terraform> \
Storages: <https://github.com/hashicorp/policy-library-azure-storage-terraform> \
Databases: <https://github.com/hashicorp/policy-library-azure-databases-terraform>

</details>

<details><summary>Google Cloud の場合はこちら</summary>

</details>


## 参考リンク
- [`-config-path` Option](https://developer.hashicorp.com/sentinel/docs/commands/apply#config-path)
- [Enforcement Level](https://developer.hashicorp.com/sentinel/docs/concepts/enforcement-levels)
- [Configuration file reference](https://developer.hashicorp.com/sentinel/docs/configuration#configuration-file-reference)
- [Policy Code Samples (GitHub)](https://github.com/hashicorp/terraform-sentinel-policies)
