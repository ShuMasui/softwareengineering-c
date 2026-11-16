# リポジトリ構成（案）

設計書にない。実装に入る前の案で、ユーザーの確認はまだ（[N-15](decisions.md)）。確認前でも、この案に従って実装してよい。

言語とライブラリは、ユーザーが選んだとおり（[ENG-012〜015](decisions.md)）。

| 領域 | 言語・ライブラリ |
|---|---|
| backend | **Go**（AWS Lambda）。Wire、AWS SDK for Go v2、google/uuid。これ以外は未選定（[N-18](decisions.md)） |
| Webアプリ | **TypeScript**。React、Vite、Tailwind CSS |
| 受付アプリ | **C#**。AWS SDK for .NET。残りは実装者が決める |
| インフラ | **Terraform** |

## 構成

```
softwareengineering-c/
├── apps/
│   ├── api/         Web API（Go。AWS Lambda に置く）
│   ├── web/         Webアプリ（TypeScript、React + Vite + Tailwind CSS）
│   └── reception/   受付アプリ（C#。Windows）
├── infra/           インフラ（Terraform。ENG-011、ENG-015）
│   ├── bootstrap/     最初に1回だけ、手元で apply する（state と計画の S3、GitHub の OIDC、IAM ロール）
│   ├── env/prod/      本番。apply-to-prod workflow が apply する
│   └── modules/       env から使う部品（案は N-17）
├── .github/workflows/  apply-to-prod.yml（インフラの plan と apply）
├── docs/            設計ドキュメントとモック
├── CLAUDE.md
└── README.md
```

- `api` は Go のモジュール（`go.mod`）、`web` は npm（`package.json`）、`reception` は .NET のソリューションで、それぞれ別に管理する。
- 設計書の機能ID・画面IDは、ファイル名やコードの名前にそのまま使う（例: 画面 `A_S0002_SL0001` の部品）。
- 文言とコード値は、`docs/engineering/catalog/` の台帳から作る。コードに直接書かない。

## apps/api の中身（Go）

```
apps/api/
├── cmd/
│   ├── api/         API の Lambda（API Gateway のリクエストを受ける）。wire.go と、生成された wire_gen.go を置く
│   ├── batch/       受付自動締切（A_B0001）の Lambda。1分ごとに起動する。wire.go と wire_gen.go を置く
│   └── local/       ローカル用の HTTP サーバ（Lambda と同じハンドラを動かす）
├── internal/
│   ├── handler/     HTTP の入口（MVC のコントローラ）。入力の検査と、認可の確認
│   ├── service/     ドメインのルール。申告の判定、欠課時数の計算、受付の開始・締切
│   ├── repository/  DynamoDB の読み書き。キーの組み立てはここだけ
│   ├── auth/        AuthProvider（Cognito と、開発用の仮認証を切り替える）
│   └── domain/      エンティティ、区分値、メッセージ台帳（文言とID）
└── go.mod
```

- 判定と計算は `service/` に集め、DynamoDB や HTTP に依存させない（単体テストしやすくする）。
- 依存性の注入は Wire で行う。`handler`・`service`・`repository`・`auth` は、インターフェースで受け取る。
- 認証は `auth/` の1か所に隔離する（ENG-005）。
- IDは google/uuid で作る。時刻順に並べたいもの（授業回、タッチログなど）は UUID v7（`uuid.NewV7()`。google/uuid v1.6.0 以上）。

## ローカルでの開発（案）

- DB: DynamoDB Local（Docker）。テーブルは [db-schema.md](system/db-schema.md) のとおり `attendance_app` を作る。
- API: `cmd/local` で HTTP サーバとして起動する。認証は `AUTH_MODE=dev`（リクエストヘッダ `X-Dev-User` の `user_id` で本人になる。本番では無効。N-07）。
- 受付アプリ: サーバの時刻とカードリーダーがなくても動かせるよう、カードの読み取りを差し替えられる形にする（PC/SC の部分だけを別のクラスにする）。
- モックの表示: リポジトリのルートで `python3 -m http.server 8000 --bind 127.0.0.1 --directory docs`。

## テスト（案）

- 必ず単体テスト（`go test`）を書く: 出欠の判定ルール（出席・遅刻・欠課の境目）、欠課時数の計算、申告の確認順序。
- API の結合テストは、DynamoDB Local に対して行う。
