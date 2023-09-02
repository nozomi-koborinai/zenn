---
title: "SRE 視点で捉える Firebase の Terraform 自動化: Cloud Build 実践ガイド"
emoji: "🔧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["googlecloud", "gcp", "cloudbuild", "firebase", "terraform"]
published: true
---

## はじめに

こんにちは、最近 SRE（Site Reliability Engineering）領域に興味のある [cobo](https://github.com/nozomi-koborinai) です。
今回はモバイルアプリ開発向けの SRE を意識したテーマで記事を執筆させていただきます。

### 説明すること

- トイルの撲滅
- Cloud Build で Firebase プロジェクトの Terraform を自動実行する方法
  - Cloud Build の基本的な使い方
  - Cloud Build の構築フロー

### 説明しないこと

- 複数環境（development, staging, production）に対応した管理方法
  ※ 本記事の応用版となるため、別記事にて紹介させていただきます。
- Terraform の使用方法
  ※ Terraform の基本的な使い方 および Firebase プロジェクトを管理する方法に関しては、私が過去に執筆した [Terraform で Firebase プロジェクトを構築してみる](https://zenn.dev/cloud_ace/articles/b791cce386d523) を参照してください。

## トイルの撲滅

数多くある SRE の考え方の中で、現時点で特に面白いなと感じているのが、この **[トイルの撲滅](https://www.oreilly.com/library/view/sre-google/9784873117911/ch05.xhtml)** という考え方です。

トイルは、**単調で反復的な作業や手動の運用作業** のことを指します。
これらの作業は基本的に、自動化することで省略可能とされています。

そこで、趣味の Flutter アプリ開発を考えてみたいと思います。
mBaaS である Firebase のセットアップは、アプリごとにプロジェクト名を入力したり、リージョンを設定したり、各種 API を有効化したりといった作業を行なっています。
この部分にフォーカスしてみると、これらの内容は反復的な作業や手動の運用作業でありヒューマンエラーが起こりうる、トイル作業に当てはまるものではないかと考えました。
トイル作業を削減することで、本来開発者が時間をかけるべき開発作業、技術のキャッチアップ等に集中することができるようになります。

上記を踏まえ、本記事では Firebase プロジェクトに対して Cloud Build × Terraform を使用してトイルを撲滅（削減）し、より効率的な運用を実現する方法を探求していきます。

:::message
Firebase プロジェクトの構成管理を自動化するための Google Cloud 上の作業は手作業で行なっていますが、この点ご了承ください。
:::

## Cloud Build とは

![cloud-build](/images/firebase-terraform-cloud-build-guide/cloud_build.png =240x)

Cloud Build とは Google Cloud の提供するフルマネージド型の CI/CD プラットフォームです。

特徴として以下のものが挙げられます。

- 拡張性
  - カスタムビルドステップやプラグインを使用して、特定のタスクやツールを簡単に統合可能
- スピード
  - 高速なビルドとテストが可能
  - ビルドプロセスが並行して行われるため、効率的にアプリケーションのデリバリが可能

プロダクト詳細については、公式ドキュメントを参照してください。
https://cloud.google.com/build?hl=ja

### 料金

無料枠も用意されているため、個人で気軽にお試しすることができるプロダクトとなっています。
| 機能 | 料金 |
| --- | --- |
| 1 日ごとの最初の 120 分 | 無料 |
| 以降 1 分 ごと | $0.003 |

さらに詳しい料金体系については、公式ドキュメントを参照してください。
https://cloud.google.com/build/pricing?hl=ja

## アーキテクチャ

本記事で扱う構成を以下の図で表してみます。
![architecture](/images/firebase-terraform-cloud-build-guide/architecture.png)

- ローカル環境で Terraform コードを実装する
- 実装したコードを GitHub 上のリポジトリに push する
- Cloud Build が GitHub 上のリポジトリの変更を検知して、自動テスト／自動デプロイ を実行する
- 実装したコードをもとに Cloud Build によって Firebase プロジェクトの構成情報が更新される

:::message
上記図を頭に入れておくことで、これから説明する構築手順が図中のどの箇所に対応しているか理解しやすくなります。
:::

## Cloud Build で Firebase プロジェクトの Terraform を自動実行

それでは Cloud Build の使い方を覚えながら Firebase プロジェクトの Terraform を自動実行するための環境を構築していきましょう。

大まかな手順は次のとおりです。

| 手順 | 概要                                                              |
| ---- | ----------------------------------------------------------------- |
| 1    | Firebase プロジェクト管理用の GitHub リポジトリを作成する         |
| 2    | ブランチ戦略を考える                                              |
| 3    | Google Cloud プロジェクトを手動で作成する                         |
| 4    | Cloud Build API を有効化する                                      |
| 5    | Cloud Build に GitHub リポジトリを連携する                        |
| 6    | Cloud Build で Terraform を実行するための YAML ファイルを作成する |
| 7    | Cloud Build で Terraform を実行するためのトリガーを作成する       |
| 8    | tfstate ファイルを置くための Cloud Storage バケットを作成する     |
| 9    | Firebase コンソールに表示するためのラベルを設定する               |
| 10   | Cloud Build サービスアカウントに Deploy 権限を付与                |
| 11   | Firebase プロジェクト管理用の Terraform コードを書く              |
| 12   | 書いたコードを GitHub に push                                     |

### 1. Firebase プロジェクト管理用の GitHub リポジトリを作成する

特別な操作は必要ありません。
通常通りの操作で空のリポジトリを作成しましょう。
今後、ここに Firebase プロジェクトの構成情報を管理するためのコードを書いていきます。

![create-empty-repository](/images/firebase-terraform-cloud-build-guide/empty-repository.png)

### 2. ブランチ戦略を考える

Cloud Build に限らず、CI/CD パイプラインを構築する際、ブランチ戦略は重要な項目なので以下に流れを定義してみます。

- `main` ブランチ上で直接作業は行わない
- 作業は `main` ブランチから分岐した `feature` ブランチ上で行う
  - PR（Pull Request）作成時 Terraform のコードフォーマットを確認する
  - 問題がなければ Terraform Plan を実行するための YAML ファイルをトリガーする
  - PR レビューで承認を受けたら `main` ブランチへのマージが可能となる
  - `main` ブランチへのマージ時には、Terraform Apply を実行するための YAML ファイルをトリガーする

### 3. Google Cloud プロジェクトを手動で作成する

Google Cloud プロジェクトの一部として Firebase プロジェクトが存在しているため、まずは手動で Google Cloud プロジェクトを作成する必要があります。
1 つの Firebase プロジェクトを格納する 1 つの 器を作ってあげるイメージですね。
![firebase-image](/images/firebase-terraform-cloud-build-guide/firebase-image.png)
![create-google-cloud-prj](/images/firebase-terraform-cloud-build-guide/google-cloud-new-project.png)

### 4. Cloud Build API を有効化する

CI/CD 環境を構築する役割を担う Cloud Build を立ち上げます。
![serarch-cloud-build](/images/firebase-terraform-cloud-build-guide/search-cloud-build.png)
![cloud-build-api](/images/firebase-terraform-cloud-build-guide/cloud-build-api.png)

### 5. Cloud Build に GitHub リポジトリを連携する

Cloud Build に対して 1. の手順で作成した GitHub リポジトリを連携します。
![cloud-build-repository-connect1](/images/firebase-terraform-cloud-build-guide/cloud-build-repository-connect1.png)
![cloud-build-repository-connect2](/images/firebase-terraform-cloud-build-guide/cloud-build-repository-connect2.png)
![cloud-build-repository-connect3](/images/firebase-terraform-cloud-build-guide/cloud-build-repository-connect3.png)
![cloud-build-repository-connect4](/images/firebase-terraform-cloud-build-guide/cloud-build-repository-connect4.png)
![cloud-build-repository-connect5](/images/firebase-terraform-cloud-build-guide/cloud-build-repository-connect5.png)

### 6. Cloud Build で Terraform を実行するための YAML ファイルを作成する

Cloud Build トリガーによって実行される Terraform の実行ファイルを作成します。
:::message
Cloud Build トリガー自体は 7. の手順で作成するため、この時点では特に意識しなくても問題ありません。
:::

- Pull Request 作成時に `terraform plan` 実行
- main ブランチへのマージ時に `terraform apply` 実行

というブランチ戦略であるため Terraform 実行のための YAML ファイルを以下のように 2 種類定義します。

```plane:YAML ファイルの構成
.
└── cloudbuild
    ├── plan.yaml
    └── apply.yaml
```

```yaml:plan.yaml
steps:
  # Terraform コードのフォーマットを確認する
  # コードが正しくフォーマットされていない場合、差分を表示する
  - id: "terraform-fmt-check"
    name: "hashicorp/terraform:$_TERRAFORM_CLIENT_VERSION"
    args: ["fmt", "-check", "-diff"]

  # Terraform の初期化を実行
  # プロジェクト内の .tf ファイルを読み込み、必要なプロバイダの設定を行う
  - id: "terraform-init"
    name: "hashicorp/terraform:$_TERRAFORM_CLIENT_VERSION"
    args: ["init"]

  # Terraform での変更内容をプレビューする
  # このステップでは、実際のリソースへの変更は行われない
  - id: "terraform-plan"
    name: "hashicorp/terraform:$_TERRAFORM_CLIENT_VERSION"
    args: ["plan", "-lock-timeout=120s", "-var=billing_account=${_BILLING_ACCOUNT}"]

timeout: "3600s"
```

```yaml:apply.yaml
steps:
  # Terraform コードのフォーマットを確認する
  # コードが正しくフォーマットされていない場合、差分を表示する
  - id: "terraform-fmt-check"
    name: "hashicorp/terraform:$_TERRAFORM_CLIENT_VERSION"
    args: ["fmt", "-check", "-diff"]

  # Terraform の初期化を実行
  # プロジェクト内の .tf ファイルを読み込み、必要なプロバイダの設定を行う
  - id: "terraform-init"
    name: "hashicorp/terraform:$_TERRAFORM_CLIENT_VERSION"
    args: ["init"]

  # Terraform での変更内容をプレビューする
  # このステップでは、実際のリソースへの変更は行われない
  - id: "terraform-plan"
    name: "hashicorp/terraform:$_TERRAFORM_CLIENT_VERSION"
    args: ["plan", "-lock-timeout=120s", "-var=billing_account=${_BILLING_ACCOUNT}"]

  # Terraform での変更内容を実際に適用する
  # このステップで、リソースへの変更を行う
  - id: "terraform-apply"
    name: "hashicorp/terraform:$_TERRAFORM_CLIENT_VERSION"
    args: ["apply", "-auto-approve", "-lock-timeout=120s", "-var=billing_account=${_BILLING_ACCOUNT}"]

timeout: "3600s"
```

#### Terraform 実行のための YAML ファイルの役割と仕組み

- steps
  - 実行する一連の作業を定義する
  - 各 step は順番に実行される
- id
  - 各 ステップに名前を付けることが可能
  - 特定のステップを識別する際に使用する
- name
  - 使用する Docker イメージの名前を指定する
  - 今回は [hashicorp/terraform](https://hub.docker.com/r/hashicorp/terraform/) イメージを使用して Terraform コマンドを実行している
- args
  - ステップの実行時に使用するコマンドやオプションを指定する
  - 今回は Terraform の init, plan, apply などのコマンドとそのオプションを設定している
- timeout
  - ステップの最大実行時間を指定する
  - 指定時間を超えるとビルドは失敗となる

:::message
次に説明する Cloud Build トリガー作成で YAML ファイルを参照するよう設定する必要があるため、この時点で対象リポジトリの main ブランチに push しておきましょう。
:::

### 7. Cloud Build で Terraform を実行するためのトリガーを作成する

Cloud Build で Terraform 実行のための YAML ファイルを指定したトリガーを 2 種類作成していきます。

![cloud-build-trigger](/images/firebase-terraform-cloud-build-guide/cloud-build-trigger.png)

#### PR 作成トリガー

![pr-trigger1](/images/firebase-terraform-cloud-build-guide/cloud-build-pr-trigger1.png)
![pr-trigger2](/images/firebase-terraform-cloud-build-guide/cloud-build-pr-trigger2.png)
![pr-trigger3](/images/firebase-terraform-cloud-build-guide/cloud-build-pr-trigger3.png)

##### 解説

`Cloud Build 構成ファイルの場所` 欄には、手順 6. で作成してリポジトリに push した plan.yaml の場所を指定してあげます。

変数 `_TERRAFORM_CLIENT_VERSION` には柔軟性を持たせてバージョンを自由に変更して渡せるようにしています。
また、変数 `_BILLING_ACCOUNT` には Google Cloud の `請求先アカウント ID` を指定しています。
Cloud Build 上で指定した変数は YAML ファイルの中で受け取れるようになっています。

```yaml:plan.yaml で変数を受け取っている部分
  - id: "terraform-plan"
    name: "hashicorp/terraform:$_TERRAFORM_CLIENT_VERSION"
    args: ["plan", "-lock-timeout=120s", "-var=billing_account=${_BILLING_ACCOUNT}"]
```

:::message alert
Firebase プロジェクトを Terraform 管理するためには、Blaze（従量制）である必要があるため、請求先アカウント ID を必ず指定してください。
:::

#### main ブランチへのマージトリガー

![merge-trigger1](/images/firebase-terraform-cloud-build-guide/cloud-build-merge-trigger1.png)
![merge-trigger2](/images/firebase-terraform-cloud-build-guide/cloud-build-merge-trigger2.png)
![merge-trigger3](/images/firebase-terraform-cloud-build-guide/cloud-build-merge-trigger3.png)

:::details PR 作成トリガー.解説 とほぼ同じ

`Cloud Build 構成ファイルの場所` 欄には、手順 6. で作成してリポジトリに push した apply.yaml の場所を指定してあげます。

変数 `_TERRAFORM_CLIENT_VERSION` には柔軟性を持たせてバージョンを自由に変更して渡せるようにしています。
また、変数 `_BILLING_ACCOUNT` には Google Cloud の `請求先アカウント ID` を指定しています。
Cloud Build 上で指定した変数は YAML ファイルの中で受け取れるようになっています。

```yaml:plan.yaml で変数を受け取っている部分
  - id: "terraform-apply"
    name: "hashicorp/terraform:$_TERRAFORM_CLIENT_VERSION"
    args: ["apply", "-auto-approve", "-lock-timeout=120s", "-var=billing_account=${_BILLING_ACCOUNT}"]
```

:::message alert
Firebase プロジェクトを Terraform 管理するためには、Blaze（従量制）である必要があるため、請求先アカウント ID を必ず指定してください。
:::

#### トリガー一覧確認

下記のようなトリガーの一覧になっていれば問題ありません。
![cloud-build-trigger-list](/images/firebase-terraform-cloud-build-guide/cloud-build-trigger-list.png)

### 8. tfstate ファイルを置くための Cloud Storage バケットを作成する

Firebase プロジェクトの構成情報を保持している tfstate ファイルを保存するための Cloud Storage バケットを作成します。
ローカルではなくリモート（Google Cloud）上に保持する理由として、以下のものが挙げられます。

- チームでの共有
  - リモートに tfstate を置くことで、チーム内での tfstate の共有や競合の回避が可能
- バックアップ
  - Cloud Storage 上に置くことで、自動的にバックアップ、バージョン管理が行われるためリカバリーが容易
- セキュリティ
  - Google Cloud のセキュリティポリシーや暗号化機能を利用して、tfstate の安全性を高めることが可能

それでは以下のスクリーンショットを参考に Cloud Storage バケットを作成していきましょう。

![create-storage-bucket1](/images/firebase-terraform-cloud-build-guide/create-storage-bucket1.png)
![create-storage-bucket2](/images/firebase-terraform-cloud-build-guide/create-storage-bucket2.png)
![create-storage-bucket3](/images/firebase-terraform-cloud-build-guide/create-storage-bucket3.png)
![create-storage-bucket4](/images/firebase-terraform-cloud-build-guide/create-storage-bucket4.png)
![create-storage-bucket5](/images/firebase-terraform-cloud-build-guide/create-storage-bucket5.png)
![create-storage-bucket6](/images/firebase-terraform-cloud-build-guide/create-storage-bucket6.png)
![create-storage-bucket7](/images/firebase-terraform-cloud-build-guide/created-storage-bucket.png)

### 9. Firebase コンソールに表示するためのラベルを設定する

Firebase として扱うためのラベルを付与しないと、Terraform Apply が正常に完了したとしても Firebase のコンソール上に表示されません。
それを回避するための手順がこちらになります。

![add-label](/images/firebase-terraform-cloud-build-guide/add-label.png)

### 10. Cloud Build サービスアカウントに Deploy 権限を付与

デフォルトの状態だと、Cloud Build のサービスアカウントは Deploy の権限を保持していません。
CI/CD パイプラインによって Cloud Build から Firebase プロジェクトへ更新を行うためのロールを付与します。

![service-account](/images/firebase-terraform-cloud-build-guide/service-account.png)

:::message
本来であれば、[最小権限の原則](https://ja.wikipedia.org/wiki/%E6%9C%80%E5%B0%8F%E6%A8%A9%E9%99%90%E3%81%AE%E5%8E%9F%E5%89%87) に従い、Cloud Build サービスアカウントが操作を行うにあたっての必要最小限の権限を持つロールのみを付与することが望ましいです。
:::

### 11. Firebase プロジェクト管理用の Terraform コードを書く

これで Cloud Build × Terraform で Firebase プロジェクトの管理を自動化する環境を構築することができました。
ここからは定義したデプロイ戦略に則ってインフラをコード化してみましょう。

以下のリポジトリを参考にしてみてください。
https://github.com/nozomi-koborinai/firebase-cicd-terraform-single

### 12. 書いたコードを GitHub に push

コードが書けたら GitHub に push をして Cloud Build のトリガーを発火させていきましょう。

#### PR 作成を行い Terraform Plan トリガーを発火

`feature` ブランチから `main` ブランチへの PR 作成を行います。

![create-pr1](/images/firebase-terraform-cloud-build-guide/create-pr1.png)

GitHub 上からも Terraform Plan トリガーが発火したことが確認できます。
自分で作った環境が動いていることが目に見えるととても気持ちがいいですよね。

![create-pr2](/images/firebase-terraform-cloud-build-guide/create-pr2.png)

さらに、Cloud Build コンソールに移動することで詳細まで確認することができます。

![run-cloud-build1](/images/firebase-terraform-cloud-build-guide/run-cloud-build1.png)
![run-cloud-build2](/images/firebase-terraform-cloud-build-guide/run-cloud-build2.png)
![runed-build](/images/firebase-terraform-cloud-build-guide/runed-build.png)

Terraform Plan の結果が意図したものであれば `main` ブランチへのマージを行い、Terraform Apply で実際の Firebase 環境に変更を加えていきましょう。

#### main ブランチへのマージを行い Terraform Apply トリガーを発火

![merge](/images/firebase-terraform-cloud-build-guide/merge.png)

マージを行うと、Terraform Apply のトリガーが発火します。緊張しますね。
コンソールで詳細を確認してみましょう。

![run-cloud-build3](/images/firebase-terraform-cloud-build-guide/run-cloud-build3.png)

Terraform Apply も無事に成功したようです。
![apply-done](/images/firebase-terraform-cloud-build-guide/apply-done.png)

#### Firebase プロジェクトのコンソールを確認

Cloud Build のログから、Terraform Apply が無事に成功したことが確認できました。
それでは実際に Firebase プロジェクトを見に行ってみましょう。

##### Firebase プロジェクト一覧

問題なく Firebase プロジェクトが作られました。
![firebase-console](/images/firebase-terraform-cloud-build-guide/firebase-console.png)

##### Authentication の認証プロバイダ

コードの内容がコンソール上に反映されていることも確認できます。
![authentication-console](/images/firebase-terraform-cloud-build-guide/authentication-console.png)

```hcl:Firebase Authentication の構成情報を管理するコード
resource "google_identity_platform_project_default_config" "default" {
  provider = google-beta
  project  = var.project_id
  sign_in {
    allow_duplicate_emails = true
    anonymous {
      enabled = true
    }
    email {
      enabled           = true
      password_required = false
    }
    phone_number {
      enabled = true
      test_phone_numbers = {
        "+11231231234" = "000000"
      }
    }
  }
  depends_on = [google_identity_platform_config.default]
}
```

## 参考

https://zenn.dev/cloud_ace/articles/b791cce386d523
https://cloud.google.com/build?hl=ja
https://firebase.google.com/docs/projects/terraform/get-started?hl=ja
https://cloud.google.com/blog/ja/products/gcp/identifying-and-tracking-toil-using-sre-principles
https://www.oreilly.com/library/view/sre-google/9784873117911/ch05.xhtml
