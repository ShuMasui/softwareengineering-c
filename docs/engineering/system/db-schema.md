# DBスキーマ

出典: [基本設計書](../basic-design.md) 7章・8章を元に、次の裁定を反映した（[決定ログ](../decisions.md)）。

- DBは Amazon DynamoDB（ENG-001）
- ドメインモデルは図のとおり（ENG-006。[domain-model.md](domain-model.md)）
- 詳細の項目は自由に決めてよいが、1エンティティにつき**キー以外5つ以内**（ENG-008）

**この文書の「DynamoDB の形」は確定**（[ENG-010](../decisions.md)）。提出版の項目は、上の方針で絞っている（「提出版との差」を参照）。

## 方針

- テーブルは1つ（単一テーブル設計）。名前は `attendance_app`。
- キーは `PK` `SK`。GSI は `GSI1`（`GSI1PK` `GSI1SK`）と `GSI2`（`GSI2PK` `GSI2SK`）の2つ。キーの項目はすべて文字列。
- キー以外の項目は、1エンティティ5つ以内。共通項目（作成者・作成日時・更新者・更新日時・削除フラグ）は持たない。削除は物理削除。
- 楽観的排他の `last_update_version` は、修正が競合する出欠（Attendance）だけが持つ。
- **欠課時数は保存しない。** backend が出欠から毎回計算する（下の「欠課時数の集計」）。
- ID: `user_id` は Cognito のユーザーID（sub）。学生・教員のIDは `user_id` と同じ値にする。授業・授業回は UUID v7（google/uuid の `NewV7`。時刻順に並ぶ）を backend で作る。
- 日時: ISO 8601 の文字列、日本時間（例 `2026-11-16T09:00:00+09:00`）。時刻だけの項目は `HH:mm`。
- 区分: 数値。値は [codes.md](../catalog/codes.md)。

## DynamoDB の形

`{...}` は値が入る場所。キー以外の項目は右から2列目、その数は右端。

| エンティティ（ドメインモデル） | PK | SK | GSI1（PK / SK） | GSI2（PK / SK） | キー以外の項目 | 数 |
|---|---|---|---|---|---|---|
| ユーザー（User） | `USER#{user_id}` | `#META` | — | — | email | 1 |
| 学生（Student） | `STUDENT#{user_id}` | `#META` | `ALL_STUDENT` / `{student_cd}` | — | student_cd, student_name | 2 |
| 教員（Teacher） | `TEACHER#{user_id}` | `#META` | — | — | teacher_name | 1 |
| 授業（Class） | `CLASS#{class_id}` | `#META` | `ALL_CLASS` / `{class_id}` | — | subject_name, room_name, session_hours, absence_limit_hours | 4 |
| 担当（ClassTeacherMembership） | `CLASS#{class_id}` | `TEACHER#{teacher_id}` | `TEACHER#{teacher_id}` / `CLASS#{class_id}` | — | なし | 0 |
| 受講（StudentClassMembership） | `CLASS#{class_id}` | `STUDENT#{student_id}` | `STUDENT#{student_id}` / `CLASS#{class_id}` | — | なし | 0 |
| 授業回（AttendanceSession） | `SESSION#{session_id}` | `#META` | `CLASS#{class_id}` / `SESSION#{start_datetime}#{session_id}` | `RECEIVING` / `{end_datetime}`（受付中のときだけ付ける） | class_id, start_datetime, end_datetime, session_div, reception_status | 5 |
| 出欠（Attendance） | `SESSION#{session_id}` | `STUDENT#{student_id}` | `STUDENT#{student_id}` / `ATT#{class_id}#{start_datetime}` | `CLASS#{class_id}` / `ATT#{student_id}#{start_datetime}` | class_id, attendance_status, touch_datetime, input_div, last_update_version | 5 |
| 学生証（図にない。ENG-007） | `STUDENT#{student_id}` | `CARD#{card_uid}` | — | — | valid_flg, invalid_datetime | 2 |
| 有効な学生証の検索・ロック | `CARD#{card_uid}` | `#VALID` | — | — | student_id | 1 |
| 学籍番号のロック | `STUDENTCD#{student_cd}` | `#LOCK` | — | — | student_id | 1 |
| 時限（図にない。ENG-007） | `PERIOD#{period_seq}` | `#META` | — | — | start_tm, end_tm | 2 |
| 出欠修正履歴（図にない。ENG-007） | `SESSION#{session_id}` | `HISTORY#{student_id}#{update_datetime}` | — | — | before_status, after_status, reason_remarks, teacher_id | 4 |
| タッチログ（図にない。ENG-007） | `SESSION#{session_id}` | `TOUCH#{touch_id}` | — | `CARD#{card_uid}` / `TOUCH#{touch_id}` | card_uid, result_div, touch_datetime | 3 |

