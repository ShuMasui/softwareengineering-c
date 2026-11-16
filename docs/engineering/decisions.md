# 共有の決定事項

2つ以上の領域（backend / frontend / windows）に関わる決定を記録する。1つの領域だけの決定は、その領域の `decisions.md` に書く。

## 決定ログ

IDは `ENG-` ＋ 3桁の連番。新しいものを下に足す。「提出版との差」は、基本設計書（提出版）と違う場合だけ書く。

| ID | 日付 | 決定事項 | 理由 | 影響する領域 | 提出版との差 |
|---|---|---|---|---|---|
| ENG-001 | 2026-11-11 | DBは MySQL ではなく **Amazon DynamoDB** を使う | 設計上の裁定（ユーザー） | backend | 提出版は MySQL（3章）。DynamoDB にはビュー・外部キー・自動採番・SQL がないため、[db-schema.md](system/db-schema.md) を改めた。欠課時数のビュー（`absence_summary_v`）は、backend の計算に変わる |
| ENG-002 | 2026-11-11 | 認証・アクセス管理は **Amazon Cognito のID・パスワード管理**にする | 設計上の裁定（ユーザー） | backend, frontend, windows | 提出版は Microsoft Entra ID（OpenID Connect）で、ID・パスワードを本システムで持たない（3章, 10.2）。変更後は Cognito がID・パスワードを管理する |
| ENG-003 | 2026-11-11 | ログインIDには **Microsoft のメールアドレス**を使う。Microsoft 側のサインイン（Entra ID）は使わない | 設計上の裁定（ユーザー） | backend, frontend, windows | 提出版は、サインインしたメールアドレスが `student_mst` / `teacher_mst` にあるかでロールを判定（10.1）。変更後は、メールアドレスは本人のレコードを引くためだけに使う（ロールは ENG-004） |
| ENG-004 | 2026-11-11 | Cognito のユーザープールは**学生用と教員用の2つ**に分け、サインイン画面は **Hosted UI** を使う | 設計上の裁定（ユーザー） | backend, frontend, windows | 提出版の「サインイン」画面（A_A0001_SD0001）は Microsoft 365 への入口。変更後は、学生用・教員用の Hosted UI への入口になる。ロールは、どのユーザープールのトークンかで決まる |
| ENG-005 | 2026-11-11 | Cognito との接続は**最終フェーズ**に行う。それまでは仮の認証で開発する | 設計上の裁定（ユーザー） | backend, frontend, windows | 提出版にない。認証の処理は1か所に隔離し、後から Cognito に差し替えられるようにする |
| ENG-006 | 2026-11-11 | ドメインモデルは、ユーザーが示した図（3つのコンテキスト、集約ルート7つ、エンティティ1つ）を使う。[domain-model.md](system/domain-model.md) に起こした | 設計上の裁定（ユーザー） | backend, frontend, windows | 提出版にない User と ClassTeacherMembership が加わる。授業と教員は多対多（提出版は授業に担当教員1人）。出欠は学生ではなく受講（StudentClassMembership）を参照し、授業回（AttendanceSession）が持つ |
| ENG-007 | 2026-11-11 | ドメインモデルの図にない学生証・時限・出欠修正履歴・タッチログは、図に足さず、**提出版の関係のまま**扱う（学生証は学生、出欠修正履歴は出欠、タッチログは授業回に紐づく。時限は授業が参照する） | 設計上の裁定（ユーザー） | backend | 提出版との差はない（ドメインモデルの図に載せないだけ） |
| ENG-008 | 2026-11-16 | ドメインモデルの詳細の項目は自由に決めてよいが、**1エンティティにつきキー以外5つ以内**に収める（必要以上に増やさない） | 設計上の指示（ユーザー） | backend | 提出版の項目を絞った。共通項目（作成者・作成日時・更新者・更新日時・削除フラグ）は持たず、授業は曜日・時限・期間・担当教員を持たない。一覧は [db-schema.md](system/db-schema.md) の「提出版との差」 |
| ENG-009 | 2026-11-16 | Cognito のユーザーは**標準フロー**で作る。登録すると Cognito が仮パスワード付きの招待メールを送り、本人が初回サインインでパスワードを決めて本登録する | 設計上の指示（ユーザー。「仮認証の方式」への回答として受けた） | backend, frontend, windows | 提出版は Microsoft 365 アカウントでサインインするだけで、アカウントの発行は本システムの外 |
| ENG-010 | 2026-11-16 | DynamoDB の形は、[db-schema.md](system/db-schema.md) の案のとおりに**確定**する（単一テーブル `attendance_app`、キー `PK` / `SK`、GSI2つ。IDは `user_id`＝Cognito の sub、授業・授業回は ULID。email は User にだけ。受付トークンは保存しない） | 設計上の確認（ユーザー） | backend | ENG-001・ENG-008 を具体化したもの。提出版との差は db-schema.md の「提出版との差」 |
| ENG-011 | 2026-11-16 | インフラ（AWS）は **Terraform** で実装する。**state（状態）は S3 に置く**。`terraform apply` は GitHub Actions の workflow で行い、**S3 に保存した計画（plan）**を使う（どちらもユーザーの確認済み）。Terraform のコードはユーザーが書き、このリポジトリには workflow を用意する | 設計上の指示（ユーザー） | engineering | 提出版にない（設計書3章は、インフラの作り方を書いていない） |

## 裁定により新たに生じた未決事項

ENG-001〜011 で決める必要が出たこと。決まったら、決定ログに記録し、ここの行を消す。「仮の決め」は、実装を止めないために置いた案で、**ユーザーの確認はまだ**。確認前でも、この仮の決めに従って実装する。

