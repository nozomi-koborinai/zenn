---
title: "Flutter における Dart の Mixin の使い所を考えてみる"
emoji: "🥤"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "dart", "mixin"]
published: false
---

# はじめに
お久しぶりです、cobo です。
今回は、Dart を触っている人でもあまり馴染みのない文法である Mixin について述べようと思います。
Mixin は機能が多いため本記事では、**個人的に気持ちのよい Mixin の使い方** と題して紹介させていただきます。

## 説明すること
- Mixin の概要
- Mixin / 継承 / Utility クラスとの使い分け
- Mixin によって冗長なコードを解消する方法
## 説明しないこと
- Dart における Mixin のベストプラクティス

# Mixin とは？
Mixin は、多重継承を実現する Dart 言語の仕組みです。
クラスに他のクラスのメソッドやプロパティを追加することができます。
これにより、コードの再利用性が高まります。

# 継承とか Utility クラスでよくない？
継承や Utility クラスも役立ちますが、Mixin にはそれらにはない利点があります。
Mixin を使用することで、継承の階層が深くなることを防ぎ、コードの組み合わせが柔軟になります。

## Mixin / 継承 / Utility クラスの使い分け
Mixin: 複数のクラスで共有される機能を追加する場合
継承: 基本的なクラスの構造を変更する場合
Utility クラス: 関連性のない機能をまとめる場合

# 冗長なコードをシンプルに解消
Mixin を使用することで、重複するコードを簡潔にまとめることができます。

## サンプルコード

# 参考
- [Mixins | Dart](https://dart.dev/language/mixins)