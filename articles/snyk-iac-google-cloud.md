---
title: "Snyk を使って Google Cloud の IaC 環境をセキュアに管理してみる"
emoji: "🐺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["snyk", "googlecloud", "gcp", "iac", "terraform"]
published: true
---

本記事は `Snyk を使ってコードをセキュアにした記事を投稿しよう！ by Snyk Advent Calendar 2023` 9 日目の記事です。
@[card](https://qiita.com/advent-calendar/2023/snyk)

こんにちは、アプリケーション開発を軸とした SRE 領域に興味のある cobo です。
今回は、Google Cloud の IaC 環境をセキュアに保つための Snyk の使い方を紹介します。

:::message
本記事では、Google Cloud (GCP) 以外のクラウドプロバイダー (AWS, Azure 等) は扱いません。
したがって、Google Cloud に依存した Snyk 設計になる可能性がある旨ご了承ください。
:::

## はじめに

Google Cloud のプロジェクトを Terraform を用いて IaC 化することはよくある話だと思います。
Snyk の話に入る前に、一旦 Terraform のリソース状態の管理方法について整理してみましょう。

### Terraform リソースの状態管理

- `.tf` ファイルでリソースをコード化
- `.tfstate` ファイルでリソースの状態を保持
- `terraform plan` or `terraform apply` 実行時のリソース差分検出で、`.tf` ファイルと `.tfstate` ファイル を比較している

![tf-state-image](/images/snyk-iac-google-cloud/tf_state_image.png)
_リソース状態管理のイメージ_

### 現状の課題

上記のような構成だと、

**実際の Google Cloud プロジェクト環境に対して、意図しないリソース追加や変更が発生した場合に検知が難しい**

といった課題が挙げられます。
例）IaC 化がルールのプロジェクトにおいて、リソースを手で追加してしまう、悪意を持った人が大量のリソースを追加、変更、削除してしまう 等

このような課題を解決するために、**Snyk IaC** を活用していきます。

## やりたいこと

Snyk IaC によって .tfstate ファイル と 実際の Google Cloud プロジェクト環境を比較して、IaC 管理内／管理外のリソースチェックを行う

## Snyk IaC

Snyk IaC は、Google Cloud, AWS, Azure などのクラウドプロバイダーで使用される Terraform リソースのセキュリティと整合性を管理するためのツールです。

これによって、.tfstate ファイルと実際のクラウドプロジェクト環境を比較することができるので、IaC 管理内外のリソースをチェックして、上記課題を解決することができます。

### ローカル実行で挙動を確認

まずは、ローカルから Snyk IaC を実行してどのような動き方をするのか確認していきましょう。

#### 1. Snyk のセットアップ

Snyk のアカウント作成や、Snyk CLI のインストールは下記の記事をおすすめします。
@[card](https://qiita.com/SnykSec/items/aa80a976eb17299fff62)

#### 2. Google Cloud のユーザー認証

下記コマンドを実行して認証を行います。

```plain
gcloud auth application-default login
```

#### 3. Snyk IaC の実行

下記コマンドを実行して Snyk IaC による差分チェックを行います。

```plain
CLOUDSDK_CORE_PROJECT=your-project \
snyk iac describe --only-unmanaged --to="gcp+tf" --from="tfstate+gs://bucket-name/default.tfstate" --org=your-org-name
```

:::message
今回は、IaC 管理対象外を出力するオプション `--only-unmanaged` を指定しました。
他にも IaC 管理対象内を出力するオプション `--only-managed` や、IaC 管理対象内外両方を出力するオプション `--all` があります。
:::

対象 Google Cloud プロジェクトに対する IaC 管理外のチェック結果は下記の通りとなりました。

```plain
Scanned states (1)
Scan duration: 6s
Provider version used to scan: 4.56.0. Use --tf-provider-version to use another version.
Snyk Scanning Infrastructure As Code Discrepancies...

  Info:    Resources under IaC, but different to terraform states.
  Resolve: Reapply IaC resources or update into terraform.

Unmanaged resources: 2

Service: google_cloud_platform [ Unmanaged Resources: 2 ]

  Resource Type: google_project_iam_member
    ID: snyk-iac-sample/roles/owner/user:foo@gmail.com
    ID: snyk-iac-sample/roles/owner/user:bar@gmail.jp

Test Summary

  Managed Resources: 3
  Unmanaged Resources: 2

  IaC Coverage: 60%
  Info: To reach full coverage, remove resources or move it to Terraform.
```

Unmanaged resources として出力された 2 件の IAM は確かに .tf ファイルには記載していなく、実際のコンソールから手で作成したものでした。

また、IaC カバレッジが表示されるのもいいですね。

:::details Managed Resources: 3 に該当する IaC 管理中のコード

```hcl
# クラウドアセット閲覧者
resource "google_project_iam_member" "snyk_scan_cloudasset_reviewer" {
  role    = "roles/cloudasset.viewer"
  member  = "serviceAccount:snyk-scan@snyk-iac-sample.iam.gserviceaccount.com"
  project = local.project
}

# セキュリティ審査担当者
resource "google_project_iam_member" "snyk_scan_security_reviewer" {
  role    = "roles/iam.securityReviewer"
  member  = "serviceAccount:snyk-scan@snyk-iac-sample.iam.gserviceaccount.com"
  project = local.project
}

# 閲覧者
resource "google_project_iam_member" "snyk_scan_reviewer" {
  role    = "roles/viewer"
  member  = "serviceAccount:snyk-scan@snyk-iac-sample.iam.gserviceaccount.com"
  project = local.project
}
```

:::

### スキャンの自動化

ローカルでの挙動が確認できたので、次は Snyk IaC のスキャンを自動（毎日 0:00 に定期実行）化していきます。

#### アーキテクチャー

Snyk IaC Scan におけるベストプラクティスは、他にあるかもしれませんが、今回は下記図の構成としました。

![snyk-auto-scan-architecture](/images/snyk-iac-google-cloud/snyk-auto-scan-architecture.png)
_Snyk IaC Scan Architecture_

- GitHub リポジトリ側
  - Cloud Build での Snyk IaC Scan の振る舞いを定義した `yaml`
  - Snyk IaC Scan 時に参照するバケット情報を定義した `backend.tf`
- Google Cloud 側
  - GitHub リポジトリを監視して、来たるタイミングで Snyk IaC Scan を実行する `Cloud Build`
  - 毎日 0:00 に Cloud Build を実行するように定義した `Cloud Scheduler`
  - .tfstate ファイルを保存するバケットを持つ `Cloud Storage`
  - Snyk IaC Scan の結果を出力する `Cloud Logging`

#### 1. GitHub リポジトリの用意

Terraform コードや以降で説明する Snyk IaC スキャンを実行するための yaml ファイルを管理するためのリポジトリを用意します。

本記事では 1 つのリポジトリ内に Terraform コードや yaml ファイルを定義していきます。

```plain:ディレクトリイメージ
.
├── README.md
├── api.tf              # API リソースの設定を定義する tf
├── backend.tf          # .tfstate ファイルを保存するバケットの参照先を定義する tf
├── cloudbuild          # Cloud Build から参照されるフォルダ
│   └── snyk.yaml       # Snyk IaC Scan の実行設定を定義する yaml
├── locals.tf           # .tf ファイルで共通で使用する定数群
├── log_write.sh        # Snyk IaC Scan 結果を Cloud Logging に書き出すシェルスクリプト
├── provider.tf         # Terraform プロバイダーの設定を定義する tf
├── scheduler.tf        # スケジューラーの設定を定義する tf
├── service_account.tf  # サービスアカウントや関連する IAM 設定を定義する tf
└── trigger.tf          # Cloud Build トリガーの設定を定義する tf
```

#### 2. API トークンの取得

Cloud Build が Snyk を使用するための API トークンを取得します。
手順はとても簡単で、[Snyk のアカウント設定画面](https://app.snyk.io/account) から取得することが可能です。

![api-token](/images/snyk-iac-google-cloud/api_token.png)
_Account Settings -> Auth Token_

#### 3. Secret Manager へ設定

2. で取得した Snyk API トークンを、秘匿情報を安全に扱うための Google Cloud プロダクトである [Secret Manager](https://cloud.google.com/secret-manager?hl=ja) に設定します。

![secret-manager](/images/snyk-iac-google-cloud/secret_manager.png)
_設定後イメージ_

#### 4. Cloud Build を動かすための Terraform コードを定義

```hcl:service_account.tf
locals {
  project = "your-project-name"

  snyk_roles = [
    "roles/viewer",                         # 閲覧者
    "roles/cloudasset.viewer",              # クラウドアセット閲覧者
    "roles/iam.securityReviewer",           # セキュリティ審査担当者
    "roles/cloudbuild.builds.builder",      # Cloud Build サービス アカウント
    "roles/secretmanager.secretAccessor",   # Secret Manager のシークレット アクセサー
    "roles/logging.configWriter",           # ログ構成書き込み
  ]
}

# Snyk IaC Scan 用
resource "google_service_account" "snyk_scan_service_account" {
  account_id   = "snyk-scan"
  display_name = "Snyk Integration"
  project      = local.project
}
resource "google_project_iam_member" "snyk_scan_service_account" {
  count   = length(local.snyk_roles)
  project = local.project
  member  = "serviceAccount:${google_service_account.snyk_scan_service_account.email}"
  role    = element(local.snyk_roles, count.index)
}
```

```hcl:trigger.tf
resource "google_cloudbuild_trigger" "snyk_scan_trigger" {
  name    = "snyk-scan-trigger"
  project = local.project

  source_to_build {
    uri       = "https://github.com/nozomi-koborinai/snyk-scan-sample"
    ref       = "refs/heads/main"
    repo_type = "GITHUB"
  }

  git_file_source {
    path      = "cloudbuild/snyk.yaml"
    uri       = "https://github.com/nozomi-koborinai/snyk-scan-sample"
    revision  = "refs/heads/main"
    repo_type = "GITHUB"
  }

  approval_config {
    approval_required = false
  }


  substitutions = {
    _PROJECT_ID               = local.project
    _TERRAFORM_CLIENT_VERSION = "1.4.0"
  }
  included_files  = ["cloudbuild/**"]
  service_account = google_service_account.snyk_scan_service_account.id
}
```

![sa-cloud-build](/images/snyk-iac-google-cloud/sa_cloudbuild.png)
![cloud-build](/images/snyk-iac-google-cloud/cloudbuild.png)
_設定後イメージ_

#### 5. Cloud Scheduler を動かすための Terraform コードを定義

```hcl:service_account.tf
locals {
  project = "your-project-name"

  region = "asia-northeast1"

  time_zone = "Asia/Tokyo"

  scheduler_roles = [
    "roles/iam.serviceAccountUser",
    "roles/cloudbuild.builds.builder",
    "roles/cloudscheduler.jobRunner",
  ]
}
```

```hcl:scheduler.tf
resource "google_cloud_scheduler_job" "job" {
  name      = "snyk-iac-scan-scheduler"
  region    = local.region
  time_zone = local.time_zone
  schedule  = "0 0 * * *"
  project   = local.project

  http_target {
    http_method = "POST"
    uri         = "https://cloudbuild.googleapis.com/v1/projects/${local.project}/locations/global/triggers/${google_cloudbuild_trigger.snyk_scan_trigger.trigger_id}:run"
    body        = base64encode("{\"projectId\": \"${local.project}\",\"triggerId\": \"${google_cloudbuild_trigger.snyk_scan_trigger.trigger_id}\",\"source\":{\"branchName\":\"main\"}}")
    oauth_token {
      service_account_email = google_service_account.scheduler_service_account.email
      scope                 = "https://www.googleapis.com/auth/cloud-platform"
    }
  }
}
```

![sa_scheduler](/images/snyk-iac-google-cloud/sa_scheduler.png)
![scheduler](/images/snyk-iac-google-cloud/scheduler.png)
_設定後イメージ_

#### 6. バックエンドのバケット指定

```hcl:backend.tf
terraform {
  backend "gcs" {
    bucket = "your-backend-bucket-name"
  }
}
```

:::message
ここまでできたら、Terraform Apply で実環境への適用をしておくことをおすすめします。
:::

#### 7. Snyk IaC スキャンを実行するための yaml ファイルの作成

Cloud Build から参照されて実際に Snyk IaC スキャンが実行される yaml を定義します。
さらに、Snyk IaC スキャン結果を Cloud Logging に書き込むようにしています。

:::details 補足：Cloud Logging 書き込み用のシェルスクリプト

```sh:log_wite.sh
# 引数が指定されているか確認
if [ "$#" -ne 2 ]; then
    echo "Usage: $0 snyk_log error_log "
    exit 1
fi

JSON_FILE="$1"
ERROR_FILE="$2"

# jq で JSON のフォーマットを確認
if jq . "$JSON_FILE" >/dev/null 2>&1; then
    echo "write snyk-log"
    gcloud logging write --payload-type=json snyk-log "$(cat $JSON_FILE)"
else
    echo "write snyk-error-log"
    gcloud logging write --payload-type=text snyk-error-log  "$(cat $ERROR_FILE)"
fi

```

:::

```yaml:cloudbuild/snyk.yaml
steps:
  # Snyk をインストール
  - name: 'gcr.io/cloud-builders/npm'
    dir: "cloudbuild"
    args: ['install', '-g', 'snyk']

  # Snyk を使用して IaC 管理外のリソースをチェックし、結果をログファイルに保存
  - id: "snyk iac unmanaged check"
    name: node
    dir: "cloudbuild"
    entrypoint: bash
    args:
      - -c
      - |
        npx snyk config set org="your-org-name"
        CLOUDSDK_CORE_PROJECT="${_PROJECT_ID}" npx snyk iac describe --only-unmanaged --quiet --tf-provider-version="4.68.0" --to="gcp+tf" --from="tfstate+gs://snyk-iac-sample-backend/default.tfstate" --json > /workspace/snyk_unmanaged.log 2>/workspace/snyk_error.log
        ls -lha /workspace/snyk_unmanaged.log
    secretEnv: ['SNYK_TOKEN']

  # Cloud Logging に IaC 管理外のリソースログを書き込む
  - id: "cloud logging unmanaged"
    name: "gcr.io/cloud-builders/gcloud"
    entrypoint: bash
    args:
      - -c
      - |
        apt -y update
        apt install -y jq
        chmod +x log_write.sh
        ./log_write.sh "/workspace/snyk_unmanaged.log" "/workspace/snyk_error.log"

# Secret Manager から Snyk のトークンを取得
availableSecrets:
  secretManager:
  - versionName: "projects/${_PROJECT_ID}/secrets/secret-name/versions/1"
    env: 'SNYK_TOKEN'

# その他オプションの設定
options:
  logging: CLOUD_LOGGING_ONLY
timeout: "43200s"
```

### 自動化された Cloud Build によって Snyk IaC スキャンを実行する

ついに、Snyk IaC スキャンの自動化構成が完成しました。
それでは実際に動作を確認していきましょう。

上記で Cloud Scheduler に設定した時間になったタイミングで Cloud Build のトリガーが発火します。
トリガーが発火すると Cloud Build 経由で Snyk IaC スキャンが実行される仕組みになっています。

![building](/images/snyk-iac-google-cloud/building.png)
_Cloud Build トリガー発火時_

Cloud Build 成功後、Cloud Logging を確認しに行くと、Snyk IaC スキャンの結果が書き込まれていることがわかります。

![scan-result](/images/snyk-iac-google-cloud/scan_result.png)
_自動 Snyk Iac スキャン結果_

:::message
Cloud Logging 確認の関係上、0:00 まで待てなかったので、今回に限り Cloud Scheduler を強制的に実行して、Cloud Build トリガーを発火させています。
:::

:::message alert
Snyk IaC スキャンによるリソース差分検出に関して、検出対象外のリソースも存在するため注意が必要です。
詳細は、[Google resources](https://docs.snyk.io/scan-using-snyk/scan-infrastructure/iac+-code-to-cloud-capabilities/detect-drift-and-manually-created-resources/supported-resources/google-resources) を参照してください。
:::

## まとめ

Snyk IaC を Google Cloud で活用する方法を紹介しました。
Snyk は個人的に興味のあるプロダクトの 1 つなので今後さらに、IaC での課題を解決するための最適な活用方法を模索していきたいと思います。

最後まで読んでいただきありがとうございました！

なお、本記事で紹介した Snyk IaC は、モバイルアプリ開発にも活用しているので興味のある方は下記の記事も読んでいただけると嬉しいです。
@[card](https://zenn.dev/nozomi_cobo/articles/flutter-firebase-modern-architecture)

## サンプルコード

@[card](https://github.com/nozomi-koborinai/snyk-scan-sample)
