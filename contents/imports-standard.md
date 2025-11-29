# Import 1: Standard imports

この章では、Sentinel の import 機能のうち、Standard Imports (標準インポート) について学習します。 \
Standard Import は Sentinel がビルドインで持つライブラリを利用する機能です。 \
(一般的なプログラミング言語における、標準ライブラリや標準パッケージに該当します)

ここでは、[`http`](https://developer.hashicorp.com/sentinel/docs/imports/http) および [`json`](https://developer.hashicorp.com/sentinel/docs/imports/json) の import を利用し、外部 API のレスポンスボディに応じて評価を行う、動的なポリシーコードを実装してみます。

ここでは、外部 API の例として、実行環境の HTTP(S) アクセス元情報を返す [`ifconfig.io`](https://ifconfig.io) を利用します。 \
通常の `curl` コマンドなどを利用すると、以下のような出力が得られるため、これを Sentinel で取り扱います。

```shell
# *** の箇所は実行環境の Global IP が返されます
% curl -s -X GET https://ifconfig.io/all.json |jq .
{
  "country_code": "JP",
  "encoding": "gzip, br",
  "forwarded": "****",
  "host": "****",
  "ifconfig_cmd_hostname": "ifconfig.io",
  "ifconfig_hostname": "ifconfig.io",
  "ip": "****",
  "lang": "",
  "method": "GET",
  "mime": "*/*",
  "port": 10196,
  "referer": "",
  "ua": "curl/8.7.1"
}
```

Sentinel コードによりこの `ifconfig.io` の API を呼び出し、JSON 形式のレスポンスに含まれる `country_code` の値が `JP` であることを確認するポリシーを実装しています。 \
（実行環境が日本ではない場合には、別の国コードであることを確認してみましょう）

```shell
$ cat <<EOF > standard-import.sentinel
import "http"
import "json"

resp = http.get("https://ifconfig.io/all.json")
r = json.unmarshal(resp.body)

// trace 用に出力する
print(r)

main = rule {
    r.country_code == "JP"
}
EOF
```

これを `-trace` オプション付きでポリシー評価することで挙動を確認します。

```shell
$ sentinel apply -trace standard-import.sentinel
```

出力結果を確認することで、Sentinel コードによって、外部 API からの JSON レスポンスを操作できていることがわかります。\
さらに、`main` ルール内で評価する `r.country_code` の値を `JP` 以外に設定し、再度ポリシー評価を行うと、`sentinel apply` の結果が FAIL することも確認できるかと思います。

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
