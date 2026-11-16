# Web API（案）

設計書には API 一覧がない。機能（4章）・画面（5章）・処理（6章）・DB の操作（[db-schema.md](../engineering/system/db-schema.md) のアクセスパターン `#`）から起こした。**仮の決め**（[N-04〜N-09](../engineering/decisions.md)）を含み、ユーザーの確認はまだ。確認前でも、この案に従って実装してよい。

## 共通

| 項目 | 内容 |
|---|---|
| ベースURL | `/api/v1`（認証だけ `/auth`） |
| 形式 | JSON（UTF-8）。日時は ISO 8601・日本時間（`2026-11-16T09:00:00+09:00`） |
| 認証 | Web: セッション Cookie。更新系（POST・PUT）には `X-CSRF-Token` も付ける。受付アプリ: `Authorization: Bearer <アクセストークン>`（Cookie を使わないので CSRF は対象外）。開発中は `X-Dev-User: <user_id>`（N-07） |
| ロール | トークンを発行したユーザープール（学生用／教員用）で決まる。本人のIDはサーバがトークンから決め、リクエストのIDは信用しない |
| 認可 | 学生は自分のデータだけ。教員は、担当授業（ClassTeacherMembership）のデータだけ（DB #14）。外れたら 403 |
| エラー | `{ "messageId": "...", "message": "..." }`（文言は [メッセージ台帳](../engineering/catalog/messages.md)） |

### HTTP ステータスとメッセージ

| 状況 | HTTP | messageId |
|---|---|---|
| 未サインイン・セッション失効 | 401 | なし |
| 権限なし・担当外 | 403 | `SYSCOMMON_E000101` |
| 入力の誤り（修正理由なしなど） | 422 | 機能ごと（例 `A_S0004_E000001`） |
| 競合（楽観的排他、学生証の重複） | 409 | `A_S0004_E000002`、`A_S0005_E000001` |
| 書き出す記録がない | 404 | `A_F0002_W000001` |

**申告（`POST /reception/touches`）の結果は、成功・申告済み・エラーのどれも HTTP 200 で、`resultDiv`（タッチ結果区分）に入れて返す。** HTTP エラーは、通信・認証・不正なリクエストのときだけ。

## 認証（A_A0001）

| メソッド | パス | 内容 | 誰が |
|---|---|---|---|
| GET | `/auth/login?pool=student` または `pool=teacher` | 該当ユーザープールの Hosted UI へ移す | 誰でも |
| GET | `/auth/callback` | Hosted UI から戻る。認可コードをトークンに換え、検証して、セッション Cookie を発行する | 誰でも |
| POST | `/auth/logout` | セッションを捨て、Hosted UI のサインアウトへ移す | サインイン済み |
| GET | `/api/v1/me` | 本人の名前とロール（DB #11） | サインイン済み |

## 学生（A_S0002）

| メソッド | パス | 内容 | DB |
|---|---|---|---|
| GET | `/api/v1/student/classes` | 受講授業と、授業ごとの欠課時数・上限・残り（A_S0002_SL0001） | #8 |
| GET | `/api/v1/student/classes/{classId}/attendances` | 授業の日付ごとの出欠（A_S0002_SL0002） | #8 |

## 教員の確認（A_S0003）

| メソッド | パス | 内容 | DB |
|---|---|---|---|
| GET | `/api/v1/teacher/classes` | 担当授業（A_S0003_SL0001） | #9 |
| GET | `/api/v1/teacher/sessions/today` | 今日の授業回（状態・申告人数。「未実施」の表示用） | #9, #5 |
| GET | `/api/v1/sessions/{sessionId}/attendances` | 授業回の受講学生全員の出欠と申告時刻（A_S0003_SL0002） | #6 |
| GET | `/api/v1/classes/{classId}/absence-summary` | 学生別の欠課・遅刻・欠課時数・残り（A_S0003_SL0003） | #7 |

## 出欠修正（A_S0004）

| メソッド | パス | 内容 | DB |
|---|---|---|---|
| PUT | `/api/v1/sessions/{sessionId}/attendances/{studentId}` | 出欠区分、理由（必須）、`version` を送る。出欠がまだない（未申告）なら新しく作る。修正履歴も同じトランザクションで書く。理由なしは 422、`version` が違えば 409 | トランザクション |

## 学生・学生証（A_S0005）

| メソッド | パス | 内容 | DB |
|---|---|---|---|
| GET | `/api/v1/students` | 学生一覧と学生証の状態（A_S0005_SL0001）。`?card=none` で学生証が未登録の学生だけ（受付アプリの登録モード用） | #15 |
| POST | `/api/v1/students` | 学生登録（学籍番号・氏名・メール）。Cognito に招待を出す（下の「ユーザーの作り方」） | トランザクション |
| POST | `/api/v1/students/{studentId}/invitation` | 招待メールの再送 | なし |
| POST | `/api/v1/students/{studentId}/cards` | 学生証の登録（本文にカードID）。受付アプリの登録モードから呼ぶ。別の学生に登録済みなら 409 | トランザクション |
| POST | `/api/v1/students/{studentId}/cards/invalidate` | 有効な学生証を無効にする（カードIDをURLに出さない） | トランザクション |

## 授業（A_S0006）と名簿（A_F0001）

