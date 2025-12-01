# ポリシーコードの開発

## Development digests


## Directory layouts
通常、インフラ構成を定義した Terraform コードが格納されるリポジトリと、Sentinel のポリシーコードが格納されるリポジトリとは、以下のような理由からそれぞれ独立してメンテナンスを行われます。
- インフラ構成のライフサイクルとポリシー定義のライフサイクルとは一般的に異なり、これらが依存し合わないようにするため
- Terraform 開発者と Sentinel 開発者との主管範囲や責務が異なるため

ポリシーコードのリポジトリ構成に、フレームワークのような厳密な制約はありませんが、以下のような形が取られることが多いです。

```shell
.
├── functions/                  # ヘルパー関数などを含むライブラリ
├── imports/                    # Sentinel ポリシーがロジック内で評価する際に利用可能なインポートデータ
├── policies/                   # Sentinel ポリシー定義
│   ├── gcp/
│   │   ├── test/
│   │   ├── 001-xxx.sentinel
│   │   │   # ...
│   │   └── 010-xxx.sentinel
│   ├── aws/
│   │   ├── test/
│   │   ├── 001-xxx.sentinel
│   │   │   # ...
│   │   └── 010-xxx.sentinel
│   ├── azure/
│   │   ├── test/
│   │   ├── 001-xxx.sentinel
│   │   │   # ...
│   │   └── 010-xxx.sentinel
│   │   # ...
│   ├── hcl/
│   └── hcp-terraform/
├── policy-sets/                # 適用対象のポリシーのセット定義
│   ├── develop/
│   │   └── sentinel.hcl
│   └── global/
│       └── sentinel.hcl
├── README.md                   # Project root
└── .gitignore
```

- **`policies/`**
  - クラウドプロバイダごとに整理された Sentinel ポリシーファイルのサブディレクトリを保持するトップレベルディレクトリ
  - ユニットテストも含まれます
- **`policy-sets/`**
  - 特定のポリシーセットを表すサブディレクトリを含むトップレベルディレクトリ
  - ポリシーセットは HCP Terraform に接続されるために必要となります
- **`functions/`**
  - Sentinel ポリシーファイルにインポートしてポリシー開発を容易にする「ヘルパー」（再利用可能）関数を含むトップレベルディレクトリ
- **`imports/`**
  - Sentinel ポリシーがインポートして評価することができる静的インポートデータを含むトップレベルディレクトリ

## Policy Graduations
Policy の開発においても、一般的なアプリケーション開発や Terraform コード開発のように適用範囲を段階的に広げていく開発アプローチを取りたいケースは多いかと思います。
（例：テスト用環境で正常にガードレールとして機能することを確認したのち、本番環境で適用したい、など）

特に、HCP Terraform 上での実行においては、Policy Set の設定として適用先の Project や Workspace を限定することができるため、
テスト用および本番用とで Policy Set を分離することによってこのような要件を実現することができます。

![Policy Set Scope - UI](../assets/images/policy-set-scopes.png)

即ち、上記のディレクトリ構成を例とした場合、
1. `policy-sets/develop/sentinel.hcl` はポリシー実装と同時に反映
2. develop policy-set が適用されている Project や Workspace で実際のポリシー評価の確認（Acceptance）
3. develop 環境でのポリシー適用が問題ないと判断したのち、`policy-sets/global/sentinel.hcl` へポリシー設定を反映（Graduation）

という VCS を中心とした開発フローによって、develop 環境のみ -> global(production) 環境というポリシー昇格/本番リリースを行うことが可能となります。

このような段階的なアプローチは、Policy 開発者の意図しない挙動により HCP Terraform 利用者への影響を最小化することができるだけでなく、\
Policy Set の柔軟さによりアプリケーション開発などと同様の開発/リリースフローを維持することができるという側面も持っています。

## Naming conventions

Policy Names
- `enforce-*`
  - 必要または強制により何かを実行させる。
  - パターンや戦略を強制する
- `require-*`
  - 特定の目的のために必要とする。
  - あるプロパティの設定/未設定を必要とする
- `restrict-*`
  - 制限を設けて、管理下に置く。
  - プロパティの値を設定できる/できないように制限する

Test files
- 特定のポリシーに対するテストを `policies/<クラウドプロバイダ>/test/<ポリシー名>/` フォルダ内に作成します。
- テストケースファイル（`*.hcl`）は `"test-*"` というプレフィックスで命名します。
- モックデータファイル（`*.sentinel`）は `"mock-*"` というプレフィックスで命名します。


## CI
`hashicorp/setup-sentinel`

リポジトリ内の全てのポリシーをテストする場合:

```shell
sentinel test ./policies/*
```

Sentinel CLI にはネイティブの再帰的フォーマットオプションはありませんが、次のような方法で対応できます：

```sh
find . -name "*.sentinel" -type f | xargs sentinel fmt
```


## 参考リンク
- [VS Code Plugin](https://marketplace.visualstudio.com/items?itemName=HashiCorp.sentinel)
