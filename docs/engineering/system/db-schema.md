# DBスキーマ

出典: [基本設計書](../basic-design.md) 7章・8章。次の2つの裁定を反映している（[決定ログ](../decisions.md)）。

- DBを MySQL から **Amazon DynamoDB** に変える（ENG-001）
- ドメインモデルは、ユーザーが示した図を使う（ENG-006。[domain-model.md](domain-model.md)）

この文書は**論理スキーマ**（エンティティ・項目・制約・ルール）を定める。DynamoDB のテーブル構成・キー・GSI（物理設計）は未決（N-01）。提出版に書かれていない項目は「**未定**」と書いてある。決めたら、この文書を更新し、`../decisions.md` に記録する。

## 方針

- エンティティの名前は、提出版の物理名（`teacher_mst` など）を引き続き使う。DynamoDB 上のテーブル名・構成は未決。提出版にないエンティティ（User、ClassTeacherMembership）は、ドメインモデルでの名前で書き、物理名は未定。
- **欠課時数は保存しない。** 出欠から backend が毎回計算する（下の「欠課時数の集計」）。
- 排他制御は楽観的排他。`last_update_version` を条件に付けた更新で行う。
- 文字コードは UTF-8。

## 提出版（MySQL）からの読み替え

| 提出版 | DynamoDB では |
|---|---|
| 外部キー（FK） | DBは守らない。参照先が存在するかは backend が確かめる |
| 一意制約（UK） | キーの設計か、条件付き書き込み・トランザクションで守る |
| ビュー `absence_summary_v` | ない。backend が計算する（N-02） |
| 自動採番の bigint の ID | ない。IDは backend が作る（方法は未決。N-03） |
| データ型（bigint・integer・varchar・datetime） | 数値（N）と文字列（S）。日時の持ち方は未定 |
| インデックス | テーブルのキーと GSI（グローバルセカンダリインデックス）。未決 |
| SQL・プリペアドステートメント | SQLを使わない。PartiQL を使う場合はパラメータ付きにする |

## 共通項目

全エンティティが持つ（設計書では ED19 テンプレートの共通項目）。

| 論理名 | 物理名 | 備考 |
|---|---|---|
| 作成者 | 未定 | |
| 作成日時 | 未定 | |
| 更新者 | 未定 | |
| 更新日時 | 未定 | |
| 更新バージョン | `last_update_version` | 楽観的排他制御用。初期値0（`attendance_trn` の定義より） |
| 削除フラグ | 未定 | |

## エンティティ一覧

「ドメインモデル」の列は [domain-model.md](domain-model.md) の名前。**No.12・13 は提出版にない**（ENG-006）。

