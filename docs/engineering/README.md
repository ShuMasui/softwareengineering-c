# engineering

全領域（backend / frontend / windows）で共有する事項をまとめる。

| パス | 内容 |
|---|---|
| [basic-design.md](basic-design.md) | 提出版の基本設計書。固定で、書き換えない |
| [decisions.md](decisions.md) | 共有の決定事項のログと、設計書11章の未決事項の状況 |
| [system/db-schema.md](system/db-schema.md) | テーブル・ビューの定義、制約、共通項目 |
| [system/domain-model.md](system/domain-model.md) | 用語、エンティティと関連、判定ルール、状態 |
| [catalog/](catalog/README.md) | 画面に出す文言とコード値の台帳 |

## 運用

- 2つ以上の領域に関わる決定は、ここ（`decisions.md`）に書く。1つの領域だけなら、その領域の `decisions.md` に書く。
- 提出版と食い違う変更は、ユーザーの確認を得てから反映する。
