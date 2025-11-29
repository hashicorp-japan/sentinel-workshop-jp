# Import 2: Static imports

この章では、Sentinel の import 機能のうち、Static Imports (静的インポート) について学習します。 \
Staic Imports は、外部の静的ファイルを Sentinel のポリシーコードに取り込む機能です。 \
これを利用することで、ポリシーコード内に記述をすると可読性が下がってしまうようなデータや、環境ごとのパラメータなどを切り出すことができるようになります。

ここでは、[前章（Import 1: Standard imports）](./imports-standard.md) で取り扱った [`ifconfig.io`](https://ifconfig.io) のレスポンスを利用して、\
ポリシコード外部に定義した静的ファイルを取り扱うポリシーコードを実装してみます。

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

上記のようなレスポンスを返す ifconfig.io からのレスポンスに対して、`ua` フィールド（User-Agent）の値を `Go-http-client/2.0` のみ許可するようなシナリオを考えます。 \
まず、許可する User-Agent の一覧を記載した JSON ファイルを作成します。 \
（慣例的に、Static Import で利用する外部の静的ファイルは、`imports/` と呼ばれるディレクトリにひとまとまりにすることが多いです）

```shell
% mkdir -p imports
% cat >> imports/allowed_ua_list.json <<EOF
{
    "user-agents": [
        "Go-http-client/2.0"
    ]
}
EOF
```

次に、上記の JSON ファイルを読み込むために、Sentinel の設定ファイルに以下を追記します。 \
この定義により、ポリシーから `imports/allowed_ua_list.json` 内のデータにアクセスすることが可能になります。

```shell
% cat >> sentinel.hcl <<EOF
import "static" "allowed_ua" {
    source = "./imports/allowed_ua_list.json"
    format = "json"
}
EOF
```

最後に、ポリシーコードから static import した JSON データにアクセスをしてみます。

```shell
% cat >> static-import.sentinel << EOF
import "allowed_ua"
import "http"
import "json"

// import したデータに対して key を指定してアクセスし、value を変数として格納
allowed_user_agents = allowed_ua["user-agents"]

resp = http.get("https://ifconfig.io/all.json")
r = json.unmarshal(resp.body)

print("expect User-Agent:", allowed_user_agents[0])
print("got User-Agent:", r.ua)

main = rule {
    r.ua == allowed_user_agents[0]
}
EOF
```

これを `-trace` オプション付きでポリシー評価することで挙動を確認します。

```shell
$ sentinel apply -trace static-import.sentinel
```

出力結果を確認することで、`imports/allowed_ua_list.json` 内のデータが Sentinel コード内で取得され、その値を元に外部 API からの値との付き合わせが行われていることがわかるかと思います。\
さらに、静的ファイルである `imports/allowed_ua_list.json` で定義した許可された User-Agent の値を `curl/8.7.1` などに変更することによって、ポリシー評価結果が FAIL することが確認できます。

上記の例では、非常にシンプルなデータ構造の JSON ファイルを取り扱いましたが、実際のポリシー開発においては以下のようなユースケースに対して Static Import を用いることが多いです。
- 許可/拒否されたマシンスペックの一覧
- 許可/拒否されたデプロイリージョンの一覧
- IP Allow/Deny List
- ユーザの属性情報

また、Static Import では階層化されたデータ構造に対して直接キーを指定して import を行うことも可能です。

上記の例では、import したデータに対して key を指定し、対応する value を変数として格納していましたが、import の方法を以下のように変えることによりより簡略化することもできます。\
先ほど作成した `static-import.sentinel` を以下のように変更してみましょう。

```shell
% cat > static-import.sentinel << EOF
import "allowed_ua/user-agents" as userAgents
import "http"
import "json"

resp = http.get("https://ifconfig.io/all.json")
r = json.unmarshal(resp.body)

print("expect User-Agent:", userAgents[0])
print("got User-Agent:", r.ua)

main = rule {
    r.ua == userAgents[0]
}
EOF
```

これを再度ポリシー評価を行うことでも、同様の結果となることがわかります。

```shell
$ sentinel apply -trace static-import.sentinel
```

これは、複数のポリシーで利用されるようなデータを持つ静的な外部ファイルに対しても、import する側で key を指定することで部分的に読み込むことが可能であることを意味します。\
これにより、複雑なデータ構造を持つ静的な外部ファイルであっても、ポリシーコードや関連する外部ファイルの可読性を保つことができます。


## 参考リンク
- [Static Imports](https://developer.hashicorp.com/sentinel/docs/extending/static-imports)
- [`http`](https://developer.hashicorp.com/sentinel/docs/imports/http)
- [`json`](https://developer.hashicorp.com/sentinel/docs/imports/json)
