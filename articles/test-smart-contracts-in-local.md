---
title: "作成したスマートコントラクトをローカル環境でテストする方法"
emoji: "💚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["web3", "hardhat", "solidity", "スマートコントラクト"]
published: true
---
# はじめに
私は、最近Web3分野の学習を始めました。
その学習のアウトプットとして本記事を書くことにしました。
間違った情報を記載している場合はご指摘願います。

# 本記事のゴール
- Hardhatのインストールができる
- 作成したスマートコントラクトをローカル環境でテストする方法がわかる

## 説明しないこと
- Web3の用語（スマートコントラクト、DAO、NFT、メタバース など）
- スマートコントラクタの作成方法
- Solidity言語について

それでは本題に入っていきます。

# そもそも Hardhat って？
Hardhatとは、`Solidity`言語でスマートコントラクトを開発するための開発環境のことです。
Hardhatでは主に以下のことができます。
- スマートコントラクトのコンパイル
- スマートコントラクトのローカル環境実行
- スマートコントラクトの本番環境へのデプロイ

# Hardhatのインストール
作業したいディレクトリに移動して次のコマンドを実行します。
```
mkdir your-project-name
cd your-project-name
npm init -y
npm install --save-dev hardhat
```

１例ではありますが、私の場合はこんな感じでやってます。
```
web3
 ┗ ここでコマンドを実行
```

次に、Hardhatコマンドを実行します。
```
npx hardhat
```
すると、`What do you want to do?`というメッセージが表示されます。
自分が書きたい言語（JavaScript or TypeScript）によって、
- Create a JavaScript project
- Create a TypeScript Project

の好きな方を選択しましょう。

この後にも、質問がいくつかありますが全て`Enter`キー押下で問題ないです。

# Hardhatの実行
作成されたプロジェクトフォルダ内で次のコマンドを実行し、すべてが機能していることを確認します。
```
 npx hardhat compile
```
```
 npx hardhat test
```
次のように表示されればOKです。
![](/images/hardhat_complete.png)
![](/images/hardhat_directly.png)

最後に、`test`フォルダ、`contracts`フォルダ配下にある`Lock`ファイルを削除しておきましょう。

# スマートコントラクトのローカル環境でのテスト
次のコマンドを入力することでローカル環境で実行することができます。
```
npx hardhat run scripts/run.js
```

# スマートコントラクトをローカル環境にデプロイ
次のコマンドを入力することでスマートコントラクトをローカル環境にデプロイすることができます。
```
npx hardhat node
```
```
npx hardhat run scripts/deploy.js --network localhost
```
:::message
2つ目のコマンドは別のターミナルを立ち上げてそちらで実行してください。
:::

# 参考
https://dev.classmethod.jp/articles/hardhat-quick-start/
https://www.sbbit.jp/article/fj/40394