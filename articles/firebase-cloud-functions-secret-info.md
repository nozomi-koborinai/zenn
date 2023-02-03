---
title: "Firebase Cloud Functionsで機密情報を扱う方法"
emoji: "🔐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["firebase", "cloudfunctions", "googlecloud", "secretmanager"]
published: true
---
# はじめに
Cloud Functionsの処理を書いていると、Google Maps APIなどの外部サービスを呼び出したくなる場面に遭遇することがあります。
その際に、APIキーなどの機密性の高い値をCloud Functionsの処理にハードコーディングするのは危険です。
本記事ではGoogleMapsAPIのAPIキーを例として、機密性の高い値をCloud Functionsで扱う方法を紹介します。

# Secret Manager を利用する
![](/images/gcp_firebase.png)
## Secret Manager とは？
`API キー、パスワード、証明書、その他の機密データを保存`するための、Google Cloud Platformのサービスの１種です。
Secret Managerの利用が有効になっていない方は次のリンクから有効化してください。
https://cloud.google.com/secret-manager
## Secret Manager に機密情報を保存する
ローカルプロジェクトフォルダのルートディレクトリから次のコマンドを実行します。
```
firebase functions:secrets:set YOUR_SECRET_KEY
```
:::message
`YOUR_SECRET_KEY`の箇所を保存したいキーの名称に変更してください。
あくまでも`キーを取り出すための名称`なので任意の文字列で問題ありません。
例）GOOGLE_MAP_API_KEY
:::
上記コマンドを実行すると、実際に秘匿情報として保存したいキー情報を求められるので、キー情報をコピーしてターミナルに貼り付けましょう。
その後、Enter押下することで、Secret Manager上にキー情報が保存されます。
## Secret Manager に機密情報が保存されたことを確認する
同じくローカルプロジェクトフォルダのルートディレクトリから次のコマンドを実行します。
```
firebase functions:secrets:access YOUR_SECRET_KEY
```
上記コマンド実行後、設定したキー情報が表示されることが確認できればOKです。

# Secret Manager に保存した値を Firebase Cloud Functions上 で読み取る
firebase-functionsの`runWith`メソッドを用います。
例として、本記事用に作成したメソッドを次に示します。
```typescript
/**
 * 例）Firestoreのcommandコレクションにドキュメントが生成されたら発火するFunction
 */
export const onCreateCommand = functions
  .region(`asia-northeast1`)
  .runWith({ secrets: [`GOOGLE_MAP_API_KEY`] })
  .firestore.document(`/command/{id}`)
  .onCreate(async (snapshot) => {

    // Functionsのログにキー情報を吐き出してみる
    functions.logger.info(process.env.GOOGLE_MAP_API_KEY)
    
  })
```
:::message
ポイント
- Secret Managerで保存した`キーを取り出すための名称`を、runWithメソッドのsecretsプロパティに指定する
- Secret Managerで保存したキー情報を、`process.env`を通じて参照する
:::
上記メソッドを Firebase Cloud Functions にデプロイ後、動作させてみると、Secret Managerに保存したキー情報が取得できていることが確認できました。
![](/images/secret_manager_key_value.png)