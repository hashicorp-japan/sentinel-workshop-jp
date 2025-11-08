# Tests 2: mocking data
この章では、Sentinel のテストを行う際に、mock データを渡す方法について学習します。

まずは Sentinel におけるテストの理解を深めるために、極めてシンプルなポリシーコードを実装し、簡単なテストコードを作成してみましょう。

```shell
$ cat <<EOF > ensure-wednesday.sentinel
import "time"

main = rule {
    time.now.weekday_name == "Wednesday"
}
EOF
```

次に、この「今日が水曜日であること」をチェックするポリシーコードに対するテスト設定ファイルを作成します。 \
ここでは成功および失敗の２つのテストケースのみ想定していますが、ポリシーコードの内容により複数のテストケースが必要となる場合には、テストケースの数だけ設定ファイルを用意します。 \

なお、mock データは各テスト設定ファイル内で `mock` ブロックを利用することによりポリシーのテスト時に読み込ませることが可能であるため、\
mock データが異なるような場合にはテストケースを分離することが望ましいと言えます。

```shell
$ mkdir -p ./test/ensure-wednesday/
$ touch ./test/ensure-wednesday/success.hcl
$ touch ./test/ensure-wednesday/failure.hcl
```

最後に、各テストケースで与えるべき mock データを考えます。 \
`success.hcl` は `ensure-wednesday.sentinel` の main ルール評価結果が `true` となる場合を想定すれば良いため、以下のようになります。

```shell
% cat <<EOF > ./test/ensure-wednesday/success.hcl
mock "time" {
    data = {
        now = {
            weekday_name = "Wednesday"
        }
    }
}

test {
    rules = {
        main = true
    }
}
EOF
```

`failure.hcl` はポリシー違反が起こる場合、つまり main ルール内の `time.now.weekday_name == "Wednesday"` が成立しない場合を想定する形となります。 \
ポリシー違反時を想定したテスト用設定ファイル（`failure.hcl`）の中で定義する `mock.data` により、異なるデータを与えることで、エラー時の挙動を模擬することが可能です。

```shell
% cat <<EOF > ./test/ensure-wednesday/failure.hcl
mock "time" {
    data = {
        now = {
            weekday_name = "Friday"
        }
    }
}

test {
    rules = {
        main = false
    }
}
```

これらの各テストケースおよび mock データを用いて `ensure-wednesday.sentinel` のテストを行うと以下のようにどちらも PASS となることがわかります。

```shell
$ sentinel test -verbose -run ensure-wednesday
PASS - ensure-wednesday.sentinel
  PASS - test/ensure-wednesday/failure.hcl
  PASS - test/ensure-wednesday/success.hcl
```

## 参考リンク
- [mocking](https://developer.hashicorp.com/sentinel/docs/writing/testing#mocking)
- [mocking static data](https://developer.hashicorp.com/sentinel/docs/configuration#mocking-static-data)