| メソッド | パス | 内容 | DB |
|---|---|---|---|
| GET | `/api/v1/periods` | 時限（開始・終了時刻） | GetItem |
| POST | `/api/v1/classes` | 授業登録。科目名・教室・1回あたりの時間数・欠課上限 ＋ 授業回の生成条件（曜日・時限・期間。保存しない）。登録した教員が担当になる（N-14） | トランザクション |
| GET | `/api/v1/classes/{classId}` | 授業の取得 | GetItem |
| PUT | `/api/v1/classes/{classId}` | 授業の編集 | UpdateItem |
| GET | `/api/v1/classes/{classId}/sessions` | 授業回の一覧 | #9 |
| POST | `/api/v1/classes/{classId}/sessions` | 補講の授業回を追加 | PutItem |
| PUT | `/api/v1/sessions/{sessionId}` | 授業回の編集（区分・日時。休講・補講・時間変更） | UpdateItem |
| POST | `/api/v1/classes/{classId}/enrollments/import?mode=preview` | 名簿ファイル（xlsx または CSV）の取り込み結果の確認。保存しない。誤りは行ごとに 422（`A_F0001_E000001`） | なし |
| POST | `/api/v1/classes/{classId}/enrollments/import?mode=commit` | 保存。誤りが1行でもあれば保存しない。未登録の学籍番号は学生として新規登録し、招待を出す | トランザクション |
| GET | `/api/v1/classes/{classId}/export` | 出欠記録の書き出し（xlsx）。希望機能。記録がなければ 404 | #7 |

## 受付（A_S0001、A_S0005_SD0002）

受付アプリが呼ぶ。教員のアクセストークンで認証する。学生は、受付アプリに学生証をかざすだけで、APIを直接は呼ばない。

| メソッド | パス | 内容 | DB |
|---|---|---|---|
| GET | `/api/v1/reception/sessions` | 今の時刻に合う担当授業回（授業開始の30分前から）。`serverTime` も返す | #9 |
| POST | `/api/v1/reception/sessions/{sessionId}/start` | 受付開始。署名付きの受付トークンを返す | UpdateItem |
| POST | `/api/v1/reception/sessions/{sessionId}/stop` | 受付終了。締切処理（記録のない受講学生を欠課（自動）で登録） | トランザクション |
| POST | `/api/v1/reception/touches` | 申告 | #1〜#4 |

### 申告（`POST /api/v1/reception/touches`）

- ヘッダ: `Authorization: Bearer ...`、`X-Reception-Token: ...`
- 本文: `{ "sessionId": "...", "cardUid": "..." }`
- 処理の順序と、エラーごとのタッチ結果区分は [domain-model.md](../engineering/system/domain-model.md) の「申告」のとおり。すべてのタッチ（エラー含む）をタッチログに残す。
- 応答（HTTP 200）:

```json
{
  "resultDiv": 1,
  "messageId": "A_S0001_I000001",
  "message": "青木 一郎さん　出席",
  "attendanceStatus": 10,
  "touchedCount": 32,
  "enrolledCount": 40,
  "serverTime": "2026-11-16T08:55:03+09:00"
}
```

`resultDiv` は 1 受付、2 申告済み、7 受講対象外、8 未登録、9 締切後（[codes.md](../engineering/catalog/codes.md)）。エラーのときは `attendanceStatus` を返さない。

## バッチ・メール（APIではない）

| ID | 内容 | 実行 | 備考 |
|---|---|---|---|
| A_B0001 | 受付自動締切。受付中で終了時刻を過ぎた授業回を締め切る（`stop` と同じ締切処理）。DB #10 | 1分ごと | 実行基盤は未決（設計書11章 #8） |
| A_M0001 | 欠課警告メール（希望）。Amazon SES で送る（N-09）。DB #13 | 毎日18:00 | 希望機能。後回し |

## ユーザーの作り方（仮の決め。N-04）

Cognito の標準フロー（ENG-009）に従う。

1. 教員が学生を登録する（`POST /students`）、または名簿取込で新規の学生が見つかる。
2. backend が、先に Cognito（学生用プール）にユーザーを作る。ユーザー名は Microsoft のメールアドレス。Cognito が、仮パスワード付きの招待メールを送る。
3. 続けて、DynamoDB に User と Student（学籍番号のロックを含む）を書く。**DBの書き込みに失敗したら、Cognito のユーザーを消す。**
4. 本人が、招待メールの仮パスワードで Hosted UI にサインインし、新しいパスワードを決めて、本登録になる。
5. 招待の再送は `POST /students/{studentId}/invitation`（Cognito の再送機能）。パスワードを忘れたときは、Hosted UI の標準機能。
6. 最初の教員は、管理者が Cognito（教員用プール）に作り、初期データの投入で User と Teacher を作る（設計書11章の未決 #3）。

## セッションと CSRF（仮の決め。N-08）

- Web は、backend が Hosted UI のログイン（認可コード）を受け持つ。トークンを検証して、署名付きのセッション Cookie（HttpOnly・Secure・SameSite=Lax）を発行する。
- Cookie は、最終操作から30分で失効する更新型（操作のたびに期限を延ばす）。
- 更新系は、CSRF トークンを `X-CSRF-Token` ヘッダで送る（Cookie に入れた値との二重送信）。
- 受付アプリは、Cookie を使わず `Authorization: Bearer` で呼ぶ。受付中の授業が終わるまで、アクセストークンを更新して維持する。
