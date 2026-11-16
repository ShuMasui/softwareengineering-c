# インフラ（Terraform）

インフラ（AWS）は **Terraform** で実装する（[ENG-011](decisions.md)）。**state（状態）は S3 に置き**、`terraform apply` は GitHub Actions の workflow で、**S3 に保存した計画（plan）を使って**行う。

- Terraform のコードは `infra/` に置く（ユーザーが書く。このリポジトリには、まだない）。
- workflow は [.github/workflows/apply-to-prod.yml](../../.github/workflows/apply-to-prod.yml)。
- state を S3 に置くことと、S3 に保存した計画で apply することは、ユーザーの確認済み。バケット名・IAM ロールなどの細部は [N-16](decisions.md) の仮の決め。

## apply までの流れ

| 順 | きっかけ | すること |
|---|---|---|
| 1 | pull_request（`infra/` の変更） | fmt・validate・plan。結果は job summary に出す。apply はしない |
| 2 | main への push | plan を作り、計画ファイルを `s3://{計画のバケット}/plans/{コミットSHA}/tfplan` に保存する |
| 3 | 承認 | GitHub の Environment「production」の承認者が、job summary の計画を見て承認する |
| 4 | 承認のあと | S3 から計画ファイルを取り出し、`terraform apply tfplan` をそのまま実行する（plan をやり直さない） |
| 5 | 手動（`workflow_dispatch`） | `plan_sha` を指定すると、S3 に保存済みのその計画で apply する。空なら、いまの main で plan から始める |

- apply は main でしか動かない。同じブランチの実行は、1つずつ順に動く（apply が重ならない）。
- 計画ファイルは、作ったときと同じコミットのコードで apply する（workflow が、そのコミットを checkout する）。状態が変わっていれば、Terraform が「計画が古い」と止める。

## 用意するもの

### GitHub

| 種類 | 名前 | 内容 |
|---|---|---|
| Variables | `TF_STATE_BUCKET` | Terraform の状態（state）を置く S3 バケット |
| Variables | `TF_PLAN_BUCKET` | 計画ファイルを置く S3 バケット |
| Variables | `TF_PLAN_ROLE_ARN` | plan 用の IAM ロール（OIDC で引き受ける） |
| Variables | `TF_APPLY_ROLE_ARN` | apply 用の IAM ロール（OIDC で引き受ける） |
| Environment | `production` | 承認者（required reviewers）を設定する。設定しないと、承認なしで apply まで進む |

### AWS

- 状態用の S3 バケット: バージョニングと暗号化を有効にする。ロックは S3 の機能（`use_lockfile`）を使うので、ロック用のテーブルは要らない。
- 計画用の S3 バケット: 暗号化し、公開を禁止し、ライフサイクルで古い計画を消す（例: 30日）。**計画ファイルには、秘密の値が入ることがある**ので、アクセスを絞る。
- GitHub の OIDC プロバイダと、2つの IAM ロール。
  - plan 用: 状態の読み書き（ロックを含む）、計画バケットへの書き込み、管理対象リソースの参照。
  - apply 用: 管理対象リソースの作成・変更、状態の読み書き、計画バケットからの読み取り。信頼ポリシーで、このリポジトリの main（Environment `production`）だけが引き受けられるようにする。

### Terraform のコード（`infra/`）

- `terraform { required_version = ">= 1.10" ... backend "s3" {} }` とし、バケット・キー・リージョン・ロックは書かない（workflow が `terraform init` で渡す）。
- `.terraform.lock.hcl` をコミットする（plan と apply でプロバイダを揃えるため）。
- 状態のキーは `attendance/terraform.tfstate`。

## 確認の範囲

workflow は、実際には動かしていない（YAML として読めること、ジョブのつながりまでの確認）。バケットや IAM ロールを用意したあとで、PR で plan が動くことから確かめる。
