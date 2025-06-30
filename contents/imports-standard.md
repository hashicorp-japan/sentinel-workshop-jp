# Import 1: Standard imports

この章では、Sentinel の import 機能のうち、Standard Imports (標準インポート) について学習します。 \
Standard Import は Sentinel がビルドインで持つライブラリを利用する機能です。 \
(一般的なプログラミング言語における、標準ライブラリや標準パッケージに該当します)

ここでは、[`http`](https://developer.hashicorp.com/sentinel/docs/imports/http) および [`json`](https://developer.hashicorp.com/sentinel/docs/imports/json) の import を利用し、外部 API のレスポンスボディに応じて評価を行う、動的なポリシーコードを実装してみます。

```shell
$ cat <<EOF > standard-import.sentinel


EOF
```

また、Standard Import の際に、パラメータをオーバーライドすることで、デフォルトの挙動から変更することが可能です。 \
以下は、デフォルトでは UTC となっている [`time`](https://developer.hashicorp.com/sentinel/docs/imports/time) を JST に変更する場合の例となります。

```hcl
import "plugin" "time" {
  config = {
    "timezone" = "Asia/Tokyo"
  }
}
```

## 参考リンク
- [Standard Imports](https://developer.hashicorp.com/sentinel/docs/imports)
- [`http`](https://developer.hashicorp.com/sentinel/docs/imports/http)
- [`json`](https://developer.hashicorp.com/sentinel/docs/imports/json)
- [`time`](https://developer.hashicorp.com/sentinel/docs/imports/time)
