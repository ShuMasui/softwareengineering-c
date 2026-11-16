# 共有の決定事項

2つ以上の領域（backend / frontend / windows）に関わる決定を記録する。1つの領域だけの決定は、その領域の `decisions.md` に書く。

## 決定ログ

IDは `ENG-` ＋ 3桁の連番。新しいものを下に足す。「提出版との差」は、基本設計書（提出版）と違う場合だけ書く。

| ID | 日付 | 決定事項 | 理由 | 影響する領域 | 提出版との差 |
|---|---|---|---|---|---|
| ENG-001 | 2026-11-11 | DBは MySQL ではなく **Amazon DynamoDB** を使う | 設計上の裁定（ユーザー） | backend | 提出版は MySQL（3章）。DynamoDB にはビュー・外部キー・自動採番・SQL がないため、[db-schema.md](system/db-schema.md) を論理スキーマとして改めた。欠課時数のビュー（`absence_summary_v`）は、backend の計算に変わる |
| ENG-002 | 2026-11-11 | 認証・アクセス管理は **Amazon Cognito のID・パスワード管理**にする | 設計上の裁定（ユーザー） | backend, frontend, windows | 提出版は Microsoft Entra ID（OpenID Connect）で、ID・パスワードを本システムで持たない（3章, 10.2）。変更後は Cognito がID・パスワードを管理する |
| ENG-003 | 2026-11-11 | ログインIDには **Microsoft のメールアドレス**を使う。Microsoft 側のサインイン（Entra ID）は使わない | 設計上の裁定（ユーザー） | backend, frontend, windows | 提出版は、サインインしたメールアドレスが `student_mst` / `teacher_mst` にあるかでロールを判定（10.1）。変更後は、メールアドレスは本人のレコードを引くためだけに使う（ロールは ENG-004） |
| ENG-004 | 2026-11-11 | Cognito のユーザープールは**学生用と教員用の2つ**に分け、サインイン画面は **Hosted UI** を使う | 設計上の裁定（ユーザー） | backend, frontend, windows | 提出版の「サインイン」画面（A_A0001_SD0001）は Microsoft 365 への入口。変更後は、学生用・教員用の Hosted UI への入口になる。ロールは、どのユーザープールのトークンかで決まる |
| ENG-005 | 2026-11-11 | Cognito との接続は**最終フェーズ**に行う。それまでは仮の認証で開発する | 設計上の裁定（ユーザー） | backend, frontend, windows | 提出版にない。認証の処理は1か所に隔離し、後から Cognito に差し替えられるようにする |
| ENG-006 | 2026-11-11 | ドメインモデルは、ユーザーが示した図（3つのコンテキスト、集約ルート7つ、エンティティ1つ）を使う。[domain-model.md](system/domain-model.md) に起こした | 設計上の裁定（ユーザー） | backend, frontend, windows | 提出版にない User と ClassTeacherMembership が加わる。授業と教員は多対多（提出版は授業に担当教員1人）。出欠は学生ではなく受講（StudentClassMembership）を参照し、授業回（AttendanceSession）が持つ |
| ENG-007 | 2026-11-11 | ドメインモデルの図にない学生証・時限・出欠修正履歴・タッチログは、図に足さず、**提出版の関係のまま**扱う（学生証は学生、出欠修正履歴は出欠、タッチログは授業回に紐づく。時限は授業が参照する） | 設計上の裁定（ユーザー） | backend | 提出版との差はない（ドメインモデルの図に載せないだけ） |

## 裁定により新たに生じた未決事項

ENG-001〜005 で決める必要が出たこと。決まったら、決定ログに記録し、ここの行を消す（または「決定」と書く）。

