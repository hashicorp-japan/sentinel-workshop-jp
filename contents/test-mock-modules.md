# Tests 3: mocking modules
この章では、Sentinel のテストを行う際に、より実践的に mock を扱うための方法を学びます。

実践的な Sentinel ポリシーのテストでは、テスト対象のポリシーコードに与える mock を `data` ブロックでハードコードするのは可読性や管理性が下がるケースが大半です。 \
また、HCP Terraform や Terraform Enterprise と連携する場合には、mock されるデータの構造も非常に複雑になり、 \
ひとつのポリシーコードに対してそれぞれの mock データを用意するのは現実的ではありません。

Sentinel では、実装したポリシーコードの再利用性を高めるために、ポリシーコードをモジュールとして取り込むことができるため、 \
テストにおいても、mock するデータを `*.sentinel` ファイルから取り込むことができます。
- これは、ファイル名が `*.sentinel` にマッチする場合には、それがポリシーコードを指す場合と、mock ファイルを指す場合とがあることを意味しています
- そのため、慣例的に mock ファイルの場合には、`mock-*.sentinel` というファイル名とされることが多いです


ここでは、極めてシンプルなポリシーコードを実装し、モジュールとして切り出した `*.sentinel` ファイルを mock として取り込んでみます。

```shell
$ cat <<EOF > ensure-monday.sentinel
import "time"

main = rule {
    time.now.weekday_name == "Monday"
}
EOF

$ mkdir -p ./test/ensure-monday/
$ touch ./test/ensure-monday/success.hcl
$ touch ./test/ensure-monday/failure.hcl
```

まず、mock ファイル（ポリシーコードが読み込むモジュール）を、成功時/失敗時のテストケース分用意します。 \
なお、テスト設定ファイルは HCL の形式ですが、mock ファイルについては Sentinel の形式になるため、`data` ブロックで mock する場合とは記述方法が異なる点に留意してください。

```shell
% cat <<EOF > ./test/ensure-monday/mock-success.sentinel
now = {
    "weekday_name": "Monday"
}
EOF

% cat <<EOF > ./test/ensure-monday/mock-failure.sentinel
now = {
    "weekday_name": "Friday"
}
EOF
```

成功/失敗の各テストケースに対する設定では、`mock.module` を用いて、Sentinel 形式の mock ファイルを取り込みます。

```shell
% cat <<EOF > ./test/ensure-monday/success.hcl
mock "time" {
    module {
        source = "./mock-success.sentinel"
    }
}

test {
    rules = {
        main = true
    }
}
EOF

% cat <<EOF > ./test/ensure-monday/failure.hcl
mock "time" {
    module {
        source = "./mock-failure.sentinel"
    }
}

test {
    rules = {
        main = false
    }
}
EOF
```

これらの各テストケースおよび mock データを用いて `ensure-monday.sentinel` のテストを行うと以下のようにどちらも PASS となることがわかります。

```shell
$ sentinel test -verbose -run ensure-monday
PASS - ensure-monday.sentinel
  PASS - test/ensure-monday/failure.hcl
  PASS - test/ensure-monday/success.hcl
```

## 参考リンク
- [mocking](https://developer.hashicorp.com/sentinel/docs/writing/testing#mocking)
- [mocking with Sentinel code](https://developer.hashicorp.com/sentinel/docs/configuration#mocking-with-sentinel-code)
