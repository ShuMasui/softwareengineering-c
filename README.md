# 出欠管理システム

学生証（FeliCa）を教室のカードリーダーにかざして出席を申告し、出欠の判定・欠課時数の集計・確認ができるシステム（F班）。

- **状態**: 設計ドキュメントとモックまで。実装はこれから。
- **構成**: Webアプリ（TypeScript、React + Vite + Tailwind CSS）、Web API（Go、AWS Lambda）、教室PCの受付アプリ（C#）。DB は Amazon DynamoDB、認証は Amazon Cognito、インフラは Terraform。
- **ドキュメント**: [docs/README.md](docs/README.md) が入口。提出版の基本設計書は [docs/engineering/basic-design.md](docs/engineering/basic-design.md)。提出版から変えた点は [docs/engineering/decisions.md](docs/engineering/decisions.md)。

## 画面モックを見る

操作できるモックです（画面の遷移、入力、エラーの分岐）。リポジトリのルートで次を実行して、ブラウザで開きます。

```bash
python3 -m http.server 8000 --bind 127.0.0.1 --directory docs
```

- Webアプリ: http://localhost:8000/frontend/mocks/
- 受付アプリ: http://localhost:8000/windows/mocks/
