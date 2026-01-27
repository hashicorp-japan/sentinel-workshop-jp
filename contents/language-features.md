# Sentinel Language

この章では、Sentinel でポリシー開発を進める上で必要となる言語文法について学習します。

Sentinel はポリシー開発のフレームワークであるため、`sentinel` コマンドにより提供される機能として以下のような側面を持ちます。
- Sentinel 言語で記述されたファイル（＝ポリシーコード）を作成するための機能（フォーマッタなど）
- ポリシーコードを解釈し実行する機能（インタプリタ/ランタイムとしての機能）
- モックなども含め、ポリシーコードをテストする機能

つまり、Python や Go などの一般的なプログラミング言語と Sentinel とはほぼ同様に考えることができ、他言語での共通的な概念（各種ライブラリ利用やユニットテストなど）は Sentinel においても存在しています。 \
Sentinel によるポリシー開発を行う際には、Sentinel を一つの言語として捉え、型（type）、変数（およびそのスコープ）、制御構文（ループや条件分岐）、サブルーチンなどの基本構文を利用しつつ「Sentinel 言語」での実装が必要となります。 \

ここでは、プログラミング言語としての Sentinel の基本的な文法について実例を交えつつ学習していきます。\
なお、Sentinel に関する文法は、言語使用を含め公式ドキュメントにも詳細な記載があるため、この章ではポリシー開発のスタートへのポイントとなる点に限定し、すべての言語使用をカバーする目的ではない点に留意してください。

