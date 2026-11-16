# 出欠管理システム

提出済みの基本設計書（[docs/engineering/basic-design.md](docs/engineering/basic-design.md)）の通りにシステムを開発する。ただし、次の「提出版から変更した事項」は、変更後が正。

@docs/engineering/basic-design.md

## 提出版から変更した事項

上の設計書（提出版）と食い違うときは、こちらを優先する。経緯と詳細は [docs/engineering/decisions.md](docs/engineering/decisions.md) の ENG-001〜006。

- **DB**: MySQL → **Amazon DynamoDB**（ENG-001）。ビュー・外部キー・自動採番・SQLがない。
- **認証**: Microsoft Entra ID → **Amazon Cognito のID・パスワード管理**（ENG-002）。ログインIDは Microsoft のメールアドレス（ENG-003）。ユーザープールは学生用と教員用の2つで、サインインは Hosted UI（ENG-004）。**Cognito との接続は最終フェーズ**で、それまでは仮の認証で開発する（ENG-005）。
- **ドメインモデル**: [docs/engineering/system/domain-model.md](docs/engineering/system/domain-model.md) の図を使う（ENG-006）。User と ClassTeacherMembership が加わり、授業と教員は多対多。出欠は学生ではなく受講（StudentClassMembership）を参照し、授業回（AttendanceSession）が持つ。

## ドキュメントの構成

詳細は [docs/README.md](docs/README.md)。

```
docs/
├── engineering/   共有事項: basic-design.md（提出版）、decisions.md、system/（db-schema, domain-model）、catalog/（messages, codes）
├── backend/       Web API・バッチ・認証・メール・ファイル処理の決定事項
├── frontend/      Webアプリ（React）の決定事項、mocks/（画面モック）
└── windows/       受付アプリ（C# / WPF）の決定事項
```

- `engineering/basic-design.md` は**提出版で、書き換えない**。
- 実装が合わせる先は `engineering/system/`（DB・ドメイン）と `engineering/catalog/`（文言・コード値）。文言やコード値をコードに直接書かず、台帳のIDで参照する。
- 開発で決めたことは、該当ディレクトリの `decisions.md` に記録する。2つ以上の領域に関わるものは `engineering/decisions.md`。

## 進め方

- 実装前に、対応する章（機能ID・画面ID・テーブル・メッセージID）を設計書と `docs/` の該当文書で確認する。
- 提出版と食い違う実装が必要になったら、**勝手に変えず先にユーザーに確認する**。承認されたら `decisions.md` に「提出版との差」を記録し、`system/` `catalog/` の該当文書を更新する（提出版は書き換えない）。
- 「必須」の要求を先に実装する。「希望」（F-07 出欠書き出し `A_F0002`、R-03-6 欠課警告メール `A_M0001`）は設計だけ済んでいるので、依頼があるまで実装しない。
- 11章の未決事項と、`engineering/decisions.md` の未決事項（N-01〜）は、書かれている「仮の決め」に従って実装する（例: 1回あたり2時間、早退は換算しない、受け忘れは「未実施」表示）。仮の決めがない未決事項は、実装前にユーザーに確認する。
- 設計書にない事項（リポジトリ構成、ディレクトリ名、ライブラリの細かな選定、テストの方針など）は、決める前にユーザーに確認する。

## 構成（3章）

- Web画面: React + TypeScript / Web API: Node.js + TypeScript（REST、MVC）/ DB: **Amazon DynamoDB** / Webサーバ: Nginx
- 受付アプリ: C#（.NET、WPF）。PC/SC で学生証のIDを読み、結果を画面と音で表示するだけ。判定ロジックは持たない。
- 認証: **Amazon Cognito**（学生用・教員用のユーザープール、Hosted UI）。認証の処理は1か所に隔離し、Cognito は最後につなぐ。
- **DB・認証・メールに接続するのはWebアプリ（API）だけ。** 受付アプリもブラウザも同じAPIを呼ぶ。

## 守るべき設計上の約束

- 出欠の判定（出席／遅刻／欠課）は**サーバ時刻**で行う。クライアントの時刻は使わない。
- 欠課時数は**保存しない**。DynamoDB にビューはないので、backend が出欠から毎回計算する（式は6.1）。公欠は数えない。
- 出欠は「授業回×学生」で1件。二重登録を防ぐ（DynamoDB の条件付き書き込みなど）。
- 全タッチ（エラー含む）を `card_touch_log_trn` に残す。
- 出欠修正は理由必須で、変更前後を `attendance_history_trn` に残す。
- 排他制御は楽観的排他（`last_update_version`）。
- 全エンティティに共通項目（作成者・作成日時・更新者・更新日時・更新バージョン・削除フラグ）を持たせる。
- 「誰のデータを見せるか」は、画面から送られたIDではなく、サーバが認証トークンから決める。全APIで担当授業かを確認する。
- カードIDは画面に出さない。DynamoDB は SQL を使わない（PartiQL を使う場合はパラメータ付き）。UIは日本語のみ。

## 命名規則（2章）

- 機能ID `A_<S|F|M|A|B><4桁>`、画面ID `<機能ID>_<SL|SD|SP><4桁>`、メッセージID `<機能ID>_<E|W|I><6桁>`
- 項目は `_cd` `_id` `_name` `_dt` `_tm` `_datetime` `_div` `_status` `_flg` などの接尾辞。テーブル（DynamoDB のテーブル名）の付け方は物理設計で決める。
- メッセージ文言・コード値は `docs/engineering/catalog/` の台帳（設計書8章が元）をそのまま使う。