- 授業回（AttendanceSession）と、その出欠・修正履歴・タッチログは、同じ `SESSION#{session_id}` のパーティションに入る（授業回の集約が出欠を持つ、というドメインモデルのとおり）。
- `touch_id` は UUID v7（時刻順に並ぶ）。
- 授業回の日時を変えたときは、その授業回の出欠の GSI キー（`start_datetime` を含む）も更新する。出欠は受付開始より後にできるので、通常は起きない。

## アクセスパターンと操作

| # | 場面 | 操作 |
|---|---|---|
| 1 | 申告: 学生証から学生を特定 | GetItem `CARD#{card_uid}` / `#VALID`（student_id が分かる） |
| 2 | 申告: 授業回の受付状態の確認 | GetItem `SESSION#{session_id}` / `#META`（class_id・reception_status） |
| 3 | 申告: 受講学生か確認 | GetItem `CLASS#{class_id}` / `STUDENT#{student_id}` |
| 4 | 申告: 登録（二重防止）とタッチログ | TransactWriteItems: 出欠を `attribute_not_exists(PK)` で Put ＋ タッチログを Put。条件に外れたら申告済み |
| 5 | 受付中の申告人数／受講者数 | Query（COUNT）`SESSION#…` で SK が `STUDENT#` 始まり／`CLASS#…` で SK が `STUDENT#` 始まり |
| 6 | 授業回別出欠一覧 | Query `SESSION#…` で SK が `STUDENT#` 始まり。氏名は BatchGet |
| 7 | 学生別欠課時数一覧 | Query GSI2 `CLASS#{class_id}` で SK が `ATT#` 始まり |
| 8 | 学生ホーム・授業別出欠記録 | Query GSI1 `STUDENT#{student_id}` で SK が `CLASS#` 始まり（受講授業）／`ATT#` 始まり（出欠） |
| 9 | 教員ホーム・今日の授業回 | Query GSI1 `TEACHER#{teacher_id}` で SK が `CLASS#` 始まり（担当授業）。授業ごとに Query GSI1 `CLASS#{class_id}` で SK が今日の範囲 |
| 10 | 受付自動締切（1分ごと） | Query GSI2 `RECEIVING` で SK が現在時刻以前。締切では GSI2 のキーを消す |
| 11 | サインイン後の本人の特定 | GetItem `USER#{user_id}`、`STUDENT#{user_id}` または `TEACHER#{user_id}` |
| 12 | 修正履歴・タッチログの参照 | Query `SESSION#…` で SK が `HISTORY#{student_id}` 始まり／`TOUCH#` 始まり。カード単位は GSI2 `CARD#{card_uid}` |
| 13 | 欠課警告メール（希望） | Query GSI1 `ALL_CLASS` で授業を取り、授業ごとに #7 と同じ取得 |
| 14 | 担当授業かの確認（全API） | GetItem `CLASS#{class_id}` / `TEACHER#{teacher_id}` |
| 15 | 学生一覧・学籍番号で探す | Query GSI1 `ALL_STUDENT`（SK＝学籍番号）。学生証の状態は Query `STUDENT#{id}` で SK が `CARD#` 始まり |