| ID | 未決事項 | 仮の決め | 関わる決定 | 主に関わる領域 |
|---|---|---|---|---|
| N-04 | Cognito ユーザーの作り方の残り（名簿取込で新規登録する学生、招待の再送、パスワードを忘れたとき、DBとの書き込み順、最初の教員） | 先に Cognito にユーザーを作り（招待メールが届く）、次に DynamoDB に書く。DBに失敗したら Cognito のユーザーを消す。再送とパスワード再設定は Cognito の標準機能。最初の教員は、管理者が Cognito に作り、初期データの投入で Teacher と User を作る（設計書11章の未決 #3 も同じ）。詳細は [api.yml](../backend/api.yml) | ENG-002, ENG-003, ENG-009 | backend |
| N-05 | サインイン画面で、学生用・教員用の Hosted UI へ分ける方法 | サインイン画面に「学生」「教員」の2つのボタンを置き、それぞれのプールの Hosted UI に移す（`/auth/login?pool=student` と `pool=teacher`）。[モック](../frontend/mocks/index.html) のとおり | ENG-004 | frontend |
| N-06 | 受付アプリ（WPF）が Hosted UI でサインインする方法 | 既定のブラウザで Hosted UI を開き、認可コード＋PKCE でサインインする（受け取りは localhost のループバックURL）。詳細は [windows/README.md](../windows/README.md) | ENG-004 | windows |
| N-07 | Cognito をつなぐ**前**の仮の認証の方式（ENG-009 は Cognito の登録フローで、これとは別。意図の確認待ち） | 開発用の認証モード（`AUTH_MODE=dev`）。リクエストヘッダ `X-Dev-User` の `user_id` で本人になる。本番では無効。認証は `AuthProvider` の1か所に隔離し、最終フェーズで Cognito に差し替える | ENG-005 | backend, frontend, windows |
| N-08 | セッション30分の失効、CSRF対策（Cookie＋CSRFトークン）と、Cognito のトークンの扱い | Web は backend がログインを受け持ち、署名付きセッション Cookie（HttpOnly・Secure・SameSite=Lax、最終操作から30分で失効する更新型）を発行する。更新系は `X-CSRF-Token`。受付アプリは Cookie を使わず Bearer。詳細は [api.yml](../backend/api.yml) | ENG-002 | backend, frontend |
| N-09 | 欠課警告メール（希望）の送信手段。提出版は Microsoft Graph API | Amazon SES（AWS を使うため）。希望機能なので後回し | ENG-002 | backend |
| N-10 | 3つの CTX の名前（図では未記入） | 仮の呼び名「ユーザー」「授業」「出欠」をそのまま使う | ENG-006 | engineering |
| N-13 | 担当教員が複数いるときの権限（修正・受付・授業の編集）。提出版は担当教員1人 | 担当教員（ClassTeacherMembership にいる教員）は、すべて同じ権限 | ENG-006 | backend |
| N-14 | 担当教員の追加方法（画面がない。モックにもない） | 授業を登録した教員が担当になる。2人目以降の追加は、今は画面もAPIも用意しない | ENG-006 | backend, frontend |
| N-15 | リポジトリ構成とローカル開発の環境（設計書にない） | [repo-structure.md](repo-structure.md) の案（`apps/api`、`apps/web`、`apps/reception`。開発は DynamoDB Local） | なし | engineering |
| N-16 | Terraform の workflow の細部（S3 バケット名、IAM ロール、承認の要否、Terraform のバージョン） | 状態は S3（`use_lockfile`）。計画は `plans/{コミットSHA}/tfplan` に保存する。main への push で plan、承認（GitHub Environment `production`）、S3 の計画で apply。AWS へは OIDC で認証する。詳細は [infra.md](infra.md) と [workflow](../../.github/workflows/apply-to-prod.yml) | ENG-011 | engineering |

## 未決事項の状況（設計書11章）

基本設計書11章の未決事項。提出版の11章は書き換えず、決まったらここの状態を更新し、決定ログに記録する。決まるまでは設計書の「仮の決め」で実装する。

| # | 未決事項 | 仮の決め（設計書） | 状態 | 主に関わる領域 |
|---|---|---|---|---|
| 1 | 1回の授業を欠課何時間と数えるか | 90分＝2時間、授業ごとに変更可 | 未決 | backend |
| 2 | 早退を欠課時数にどう換算するか | 区分だけ用意し、換算しない | 未決 | backend |
| 3 | 最初の教員アカウントを誰が登録するか | 初期データとして投入 | 仮の決めあり（管理者が Cognito に作り、初期データの投入で Teacher を作る。N-04） | backend |
| 4 | 教員が受付を開始し忘れた授業回の扱い | 自動で欠課にせず「未実施」と表示 | 未決 | backend, frontend |
| 5 | 学生証から読めるIDの種類（FeliCa の IDm か別方式か） | RC-S300 の実機で確認する | 未決 | windows |
| 6 | 受付アプリを Windows デスクトップアプリにする案でよいか | Windowsデスクトップアプリ（代替案: ブラウザ＋WebUSB） | 未決 | windows |
| 7 | 教室PCがネットワークにつながらないときの受付の一時保存 | 扱わない。教員が後で修正 | 未決 | windows, backend |
| 8 | サーバの置き場所（学内かクラウドか）と運用担当 | なし | 一部決定（DynamoDB・Cognito を使うので AWS）。運用担当と、API・画面の置き場所は未決 | engineering |
| 9 | 欠課警告メールのしきい値 | 例: 上限の3分の2 | 未決（希望機能） | backend |
| 10 | R-05-3 の文意（学籍番号のみで登録できない場合と学生証再発行の関係） | なし。要求定義書側で整理する | 未決 | engineering |
