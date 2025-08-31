# Import 3: Module imports

この章では、Sentinel の import 機能のうち、Module Imports (モジュールインポート) について学習します。 \
Module Imports は、ポリシーコード外部に存在するポリシーコードを取り込む機能です。

Sentinel では、実装したポリシーコードの再利用性を高めるために、ポリシーコードをモジュールとして取り込むことができます。

また、Sentinel では `func()` ブロックを利用して、ユーザ定義の関数を作成することが可能であるため、関数のみを定義した `*.sentinel` ファイルをモジュールとしてインポートすることもあります。 \
これは共通的なロジックのみを別の `*.sentinel` ファイルとして切り出すことでモジュール化し、再利用している形になります。

ここでは、[`runtime` import](https://developer.hashicorp.com/sentinel/docs/imports/runtime) を利用した関数を自作し、それをモジュールとしてインポートするポリシーコードを実装してみます。 \
`runtime` インポートの機能として、`runtime.version` により `sentinel` コマンドが実行されている環境（Sentinel ランタイム）のバージョン情報を取得することができます。\
これを利用して、Sentinel バージョンが想定されたバージョンであるかをチェックするためのポリシーコードを実装してみましょう。

まずは、ポリシーコードから呼び出されるモジュール側を実装します。\
ここでは `get_sentinel_version` という名前の関数を `func()` ブロックにより定義しています。

```shell
$ mkdir -p functions

$ cat >> functions/helpers.sentinel <<EOF
import "runtime"

get_sentinel_version = func() {
    return runtime.version
}
EOF
```

次に、上記のモジュールをポリシーコード内で読み込むことができるように、設定ファイルへ以下を追記します。 \
これは、`source` で指定された Sentinel コードを `helper` という名前をつけモジュールとしてインポートしていることを意味しています。

```shell
$ cat >> sentinel.hcl <<EOF
import "module" "helper" {
    source = "./function/helpers.sentinel"
}
EOF
```

最後に読み込んだモジュールをポリシーコードから呼び出してみます。 \
自作した関数により得られた Sentinel ランタイムのバージョンが、`0.40.0` であることを確認するポリシーコード `ensure-sentinel-is-latest.sentinel` を実装します。

```shell
$ cat >> ensure-sentinel-is-latest.sentinel <<EOF
import "helper"

main = rule {
    helper.get_sentinel_version() == "0.40.0"
}
EOF
```

これを `-trace` オプション付きでポリシー評価することで挙動を確認します。

```shell
$ sentinel apply -trace
```

出力結果を確認することで、モジュールの評価は `sentinel` コマンド実行時の最初に行われていることがわかるかと思います。


## 参考リンク
- [Modules](https://developer.hashicorp.com/sentinel/docs/extending/modules)
- [Functions](https://developer.hashicorp.com/sentinel/docs/language/functions)
- [`runtime`](https://developer.hashicorp.com/sentinel/docs/imports/runtime)
