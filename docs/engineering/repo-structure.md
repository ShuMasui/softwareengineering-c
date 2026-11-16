# リポジトリ構成（案）

設計書にない。実装に入る前の案で、ユーザーの確認はまだ（[N-15](decisions.md)）。確認前でも、この案に従って実装してよい。ライブラリの細かな選定は、実装のときに決める。

## 構成

```
softwareengineering-c/
├── apps/
│   ├── api/         Web API（Node.js + TypeScript、REST、MVC）
│   ├── web/         Webアプリ（React + TypeScript）
│   └── reception/   受付アプリ（C# / .NET / WPF。Windows）
├── docs/            設計ドキュメントとモック
├── CLAUDE.md
└── README.md
```

- `api` と `web` は、ルートの `package.json`（npm workspaces）でまとめる。`reception` は .NET のソリューションで管理する。
- 設計書の機能ID・画面IDは、ファイル名やコードの名前にそのまま使う（例: 画面 `A_S0002_SL0001` の部品）。
- 文言とコード値は、`docs/engineering/catalog/` の台帳から作る。コードに直接書かない。

## apps/api の中身（MVC）

```
apps/api/src/
├── routes/         HTTP の入口（コントローラ）。入力の検査と、認可の確認
├── services/       ドメインのルール。申告の判定、欠課時数の計算、受付の開始・締切
├── repositories/   DynamoDB の読み書き。キーの組み立てはここだけ
├── auth/           AuthProvider（Cognito と、開発用の仮認証を切り替える）
├── batch/          A_B0001（受付自動締切）。A_M0001（希望）
└── messages/       メッセージ台帳（文言とID）
```

- 判定と計算は `services/` に集め、DynamoDB や HTTP に依存させない（単体テストしやすくする）。
- 認証は `auth/` の1か所に隔離する（ENG-005）。

## ローカルでの開発（案）

- DB: DynamoDB Local（Docker）。テーブルは [db-schema.md](system/db-schema.md) のとおり `attendance_app` を作る。
- 認証: `AUTH_MODE=dev`。リクエストヘッダ `X-Dev-User` の `user_id` で本人になる（本番では無効。N-07）。
- 受付アプリ: サーバの時刻とカードリーダーがなくても動かせるよう、カードの読み取りを差し替えられる形にする（PC/SC の部分だけを別のクラスにする）。
- モックの表示: リポジトリのルートで `python3 -m http.server 8000 --bind 127.0.0.1 --directory docs`。

## テスト（案）

- 必ず単体テストを書く: 出欠の判定ルール（出席・遅刻・欠課の境目）、欠課時数の計算、申告の確認順序。
- API の結合テストは、DynamoDB Local に対して行う。
