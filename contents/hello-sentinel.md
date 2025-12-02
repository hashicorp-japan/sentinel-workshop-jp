# 初めての Sentinel

ここでは、最もシンプルなポリシーを作成し、Sentinel の基本的な考え方を学びます。

Sentinel でのポリシー開発では、`sentinel` CLI を利用します。 \
Sentinel CLI がインストールされていない場合には、[こちら](https://developer.hashicorp.com/sentinel/install) からダウンロードをしてください。 \
ダウンロードしたら unzip して実行権限を付与し、パスを通します。下記は macOS の手順です。

```console
$ unzip sentinel*.zip
$ chmod + x sentinel
$ mv sentinel /usr/local/bin
$ sentinel -version
Sentinel v0.40.0
```

次に任意の作業用ディレクトリを作ります。

```shell
$ mkdir -p sentinel-policies/hello-sentinel
$ cd sentinel-policies/hello-sentinel
```

早速このフォルダに Sentinel のポリシーファイルを作ってみます。 \
Sentinel は最低限二つのファイルが必要です。一つは `sentinel.hcl`、もう一つは `<POLICYNAME>.sentinel` です。

* `sentinel.hcl` の作成

```shell
$ cat <<EOF > sentinel.hcl
policy "hello" {
    source            = "./hello.sentinel"
    enforcement_level = "hard-mandatory"
}
EOF
```

`sentinel.hcl` のファイルは実際のポリシーが定義されているコードの設定を行います。 \
`source` ではポリシーコードへのパスを記述し、`enforcement_level` の指定によりそのポリシーの強制度合いを設定します。

* soft-mandatory
	* ポリシーに引っかかりエラーになった時にそれをオーバーライドして実行を許可するか、拒否するかを選択できるモード
* hard-mandatory
	* ポリシーに引っかかったら必ず実行を拒否するモード
* advisory
	* 実行は許可するが、警告を出すモード

つまり、この `sentinel.hcl` は、Sentinel によるポリシーコード評価の挙動を決定する Sentinel 自身の設定ファイルと考えることができます。 \
Sentinel の設定ファイルに関しては、[Sentinel 設定ファイル](contents/configurations.md) のワークショップも併せて参考にしてください。


* `<POLICYNAME>.sentinel` の作成

```shell
$ cat <<EOF > hello.sentinel
main = rule {
  true
}
EOF
```

ここでは、`hello` という名前のポリシーコードを作成しました。`hello.sentinel` のファイルは実際のポリシーコードになります。 \
Sentinel においては、`*.sentinel` の拡張子を持つファイルにポリシー定義を記載します。　\
各ポリシーコードは、最低一つの `rule` ブロックが含まれ、実際に評価するポリシーの処理を記載します。

上記の `hello.sentinel` というポリシーコードを評価するためには、`sentinel apply` コマンドを利用します。

```shell
$ sentinel apply
Pass - hello.sentinel
```

出力から `hello.sentinel` が評価され、作成したポリシーコードが準拠している（Pass している）ということがわかります。 \
次に、`hello.sentinel` のポリシーコードを以下のように変更して、再度、評価してみます。

```shell
$ cat <<EOF > hello.sentinel
main = rule {
  true == false
}
EOF

# ポリシーの評価
$ sentinel apply
Execution trace. The information below will show the values of all
the rules evaluated. Note that some rules may be missing if
short-circuit logic was taken.

Note that for collection types and long strings, output may be
truncated; re-run "sentinel apply" with the -json flag to see the
full contents of these values.

The trace is displayed due to a failed policy.

Fail - hello.sentinel

hello.sentinel:1:1 - Rule "main"
  Value:
    false
```

これは `rule` ブロック内のロジックが評価された結果として、ポリシーが準拠していないことを意味しています。 \
（`true == false`、つまり `false`） \
また、このことから、`rule` ブロックは記載したロジックの結果が `true` となることを期待しており、この動きによりブロック内のポリシーが準拠されているのか否かを判定しているということがわかります。

ここでは、`true` / `false` という極めてシンプルなロジックを持つポリシーコードを作成しましたが、Sentinel では一般的なプログラミング言語同様の様々な機能（変数や演算子、関数など）を利用することで、\
非常に柔軟なポリシーを実装することができます。

以降の章では、Setinel のポリシーコードを実装していく上で有用な様々な機能や仕様について触れていきます。


## 参考リンク
- [Sentinel Language](https://developer.hashicorp.com/sentinel/docs/language)
- [Rules](https://developer.hashicorp.com/sentinel/docs/language/rules)
- [Sentinel configuration files](https://developer.hashicorp.com/sentinel/docs/configuration)