| No. | 物理名（提出版） | 論理名 | ドメインモデル | エンティティID | 主な項目 | 説明 |
|---|---|---|---|---|---|---|
| 1 | teacher_mst | 教員 | Teacher | E000001 | teacher_id, teacher_name, email, User への参照 | User を参照する。email を User に移すかは未決（N-11） |
| 2 | student_mst | 学生 | Student | E000002 | student_id, student_cd（学籍番号）, student_name, email, User への参照 | 学籍番号は一意。User を参照する。email を User に移すかは未決（N-11） |
| 3 | student_card_mst | 学生証 | 図にない（ENG-007） | E000003 | student_card_id, card_uid, student_id, valid_flg, invalid_datetime | 再発行時は古い学生証を valid_flg=0 にする。有効な card_uid は一意（R-05-2, R-05-3） |
| 4 | period_mst | 時限 | 図にない（ENG-007） | E000004 | period_seq, start_tm, end_tm | 各時限の開始・終了時刻 |
| 5 | course_mst | 授業 | Class | E000005 | course_id, subject_name, day_div, period_seq, room_name, start_dt, end_dt, session_hours, absence_limit_hours | session_hours＝1回あたりの時間数、absence_limit_hours＝欠課上限。**提出版の teacher_id は持たない**（担当教員は No.13） |
| 6 | enrollment_mst | 受講 | StudentClassMembership | E000006 | course_id, student_id | 授業と学生の組。組で一意 |
| 7 | class_session_trn | 授業回 | AttendanceSession | E000007 | class_session_id, course_id, session_dt, start_tm, end_tm, session_div, reception_status | 授業登録時に期間と曜日から自動生成し、休講・補講で編集する。授業回の項目と受付状態の持ち方は未決（N-12） |
| 8 | attendance_trn | 出欠 | Attendance | E000008 | attendance_id, class_session_id, student_id, attendance_status, touch_datetime, input_div | 授業回×学生で1件。授業回（AttendanceSession）が持ち、受講（StudentClassMembership）を参照する |
| 9 | attendance_history_trn | 出欠修正履歴 | 図にない（ENG-007） | E000009 | attendance_history_id, attendance_id, before_status, after_status, reason_remarks, teacher_id, update_datetime | いつ・誰が・何を直したか（R-04-2） |
| 10 | card_touch_log_trn | タッチログ | 図にない（ENG-007） | E000010 | touch_log_id, class_session_id, card_uid, result_div, touch_datetime | エラーも含めて全件記録 |
| 11 | absence_summary_v | 欠課時数集計 | 図にない（計算で求める） | E000011 | course_id, student_id, absent_count, late_count, absence_hours, remaining_hours | **DBには作らない。** backend が計算する（下の「欠課時数の集計」） |
| 12 | User（物理名・エンティティIDは未定） | ユーザー | User | 未定 | 項目は未決（N-11）。ログインID（Microsoft のメールアドレス）を持つ想定 | サインインするユーザー。Student / Teacher から参照される（ENG-003, ENG-006） |
| 13 | ClassTeacherMembership（物理名・エンティティIDは未定） | 担当 | ClassTeacherMembership | 未定 | course_id, teacher_id | 授業と教員の組。1つの授業に複数の教員がいてよい（ENG-006）。提出版の course_mst.teacher_id の代わり |

「主な項目」は設計書にある項目の名前だけで、**`attendance_trn` 以外は型・NULL制約が未定**。

## attendance_trn（出欠）

データ型は提出版（RDB）のもの。DynamoDB では数値（N）・文字列（S）に読み替える。

| No. | 論理名 | 物理名 | キー | 提出版のデータ型 | NULL制約 | 備考 |
|---|---|---|---|---|---|---|
| 1 | 出欠ID | attendance_id | PK | bigint | NOT NULL | |
| 2 | 授業回ID | class_session_id | FK, UK | bigint | NOT NULL | class_session_trn 参照。この授業回（AttendanceSession）が出欠を持つ |
| 3 | 学生ID | student_id | FK, UK | bigint | NOT NULL | student_mst 参照。その授業の受講（enrollment_mst）がある学生に限る（ENG-006） |
| 4 | 出欠区分 | attendance_status | | integer | NOT NULL | [codes.md](../catalog/codes.md) の出欠区分 |
| 5 | 申告日時 | touch_datetime | | datetime | NULL可 | 学生証で申告したサーバ時刻。教員入力・自動は NULL |
| 6 | 入力区分 | input_div | | varchar(1) | NOT NULL | 1:カード 2:教員修正 3:自動 |
| 7 | 更新バージョン | last_update_version | | integer | NOT NULL | 楽観的排他制御用。初期値0 |

提出版のインデックスは、主キー（attendance_id）と一意キー（class_session_id, student_id）の2つ。共通項目は省略している。

## 制約（設計書に書かれているもの）

DynamoDB には一意制約がないので、「守り方」の列は候補で、物理設計（N-01）で決める。

| エンティティ | 制約 | 出典 | 守り方（候補） |
|---|---|---|---|
| student_mst | student_cd（学籍番号）は一意 | 7.1 | 条件付き書き込み＋トランザクション |
| student_card_mst | 有効な card_uid は一意 | 7.1（R-05-2, R-05-3） | 条件付き書き込み＋トランザクション |
| enrollment_mst | (course_id, student_id) は一意 | 7.1 | キーを (授業, 学生) にする |
| attendance_trn | (class_session_id, student_id) は一意。同時に2回かざしても二重にならない | 6.2（R-01-3）, 7.3 | キーを (授業回, 学生) にして、`attribute_not_exists` の条件付き書き込みで登録する |

## 欠課時数の集計（提出版の absence_summary_v）