| ID | 未決事項 | 仮の決め | 関わる決定 | 主に関わる領域 |
|---|---|---|---|---|
| N-01 | DynamoDB のテーブル構成・キー・GSI | なし。必要なアクセスパターンは [db-schema.md](system/db-schema.md) に整理してある | ENG-001 | backend |
| N-02 | 欠課時数の集計の仕方（ビューが使えない） | backend が出欠から毎回計算する。保存はしない（提出版の方針を保つ） | ENG-001 | backend |
| N-03 | IDの採番（DynamoDB には自動採番がない） | なし | ENG-001 | backend |
| N-04 | Cognito ユーザーの作り方（学生・教員の登録と同時に作るか、初期パスワードの渡し方、パスワードを忘れたとき）。名簿取込で新規登録する学生の扱いも含む | なし | ENG-002, ENG-003 | backend |
| N-05 | サインイン画面で、学生用・教員用の Hosted UI へ分ける方法 | なし | ENG-004 | frontend |
| N-06 | 受付アプリ（WPF）が Hosted UI でサインインする方法 | なし | ENG-004 | windows |
| N-07 | 最終フェーズまでの仮の認証の方式 | なし | ENG-005 | backend, frontend, windows |
| N-08 | セッション30分の失効、CSRF対策（Cookie＋CSRFトークン）と、Cognito のトークンの扱い | なし | ENG-002 | backend, frontend |
| N-09 | 欠課警告メール（希望）の送信手段。提出版は Microsoft Graph API | なし。希望機能なので後回しにできる | ENG-002 | backend |
| N-10 | 3つの CTX の名前（図では未記入） | 仮に「ユーザー」「授業」「出欠」と呼ぶ | ENG-006 | engineering |
| N-11 | User が持つ項目（ログインID、Cognito のユーザーとの対応）。Student / Teacher の email を User に移すか | なし。email は提出版のとおり student_mst / teacher_mst に残し、User への参照を足す | ENG-003, ENG-006 | backend |
| N-12 | AttendanceSession と提出版の「授業回」の関係。授業回の日付・時刻・区分（通常・補講・休講）、時間変更を AttendanceSession が持つのか、Class 側に授業回を別に持つのか | AttendanceSession が授業回（class_session_trn）の項目と受付状態をすべて持つ | ENG-006 | backend |
| N-13 | 担当教員が複数いるときの権限（修正・受付・授業の編集）。提出版は担当教員1人 | 担当教員（ClassTeacherMembership にいる教員）は、すべて同じ権限 | ENG-006 | backend |

## 未決事項の状況（設計書11章）

基本設計書11章の未決事項。提出版の11章は書き換えず、決まったらここの状態を更新し、決定ログに記録する。決まるまでは設計書の「仮の決め」で実装する。

| # | 未決事項 | 仮の決め（設計書） | 状態 | 主に関わる領域 |
|---|---|---|---|---|
| 1 | 1回の授業を欠課何時間と数えるか | 90分＝2時間、授業ごとに変更可 | 未決 | backend |
| 2 | 早退を欠課時数にどう換算するか | 区分だけ用意し、換算しない | 未決 | backend |
| 3 | 最初の教員アカウントを誰が登録するか | 初期データとして投入 | 未決（Cognito の教員ユーザープールにも作る必要がある。N-04） | backend |
| 4 | 教員が受付を開始し忘れた授業回の扱い | 自動で欠課にせず「未実施」と表示 | 未決 | backend, frontend |
| 5 | 学生証から読めるIDの種類（FeliCa の IDm か別方式か） | RC-S300 の実機で確認する | 未決 | windows |
| 6 | 受付アプリを Windows デスクトップアプリにする案でよいか | Windowsデスクトップアプリ（代替案: ブラウザ＋WebUSB） | 未決 | windows |
| 7 | 教室PCがネットワークにつながらないときの受付の一時保存 | 扱わない。教員が後で修正 | 未決 | windows, backend |
| 8 | サーバの置き場所（学内かクラウドか）と運用担当 | なし | 一部決定（DynamoDB・Cognito を使うので AWS）。運用担当と、API・画面の置き場所は未決 | engineering |
| 9 | 欠課警告メールのしきい値 | 例: 上限の3分の2 | 未決（希望機能） | backend |
| 10 | R-05-3 の文意（学籍番号のみで登録できない場合と学生証再発行の関係） | なし。要求定義書側で整理する | 未決 | engineering |
