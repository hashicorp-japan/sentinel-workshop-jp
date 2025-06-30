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
- `imports`
  - ポリシーコードの再利用性を高めるための仕組みやライブラリの設定を行う
- `params`
  - 変数定義やその値を指定する

## Policy Libraries & Pre-written policies


## 参考リンク
