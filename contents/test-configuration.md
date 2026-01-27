# Tests 1: 設定ファイル
Sentinel ではランタイムとして、実装したポリシーが正常に動作するかをテストするための機構を兼ね備えています。 \
これにより、ポリシーコードの開発者は `sentinel test` コマンドを用いて、ユニットテストを行うことが可能です。

`sentinel apply` コマンド時にはデフォルトで `sentinel.hcl` （または `-config` オプションで指定したパスの設定ファイル）を読み込み、\
`*.sentinel` ファイルに記載されたポリシーを評価する挙動をします。 \
一方、一般的なプログラミング言語同様に、Sentinel のテストコード実行時には、変数や依存関係が異なるケースがあるため、一般的には `sentinel apply` 時に利用する `sentinel.hcl` とは異なる、\
テスト用の設定ファイルやモックデータを用意することが多いです。

例えば、 `import "time"` を利用するライブラリは Sentinel 実行環境の現在時刻に関連する情報を返すため、水曜日であることを遵守させるポリシーの FAILURE ケースは水曜日以外には用意することができません。 \
このような問題を避けるためには、mock データが必要となります（`import "time"` で得られる出力結果をテストするために必要な、様々な入力を開発者側で用意する）

テスト用の設定ファイルは sentinel.hcl の記述と基本的に同じとなりますが、`test` や `mock` などテストに関連する固有のブロックが活用されることが多いです。

## Search Path
`sentinel test` コマンド実行時には `test/<policy_name>/*.hcl` が探索される挙動をします。 \
なお、`<policy_name>` は定義したポリシーファイルから `.sentinel` を除いた文字列と一致している必要があります。

また、多くの場合、テスト成功時/失敗時とでは期待される結果および mock する入力値は異なるため、両方の sentinel.hcl を用意します。

そのため、ディレクトリ構成としては以下のようになります。

```shell
.
├── README.md
├── hello.sentinel
└── test/
    └── hello/
        ├── failure.hcl     # hello.sentinel が失敗する場合を見越したテストの入力や期待する挙動（設定）
        └── success.hcl     # hello.sentinel が成功する場合を見越したテストの入力や期待する挙動（設定）
```

## Run tests
テストを実行する際には、`sentinel test` コマンドを用いますが、テスト対象のポリシーコードを個別に指定することが可能です。 \
例えば、上記のディレクトリ構成のようになっており、`hello.sentinel` というポリシーコードをテストしたい場合には以下のようにポリシー名を指定します。

```shell
$ sentinel test -verbose -run hello
```

`-verbose` オプションを付与する事で、ポリシーコード中の `print()` 関数の結果や、詳細な出力を得ることができます。\
また、数多くのポリシーを開発する際に、リグレッションを防ぐために一括でテストを行うような場合には、以下のようにワイルドカードを利用することができます。

```shell
$ sentinel test -verbose policies/common/*
```

## 参考リンク
- [Testing](https://developer.hashicorp.com/sentinel/docs/writing/testing)
- [Testing (tutorial)](https://developer.hashicorp.com/sentinel/tutorials/get-started/testing)
