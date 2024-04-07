---
title: "Flutter × Firebase におけるモダンなアプリケーションアーキテクチャへの挑戦"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["googlecloud", "gcp", "firebase", "flutter", "snyk"]
published: true
---

本記事は Flutter 大学アドベントカレンダー 7 日目の記事です。
（空きが出たので急遽参加させていただきました！）

@[card](https://qiita.com/advent-calendar/2023/flutteruniv)

## はじめに

こんにちは、アプリケーション開発を軸とした SRE 領域に興味のある cobo です。

現在、Flutter 大学内の友人と Flutter アプリを開発しています。
そこでは、大きく **Flutter × Firebase** といった比較的馴染みのある技術構成を採用しています。

これら 2 つの技術構成のみで迅速なアプリケーション開発が可能となりますが、
今回は加えて、**プロジェクト管理と運用の効率化**、**セキュリティ面の強化** を意識したアプリケーションアーキテクチャに挑戦してみました。

僕の中では、従来の Flutter × Firebase 構成で大きな課題は感じていなかったものの、**Firebase の特徴を生かすことで、さらにセキュアな構成 かつ プロジェクト管理や運用の効率化が見込めるのでは？** と感じ本挑戦に至りました。

### Firebase の特徴

**Firebase は Google Cloud の一部** です。

そのため、Firebase プロジェクトを作成すると、全く同じプロジェクト名で Google Cloud プロジェクトが自動的に作成されます。

つまり、Firebase の構成を拡張したい場合は、同 Google Cloud プロジェクトのプロダクトを活用すると良いということになります。

## アプリケーションアーキテクチャの紹介

Flutter × Firebase アプリ開発におけるアプリケーション全体としてのアーキテクチャ（以下、アプリケーションアーキテクチャ）を下記に示します。

![firebase-architecture](/images/flutter-firebase-modern-architecture/firebase-architecture.png)
_現在開発中の Flutter アプリにおけるアーキテクチャの全体像_

## 技術構成の概要

- VCS
  - GitHub
- CI / CD
  - GitHub Actions
- Frontend
  - Flutter
- Backend
  - Firebase
- Infrastructure
  - Google Cloud
- IaC (Infrastructure as Code)
  - Terraform
- 脆弱性管理
  - Snyk

### VCS

@[card](https://github.com/Mood-Trend)

下記 4 リポジトリで構成されています。

- Frontend
  - Flutter アプリのコード
- Backend
  - Firebase
    - Firebase Functions に関するコード
    - Firebase Hosting に関する静的ファイル
- IaC (Infrastructure as Code)
  - Firebase / Google Cloud のリソース状態を Terraform にて管理するコード
  - Cloud Firestore の Security Rules や Clous Storage for Firebase の Security Rules、Firestore のマスタドキュメントデータなどもこちらで管理している
- 脆弱性管理
  - Firebase / Google Cloud の実リソースとコード状態の差分を検知するためのコード

### CI / CD

冒頭で示したアプリケーションアーキテクチャから抜き出すと下記部分に該当します。
IaC リポジトリ および バックエンドリポジトリが対象です。

![cicd](/images/flutter-firebase-modern-architecture/cicd.png)
_GitHub Actions から Google Cloud (Firebase 含む) へのセキュアな通信_

#### パイプライン

まずは、CI / CD のパイプラインを整理してみます。

```mermaid
flowchart TD
    PR[Pull Request] --> CI{Auto check & test}
    CI -->|OK| Review{Code review}
    CI -->|NG| Fix[Fix codes]
    Fix --> PR
    Review -->|Approve| Merge[Merge to main]
    Review -->|Change Request| Fix
    Merge --> Dev[Deploy to Dev env]
    Tag[Push tag v.*] --> Prd[Deploy to Prd env]
```

- `Pull Request` 作成時
  - CI 実行
    - Firebase Functions フォーマットチェック、静的解析
    - Terraform フォーマットチェック、実行計画
- `main` ブランチマージ時
  - 開発環境に対する CD 実行
    - Firebase デプロイ
      - Firebase Functions
      - Firebase Hosting
    - Terraform 適用
- `tag` 付与時
  - 本番環境に対する CD 実行
    - Firebase デプロイ
      - Firebase Functions
      - Firebase Hosting
    - Terraform 適用

#### Workload Identity 連携の活用

GitHub Actinos と Google Cloud (Firebase 含む) 間で Workload Identity を使用するメリットとして、長期に渡って有効であったサービスアカウントキーに代わって、短時間で有効期限が切れるトークンを利用する点が挙げられます。

これにより、セキュリティリスクを大幅に減少させることができます。

GitHub Actions は Google Cloud、Firebase のリソースにアクセスする際に、外部の ID プロバイダから得たトークンを使って、Google Cloud の短期アクセストークンと交換して、セキュアに認証を行うことができます。

@[card](https://cloud.google.com/blog/ja/products/identity-security/enabling-keyless-authentication-from-github-actions)

:::details GitHub Actions Workflows 側のコード例

```yaml:cd_dev.yaml
name: CD Development
on:
  push:
    branches:
      - main
jobs:
  deploy_functions:
    name: Deploy Functions
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    timeout-minutes: 60
    env:
      PROJECT_ID: ${{ secrets.PROJECT_ID_DEV }}
      WORKLOAD_IDENTITY_PROVIDER: ${{ secrets.WORKLOAD_IDENTITY_PROVIDER_DEV }}
      SERVICE_ACCOUNT: ${{ secrets.SERVICE_ACCOUNT_DEV }}
    steps:
      - name: Check out repository
        uses: actions/checkout@v2
      - name: "Authenticate to Google Cloud"
        uses: "google-github-actions/auth@v1"
        with:
          workload_identity_provider: ${{ env.WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ env.SERVICE_ACCOUNT }}
          create_credentials_file: true
          export_environment_variables: true
      - name: Setup node
        uses: actions/setup-node@v1
        with:
          node-version: 18
      - name: Install Dependencies
        run: npm install
        working-directory: ./functions
      - name: Deploy to Functions
        run: |
          npm install -g firebase-tools
          firebase deploy --only functions --force --project=${{ env.PROJECT_ID }}
```

:::

### Frontend / Backend

こちらは比較的シンプルな構成となっています。

![frontend-backend](/images/flutter-firebase-modern-architecture/front_back.png)
_Mobile Frontend: Flutter, Backend: Firebase_

#### Flutter

##### Frontend Architecture

![flutter-architecture](/images/flutter-firebase-modern-architecture/frontend_architecture.png)
_MVC をベースとした 4 層レイヤードアーキテクチャ_

以下のルールに則っています。

1. Presentation Layer には View に関するコードを記載する。
   直接 Firebase の各種 API をコールするようなコードを記載してはいけない。
   Firebase 側の処理を実行したい場合は、Application Layer の Usecase クラスのメソッドをコールする。
1. Application Layer には Presentation Layer（View）と Firebase の橋渡しとなるコードを記載する。
1. Domain Layer にはビジネスロジックとエンティティを記載する。
1. Infrastructure Layer には具体的なデータの取得や保存の実装を行う。
   例えば、Firebase, API エンドポイント, データベースなどとの通信や操作をこのレイヤーで行う。
   直接 View 側のコードから呼ばれることはない。

##### Flutter におけるレイヤードアーキテクチャの詳細はこちらから

別記事にて執筆しています。
@[card](https://zenn.dev/flutteruniv/books/flutter-architecture/viewer/5_layered-architecture)

#### Firebase

下記 4 プロダクトのみで構成されています。

##### Firebase Authentication

![anonymous](/images/flutter-firebase-modern-architecture/anonymous.png)
_匿名認証_

##### Cloud Firestore

アプリのデータ管理に使用しています。
:::details Firestore 設計例

```mermaid
erDiagram
  user {
    string display_name
    string image_url
    timestamp created_at
    timestamp updated_at
  }

  conf {
    number max_planned_volume
		bool isOnboardingCompleted
    timestamp created_at
    timestamp updated_at
  }

  mood_worksheet {
    string minus_5
    string minus_4
    string minus_3
    string minus_2
    string minus_1
    string plus_1
    string plus_2
    string plus_3
    string plus_4
    string plus_5
    timestamp created_at
    timestamp updated_at
  }

  mood_points {
    number point
    number planned_volume
    timestamp mood_date
    timestamp created_at
    timestamp updated_at
  }

  users ||--o{ user : contains
  user ||--|{ conf : has
  user ||--|{ mood_worksheet : has
  user ||--|{ mood_points : has
```

:::

##### Cloud Functions for Firebase

アプリユーザーの追加 / 削除によって起動する関数を定義しています。

- 追加時
  - 該当ユーザーに紐づく Firestore 上のデフォルトのアプリデータの追加
- 削除時
  - 該当ユーザーに紐づく Firesotre 上のアプリデータ削除

##### Firebase Hosting

静的なコンテンツを配信しています。

- [LP](https://mood-trend-prod.web.app/)
- [プライバシーポリシー](https://mood-trend-prod.web.app/privacy-policy.html)
- [利用規約](https://mood-trend-prod.web.app/term-of-service.html)

### Infrastructure / IaC (Infrastructure as Code)

![iac](/images/flutter-firebase-modern-architecture/iac.png)
_Terraform によって Google Cloud と Firebase 環境をコードで管理_

直接コンソール上から値を変更できるのは、便利ではありますが、意図しない変更によりエラーを招いてしまう可能性があります。

この課題を解決するために Terraform によるインフラ環境のコード管理を導入しました。

#### Terraform の概要

Terraform は、Google Cloud や Firebase などのクラウドプロバイダーのインフラストラクチャの状態をコードとして表現することで、バージョン管理、共有、再利用が容易なインフラを可能にします。

例えば、開発、ステージング、本番環境などの複数環境での設定差異を最小化することで、コンソールからの手作業による予期せぬエラーを防ぐこともできます。

Firebase での環境構築をイメージすると、必要な環境分コンソール上から

- プロジェクト名の入力が必要
- Firebase Authentication のプロバイダーの有効化が必要
- Cloud Firestore にマスタデータの登録が必要（サブコレクションになっていたりするととても面倒。さらに入力ミスが発生しやすい）

などが考えられますよね。

これらをコードで共通化することで、どの環境にも一貫性を持った状態で適用することができます。

下記は、Cloud Firestore のリソースを dev / prod で共通管理する際のコード例です。
これによって、初期コレクション／ドキュメントのデータソースを 1 箇所に集約することができるので、コンソールからの入力ミスなどによる環境差分は発生しなくなります。

```hcl:Firestore リソースのコード共通化例
locals {
  # Firestore master data
  docs = [
    {
      collection  = "app_confs"
      document_id = "is_show_review_menu"
      fields = jsonencode({
        "value" = {
          "booleanValue" = false
        },
      })
    },
    {
      collection  = "app_confs"
      document_id = "review_url_ios"
      fields = jsonencode({
        "value" = {
          "stringValue" = " "
        },
      })
    },
    {
      collection  = "app_confs"
      document_id = "review_url_android"
      fields = jsonencode({
        "value" = {
          "stringValue" = " "
        },
      })
    },
  ]
}

# Firebase Firestore Instance
resource "google_firestore_database" "default" {
  project                     = var.project_id
  name                        = "(default)"
  location_id                 = local.region
  type                        = "FIRESTORE_NATIVE"
  concurrency_mode            = "OPTIMISTIC"
  app_engine_integration_mode = "DISABLED"

  depends_on = [
    google_firebase_project.default,
    google_project_service.default,
  ]
}

# Firebase Firestore コレクション／ドキュメント定義
resource "google_firestore_document" "docs" {
  for_each    = { for doc in local.docs : doc.document_id => doc }
  provider    = google-beta
  project     = var.project_id
  collection  = each.value.collection
  document_id = each.value.document_id
  fields      = each.value.fields

  depends_on = [
    google_firestore_database.default,
  ]
}
```

:::message
なお、Terraform の CI / CD パイプラインも、上記 [Workload Identity 連携](#ci-%2F-cd) を行って、GitHub Actions から Google Cloud や Firebase に対するセキュアなアクセスをしています。
:::

### 脆弱性管理

![snyk-scan](/images/flutter-firebase-modern-architecture/snyk-scan.png)
_意図しないリソース変更を検知するための仕組み_

Snyk に入る前に Terraform でのリソース状態管理の方法について確認しましょう。

#### Terraform リソースの状態管理

- `.tf` ファイルでリソースをコード化
- `.tfstate` ファイルでリソースの状態を保持
- `terraform plan` or `terraform apply` 実行時のリソース差分検出で、`.tf` ファイルと `.tfstate` ファイル を比較している

そのため、

**実際の Google Cloud や Firebase プロジェクトに対して、意図しないリソース変更が発生した場合に検知が難しい**

といった課題が挙げられます。
例）IaC 化がルールのプロジェクトにおいて、リソースを手で追加してしまう、悪意を持った人が大量のリソースを追加、変更、削除してしまう、Firewall 設定を変更されてしまいそれに気付けない 等

このような課題を解決するために、**Snyk IaC** を活用していきます。

Snyk IaC を活用することで実際の環境とコードでの比較ができるようになるので、意図しないリソース変更を検知することが可能となります。

今回のアプリでは、毎日午前 3:00 に Snyk IaC のスキャンを実行してその結果を Cloud Logging に書き出すようにしています。
ちなみに、現在の IaC カバレッジは 95 % のようです。

![log](/images/flutter-firebase-modern-architecture/log.png)
_Cloud Logging のログの一部_

いずれはアラートの発報まで実装できたらいいなと思っています。

こちらの Snyk IaC に関する記事は Snyk 社のアドベントカレンダーでも書いたので、興味のある方はそちらも見て頂けると嬉しいです！

@[card](https://zenn.dev/nozomi_cobo/articles/snyk-iac-google-cloud)

## まとめ

![firebase-architecture](/images/flutter-firebase-modern-architecture/firebase-architecture.png)

- Firebase プロジェクトを作成すると、全く同じプロジェクト名で Google Cloud プロジェクトも作成される
- GitHub Actions -> Google Cloud や Firebase へのセキュアなアクセスの方法として Workload Identity 連携を活用する
- 意図しないリソース変更によるエラーを防ぐため Terraform を活用する
- 意図しないリソース検知には Snyk を活用する

---

技術的な挑戦ができたことで、自身の技術領域が少しだけ広がった気がしました。

リリースはまだですが、リリース後の運用を意識して、アプリケーション全体の設計をしていきたいと思います。

最後まで読んでいただきありがとうございました！！