出典: 設計書 6.1・7.1。授業 × 学生ごとの欠課時数を、出欠から**毎回計算する**（保存しない）。DynamoDB にビューはないので、backend が計算する。計算式は提出版のまま。

| 項目 | 内容 |
|---|---|
| course_id, student_id | 授業と学生 |
| absent_count | 出欠区分が欠課（40）の件数 |
| late_count | 出欠区分が遅刻（20）の件数 |
| absence_hours | `absent_count × course_mst.session_hours ＋ (late_count を3で割った商)` |
| remaining_hours | `course_mst.absence_limit_hours － absence_hours`（上限までの残り） |

- 公欠（50）は数えない。早退（30）も数えない（換算は未決。[decisions.md](../decisions.md) の #2）。
- 休講の授業回（session_div=9）は出欠を記録しないので、集計に入らない。
- 出欠が1件もない受講学生にも1行出す（欠課0時間）ため、`enrollment_mst`（受講）を起点に集計する（設計書に明記はない。学生ホームで「上限までの残り」を出すために必要）。

## アクセスパターン

物理設計（キー・GSI）を決めるための材料。設計書の処理から、DBに必要な検索を挙げた。

| # | 場面 | 何で探すか | 出典 |
|---|---|---|---|
| 1 | 申告: 学生証から学生を特定 | card_uid（有効なもの） | 6.2 手順3 |
| 2 | 申告: 授業回の受付状態の確認 | class_session_id | 6.2 手順2 |
| 3 | 申告: 受講学生か確認 | course_id ＋ student_id | 6.2 手順4 |
| 4 | 申告: 申告済みか確認・登録 | class_session_id ＋ student_id（一意） | 6.2 手順5・6 |
| 5 | 受付中の申告人数／受講者数 | 授業回の出欠件数、授業の受講学生数 | 6.4 |
| 6 | 授業回別出欠一覧 | class_session_id の全出欠 | A_S0003_SL0002 |
| 7 | 学生別欠課時数一覧 | course_id の全授業回の全出欠 | A_S0003_SL0003 |
| 8 | 学生ホーム・授業別出欠記録 | student_id の受講授業。student_id ＋ course_id の出欠 | A_S0002_SL0001, SL0002 |
| 9 | 教員ホーム | 担当（ClassTeacherMembership）から teacher_id の担当授業と、今日の授業回 | A_S0003_SL0001 |
| 10 | 受付自動締切（1分ごと） | 受付中（reception_status=1）で終了時刻を過ぎた授業回 | A_B0001 |
| 11 | サインイン後の本人の特定 | ログインID（メールアドレス）から User、その学生／教員のレコード | ENG-003, ENG-006 |
| 12 | 出欠修正履歴・タッチログの参照 | attendance_id、class_session_id、card_uid | 7.1 |
| 13 | 欠課警告メール（希望） | 欠課時数がしきい値以上の学生（全授業） | A_M0001 |
| 14 | 担当授業かの確認（全API） | course_id ＋ teacher_id | 10.2 認可 |

## 実装で決める事項（提出版に記載なし）

実装に入る前に決めて、`../decisions.md` に記録する。

- テーブル構成（エンティティごとに分けるか、まとめるか）、キー、GSI。上のアクセスパターンを満たすこと（N-01）。
- IDの採番方法（N-03）。
- User の項目、email の置き場所、Student / Teacher からの User への参照の持ち方（N-11）。
- 授業回（AttendanceSession）の項目と受付状態の持ち方（N-12）。
- 担当（ClassTeacherMembership）の一意制約。受講と同じく (授業, 教員) の組で一意にする想定。
- 共通項目の物理名、論理削除（削除フラグ）の扱い。削除済みのデータを一意制約や集計から外す方法。
- `attendance_trn` 以外の項目の型。日時・時刻の文字列の形式。
- 「有効な card_uid は一意」の守り方（トランザクションでの更新など）。
- メールアドレス（User のログインID）の一意性。ユーザープールが学生用と教員用に分かれているので、プールごとに一意であればよいが、同じアドレスを両方に作ってよいかを決める。
- 外部キー相当の整合性（例: 授業を削除したときの授業回・出欠・受講・担当の扱い）を、backend のどこで守るか。