## 型（type）
Sentinel において利用することができるデータ型は以下のとおりです。
- [Boolean](https://developer.hashicorp.com/sentinel/docs/language/values#boolean) (`true` / `false`)
- [Integer](https://developer.hashicorp.com/sentinel/docs/language/values#integer)
- [Float](https://developer.hashicorp.com/sentinel/docs/language/values#float)
- [String](https://developer.hashicorp.com/sentinel/docs/language/values#string)

変数は `=` を利用し値を代入します。

```go
isExternal = true
username = "alice"
userid = 123
age = 10.5
```

また、これらのデータ型を組み合わせたもの（複合型）として以下を利用することも可能です
- [Map](https://developer.hashicorp.com/sentinel/docs/language/maps)
- [List](https://developer.hashicorp.com/sentinel/docs/language/lists)

```go
user = {"name": "alice"}
// 複数行で定義する際には、trailing comma が必要になる点に注意するようにしてください
userInfo = {
    "name": "alice",
    "id": 123,
}

initials = ["a", "b", "c"]
// 複数行で定義する際には、trailing comma が必要になる点に注意するようにしてください
code = [
    "alpha",
    "bravo",
    "charie",
]
```

なお、`int()`, `float()`, `string()`, `bool()` の組み込み関数を利用することで、データ型のキャストについても行うことが可能です（[参考](https://developer.hashicorp.com/sentinel/docs/language/values#type-conversion)）


## 変数およびそのスコープ
変数名については、数値および記号から開始することはできませんが、`_` は文字として扱われるため変数名として利用することができます。 \
（変数名については慣例的に camelCase が利用されることが多いですが、言語仕様上の制約はありません）

また、Sentinel においては、定数の概念はないため、同一スコープ内で変数に対して再代入が可能な点に注意して下さい。

```go
a = 1       // a = 1
b = a       // b = 1

a = "value" // a = "value", b = 1
c = a       // c = "value", b = 1
```

Sentinel において、ブロック内部のスコープについては注意が必要です。\
後述の通り、Sentinel においては for, if, func などブロックを伴った制御構文を利用することができますが、制御構文ごとにブロック内での変数スコープは異なります。
- `for`: <https://developer.hashicorp.com/sentinel/docs/language/loops#scoping>
- `if/else`: <https://developer.hashicorp.com/sentinel/docs/language/conditionals#scoping>
- `func`: <https://developer.hashicorp.com/sentinel/docs/language/functions#scoping>


## ループ（for）
`for` を利用して複合型（Map および List）をイテレーションすることが可能です。

### List の場合

```go
count = 0
for [1, 2, 3] as n {
    count += n
}
print(count) //-> 6
```

なお、`as n` の箇所を `as idx, n` としてイテレーションしている要素の index を取得することも可能です。\
List の要素を追加する場合には、組み込み関数の [`append()`](https://developer.hashicorp.com/sentinel/docs/functions/append) を利用することができます。

### Map の場合

```go
for {"a": "alpha", "b": "bravo"} as pcode {
    print(code)
}

for {"a": "alpha", "b": "bravo"} as key, value {
    print(key, "as in", value)
}
```

また、value のみを取得する場合には、`_` を利用して明示的に key を捨てることが可能です。 \
（`as key, value` でループを回し、`for` ブロック内で変数 `key` を参照しなくてもエラーとはなりません）

```go
for {"a": "alpha", "b": "bravo"} as _, value {
    print(value)
}
```

Map に対して要素を追加する際には、変数に対して Key 名を指定し代入を（`m[key] = value` の形）行うことで可能で、逆に要素を削除する際には組み込み関数の [`delete()`](https://developer.hashicorp.com/sentinel/docs/functions/delete) を利用します。\
他にも、Map 型の変数に対しては、[`keys()`](https://developer.hashicorp.com/sentinel/docs/functions/keys) で Key のみを取り出し、[`values()`](https://developer.hashicorp.com/sentinel/docs/functions/values) で Value のみを取り出すことなども可能です。


## 条件分岐

### if/else を利用する場合
`if ... else` および `if ... else if ... else` を用いて評価式に応じた条件分岐を利用することが可能です。 \
`if` および `else if` ブロックの後の評価式結果が `true` となる場合にのみ、対応ブロック内の処理へ分岐します。

```go
if sizeGb <= 10 {
    print("Snapshot only")
} else if 10 < sizeGb and sizeGb <= 500 {
    print("Daily Incremental")
} else if 500 < sizeGb and sizeGb <= 2000 {
    print("Cross-region Replication")
} else {
    print("Architecture Review Required")
}
```

なお、if/else if/else では改行を挟むことはできない点に注意してください。以下の構文はエラーとなります。

```go
if isValidSize {
    print("backup configured properly")
}
else {
    print("check backup design")
}
```

[論理演算](https://developer.hashicorp.com/sentinel/docs/language/boolexpr#logical-operators)については `and`, `or`, `xor`, `not` を利用することができ、[比較演算子](https://developer.hashicorp.com/sentinel/docs/language/boolexpr#comparison-operators)としては `==`, `!=`, `<`, `<=`, `=>`, `>`, `is`, `is not` が利用可能です。

その他、boolean の戻り値を取るものとして、[`contains/in`](https://developer.hashicorp.com/sentinel/docs/language/boolexpr#set-operators), [`matches`](https://developer.hashicorp.com/sentinel/docs/language/boolexpr#matches-operator) などがあり、if/else ブロックの評価式として併用されることも多いです。


### case/when
`case` ブロックを利用することで、特定パターンの条件分岐に対応することが可能です。

```go
case env {
    when "dev", "development":
        print("Non Production")
    when "stg", "staging":
        print("Staging")
    when "prd", "prod", "production":
        print("Production")
    else:
        print("Unknown")
}
```

また、より単純なパターンマッチの場合には、`case` に評価対象を取らず、以下のように記述することも可能です。

```go
case {
    when 9 <= hour and hour < 18:
        print("business hour")
    else:
        print("out of business")
}
```

## サブルーチン（func）
ここまで見てきた通り、Sentinel ではいくつかの[組み込み関数](https://developer.hashicorp.com/sentinel/docs/functions)がありますが、`func` を利用することで、ユーザ定義の関数を定義することも可能です。

制御構文として `func` を利用し関数定義を行う場合には、関数に名前を付与する必要があります。 \
関数は引数を取ることも可能ですが、引数のデフォルト値については利用できない点については注意が必要です。 \
戻り値については、`return` を用います。

```go
// 関数定義
func double(n) {
    return n * 2
}

// 関数の呼び出し
print(double(10)) //-> 20
```

関数は変数として扱うことも可能で、この場合には無名関数として利用することも可能です。

```go
p = func(x) {return x * x}
p(10) //-> 100
```

なお、この場合には、関数名としては変数名が利用されるため、上述の名前付き関数（named function）は利用することができません。\
そのため、以下の構文はエラーとなります。

```go
// NG
d = func double(x) {
    return x * 2
}
//-> expected '(', found 'IDENT' double (and 1 more errors) のエラーとなる
```

関数内で自分自身を呼び出すことも可能であるため、再帰関数の実装もすることができます。

```go
// ユークリッドの互助法
gcd = func(a, b) {
    if b == 0 {
        return a
    }
    return gcd(b, a % b)
}

print(gcd(63,18))   //-> 9
```