## 制約の守り方

| 制約 | 守り方 |
|---|---|
| 授業回×学生で出欠1件 | 上の #4。キーが (授業回, 学生) なので、条件付き Put で二重にならない |
| 有効な card_uid は一意 | 学生証の登録: TransactWriteItems で `CARD#{card_uid}` / `#VALID`（`attribute_not_exists`）と学生証の項目を作る。重複は `A_S0005_E000001`。無効化: `#VALID` を消し、学生証の valid_flg=0・invalid_datetime を書く |
| 学籍番号は一意 | 学生登録: TransactWriteItems で `STUDENTCD#{student_cd}` / `#LOCK`、学生、ユーザーを作る |
| 受講・担当は組で一意 | キーが (授業, 学生／教員) なので、条件付き Put で守る |
| 出欠の楽観的排他 | 出欠修正: `last_update_version = :v` を条件にした更新。外れたら `A_S0004_E000002`。修正履歴は同じトランザクションで書く |
| 受付トークン | **保存しない。** 受付開始時に、授業回IDと期限を入れた署名付きトークンを発行し、検証だけで確かめる（項目を増やさないため）。受付中でなくなれば使えない |
| 休講の回は出欠を記録しない | backend が `session_div=9` の授業回への申告を断る |
| 外部キー相当の整合性 | backend が確かめる（DynamoDB は守らない） |

## 欠課時数の集計（提出版の absence_summary_v）

出典: 設計書 6.1。授業 × 学生ごとの欠課時数を、出欠から**毎回計算する**（保存しない）。取得は上の #7（教員）と #8（学生）。計算式は提出版のまま。

| 項目 | 内容 |
|---|---|
| absent_count | 出欠区分が欠課（40）の件数 |
| late_count | 出欠区分が遅刻（20）の件数 |
| absence_hours | `absent_count × session_hours ＋ (late_count を3で割った商)` |
| remaining_hours | `absence_limit_hours － absence_hours`（上限までの残り） |

- 公欠（50）は数えない。早退（30）も数えない（換算は未決。[decisions.md](../decisions.md) の #2）。
- 休講の授業回は出欠がないので、集計に入らない。
- 出欠が1件もない受講学生にも欠課0時間で出す。受講（StudentClassMembership）を起点にする。

## 提出版との差

| 提出版 | この案 |
|---|---|
| 10テーブル＋1ビュー | 単一テーブル。ビューはなく、backend が計算する |
| 共通項目（作成者・作成日時・更新者・更新日時・削除フラグ）を全テーブルに | 持たない。更新バージョンは出欠だけ。削除は物理削除 |
| `course_mst` に曜日・時限・期間・担当教員 | 授業は科目名・教室・1回あたりの時間数・欠課上限だけ。曜日・時限・期間は授業回を作るときの入力で、保存しない。担当教員は担当（ClassTeacherMembership） |
| `student_mst` / `teacher_mst` に email | email はユーザー（User）にだけある（ログインID） |
| `attendance_id`・`touch_log_id` などの連番ID | キーに吸収した。出欠は (授業回, 学生) で特定する |
| 出欠修正履歴の `attendance_history_id`・`attendance_id` | キー（授業回＋学生＋更新日時）に吸収した |
| 受付トークン（保存する想定） | 署名付きトークンで、保存しない |

## 実装で決める事項

- 学生・教員の登録で、Cognito のユーザー作成とDBへの書き込みの順序と、片方が失敗したときの戻し方（[ENG-009](../decisions.md)）。
- 時限マスタの初期データ（各時限の開始・終了時刻）。
- 授業回を作る処理（曜日・時限・期間から日付ごとに作る）の詳細。
- 外部キー相当の整合性（例: 授業を削除したときの授業回・出欠・受講・担当の扱い）。
