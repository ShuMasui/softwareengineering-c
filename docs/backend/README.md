# backend

Web API と、サーバ側の処理（認証・バッチ・メール・ファイル）。**DB・認証・メールに接続するのはここだけ**で、ブラウザも受付アプリも同じAPIを呼ぶ。

技術: REST API、HTTPS は設計書3章のまま。次の点は設計書から変えている（[決定ログ](../engineering/decisions.md)）。

- 言語と置き場所: **Go**、**AWS Lambda**（ENG-012。提出版は Node.js + TypeScript と Nginx）
- ライブラリ: **Wire**（依存性の注入）、**AWS SDK for Go v2**、**google/uuid**（IDは UUID v7）。これ以外は未選定（N-18）
- DB: **Amazon DynamoDB**（ENG-001。提出版は MySQL）
- 認証: **Amazon Cognito**（ENG-002〜005。提出版は Microsoft Entra ID）
- メールの送信手段: 未決（N-09。提出版は Microsoft Graph API。希望機能）

## 設計書上の担当範囲

| 機能ID | 機能 | 必須/希望 | 設計書 |
|---|---|---|---|
| A_A0001 | サインイン・サインアウト（ロール判定を含む） | 必須 | 3章, 10.1 |
| A_S0001 | 出欠受付（受付の開始・終了、受付トークン、申告の判定と記録） | 必須 | 6.1〜6.3 |
| A_S0002 | 学生の出欠確認（API） | 必須 | 4章 |
| A_S0003 | 教員の出欠確認（API） | 必須 | 4章 |
| A_S0004 | 出欠修正（理由必須、履歴、楽観的排他） | 必須 | 4章, 6.2 |
| A_S0005 | 学生・学生証登録 | 必須 | 4章 |
| A_S0006 | 授業・授業回・受講学生登録 | 必須 | 4章 |
| A_F0001 | 受講学生名簿の取込 | 必須 | 9.1 |
| A_B0001 | 受付自動締切（1分ごと） | 必須 | 9.2 |
| A_F0002 | 出欠記録の書き出し | 希望 | 9.1 |
| A_M0001 | 欠課警告メール（毎日18:00） | 希望 | 9.2, 9.3 |

## 守る約束

- 判定（出席／遅刻／欠課）はサーバ時刻で行う。
- 学生ID・教員IDはサーバが認証トークンから決め、全APIで担当授業かを確かめる。
- 全タッチ（エラー含む）をタッチログに残す。
- 応答性能: サーバ処理0.2秒以内を目標。
- セッションは最終操作から30分で失効。ログは操作ログ・APIアクセスログを1年保存し、日次でアーカイブする。
- CSRF: Cookie を SameSite=Lax にし、更新系APIに CSRF トークンを付ける（Cognito のトークンとの関係は未決。N-08）。
- SQLインジェクション: DynamoDB は SQL を使わない。PartiQL を使う場合はパラメータ付きにする。
- ユーザープールは学生用と教員用の2つ。どちらのプールのトークンかでロールを決め、ログインID（メールアドレス）から User、Student / Teacher を引く（ENG-003, ENG-004, ENG-006）。
- 認証の処理は1か所に隔離する。Cognito は最終フェーズでつなぐので、それまでは仮の認証で動かす（ENG-005, N-07）。
- 「担当授業」は ClassTeacherMembership で判定する。授業と教員は多対多（ENG-006）。

## 関連文書

- 共有: [DBスキーマ](../engineering/system/db-schema.md)、[ドメインモデル](../engineering/system/domain-model.md)、[メッセージ台帳](../engineering/catalog/messages.md)、[コード台帳](../engineering/catalog/codes.md)
- [api.yml](api.yml): API仕様（OpenAPI 3.0。案）。ユーザーの作り方、セッションと CSRF も冒頭の説明に書いてある
- 決定事項: [decisions.md](decisions.md)
