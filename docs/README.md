# docs

出欠管理システムの開発ドキュメント。共有事項は `engineering/`、領域ごとの決定事項は `backend/` `frontend/` `windows/` にまとめる。

```
docs/
├── engineering/        共有事項（全領域で使う）
│   ├── basic-design.md   提出版の基本設計書（固定）
│   ├── decisions.md      共有の決定事項・未決事項の状況
│   ├── repo-structure.md リポジトリ構成の案
│   ├── system/           db-schema、domain-model
│   └── catalog/          画面に出す文言・コード値の台帳
├── backend/            Web API・バッチ・認証・メール・ファイル処理（api.md: API一覧の案）
├── frontend/           Webアプリ（React）
│   └── mocks/            画面モック
└── windows/            教室PCの受付アプリ（C# / WPF）
    └── mocks/            画面モック
```

## 各ディレクトリの役割

| ディレクトリ | 置くもの | 置かないもの |
|---|---|---|
| [engineering/](engineering/README.md) | 複数の領域にまたがる定義（DB、ドメイン、文言、コード値）と共有の決定事項 | 1つの領域だけに関わる決定 |
| [backend/](backend/README.md) | API・バッチ・認証など、サーバ側だけの決定事項 | DB定義、文言（共有なので engineering/） |
| [frontend/](frontend/README.md) | Webアプリだけの決定事項、`mocks/` の画面モック | API仕様の決定、文言 |
| [windows/](windows/README.md) | 受付アプリだけの決定事項（PC/SC、画面、受付の流れ） | API仕様の決定、文言 |

## どの文書が正か

1. [engineering/basic-design.md](engineering/basic-design.md) は**提出版**。書き換えない。
2. 開発で決めたこと・変えたことは、該当ディレクトリの `decisions.md` に記録する。
3. 提出版と食い違う決定は、ユーザーの確認を得てから `decisions.md` に「提出版との差」を書き、`system/` や `catalog/` の該当文書を更新する。
4. `system/` と `catalog/` は提出版の内容を元に起こした**開発中の正**。実装はこちらに合わせる。

## 文書IDの対応

設計書の機能ID・画面ID・メッセージIDは、そのまま全文書で使う（命名規則は設計書2章）。
