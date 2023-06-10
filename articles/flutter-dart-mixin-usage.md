---
title: "Flutter における Dart の Mixin を気持ちよく使ってみる"
emoji: "🍹"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "dart", "mixin"]
published: true
---

こんにちは、[cobo](https://linktr.ee/nozomi_koborinai) です。
今回は、`Flutter における Dart の Mixin を気持ちよく使ってみる` というテーマで本記事を執筆させていただきます。
なお、Mixin のベストプラクティスを紹介する記事ではないため、その点ご了承ください。

## Mixin とは？

`Mixin` は、多重継承を実現する Dart 言語の仕組みです。
これにより、コードの再利用性を向上させることが可能です。
詳細については、
[【Flutter/Dart】mixin とは？概念や使い方を徹底解説！](https://terupro.net/flutter-dart-grammar-mixin/)
の記事にきれいにまとめられています。

## Mixin を使用しないコードを書いてみる

まずは、Mixin を使用しないコードを書いてみます。

概要は次のとおりです。

- 1 つの画面に 3 つのボタンを配置する
- それぞれのボタンがタップされると、次の処理を同期的に実行する
  - 画面上にローディングアニメーションを表示する
  - ボタンタップ時に実行したい主となる処理を呼び出す ※
  - 成功時のメッセージをスナックバーに表示する
  - 画面上に表示されているローディングアニメーションを停止する

:::message
※ `ボタンタップ時に実行したい主となる処理を呼び出す` とは具体的に、外部サービスの処理呼び出しのことを指しています。
また、今回のサンプルコードでは外部サービスの処理を呼び出す想定で説明をしていきます。
:::

```dart
/// ホームページ
class HomePage extends ConsumerWidget {
  const HomePage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final scaffoldMessenger = ScaffoldMessenger.of(context);
    return Scaffold(
      appBar: AppBar(
        title: const Text('🍹 Flutter Mixin Sample 🍹'),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            // ボタン ①
            ElevatedButton(
              child: const Text('1 のボタン'),

              // ボタンタップのイベントハンドラー
              onPressed: () async {
                try {
                  // 画面上にローディングアニメーションを表示する
                  ref.watch(overlayLoadingProvider.notifier).state = true;

                  // ボタンタップ時に実行したい主となる処理を呼び出す
                  // ※ 外部サービスへの通信が発生する想定
                  await Utils.instance.function1();

                  // 成功時のメッセージをスナックバーに表示する
                  Utils.instance.showSnackBar(
                    scaffoldMessenger,
                    mode: SnackBarMode.success,
                    message: '1 のボタンがタップされました',
                  );
                } catch (e) {
                  // 例外が発生した場合はメッセージをスナックバーに表示する
                  Utils.instance.showSnackBar(
                    scaffoldMessenger,
                    mode: SnackBarMode.failure,
                    message: 'failure: ${e.toString()}',
                  );
                } finally {
                  // 画面上に表示されているローディングアニメーションを停止する
                  ref.watch(overlayLoadingProvider.notifier).state = false;
                }
              },
            ),

            // ボタン ②
            // ※ 処理手順はボタン ① と同じ
            ElevatedButton(
              child: const Text('2 のボタン'),
              onPressed: () async {
                try {
                  ref.watch(overlayLoadingProvider.notifier).state = true;
                  await Utils.instance.function2();
                  Utils.instance.showSnackBar(
                    scaffoldMessenger,
                    mode: SnackBarMode.success,
                    message: '2 のボタンがタップされました',
                  );
                } catch (e) {
                  Utils.instance.showSnackBar(
                    scaffoldMessenger,
                    mode: SnackBarMode.failure,
                    message: 'failure: ${e.toString()}',
                  );
                } finally {
                  ref.watch(overlayLoadingProvider.notifier).state = false;
                }
              },
            ),

            // ボタン ③
            // ※ 処理手順はボタン ① と同じ
            ElevatedButton(
              child: const Text('3 のボタン'),
              onPressed: () async {
                try {
                  ref.watch(overlayLoadingProvider.notifier).state = true;
                  await Utils.instance.function3();
                  Utils.instance.showSnackBar(
                    scaffoldMessenger,
                    mode: SnackBarMode.success,
                    message: '3 のボタンがタップされました',
                  );
                } catch (e) {
                  Utils.instance.showSnackBar(
                    scaffoldMessenger,
                    mode: SnackBarMode.failure,
                    message: 'failure: ${e.toString()}',
                  );
                } finally {
                  ref.watch(overlayLoadingProvider.notifier).state = false;
                }
              },
            ),
          ],
        ),
      ),
    );
  }
}

```

### 問題点

問題点は、冗長なコードが数多く記載されているという点です。

具体的には次の処理が挙げられます。

- 画面上にローディングアニメーションを表示する
- 成功時にメッセージをスナックバーに表示する
- 例外が発生した場合はメッセージをスナックバーに表示する
- 画面上に表示されているローディングアニメーションを停止する

これらの処理を１箇所にまとめて、ボタンタップ時には本当に実行したい処理だけを呼び出すようにする。
これが再利用性の高いコードであり理想的ですよね。

## Mixin を使用したコードにリファクタリングしてみる

ということで、ここまで説明してきた冗長なコードを Mixin を使用して再利用性の高いコードにリファクタリングしていきます。

### 冗長な箇所を Mixin として抜き出す

上記問題点で述べた冗長なコードを Mixin として抜き出します。
すると、次のようなコードになります。

画面関連のクラスから呼び出される Mixin として `PageMixin` という名前に定義しています。

```dart
/// 画面側での共通的な処理をラップした Mixin
/// in_1: BuildContext
/// in_2: WidgetRef
/// in_3: 画面側のイベントハンドラーとして呼び出したい処理
/// in_4: 処理成功時に表示したいメッセージ
mixin PageMixin {
  Future<void> execute(
    BuildContext context,
    WidgetRef ref, {
    required Future<void> Function() action,
    required String successMessage,
  }) async {
    final scaffoldMessenger = ScaffoldMessenger.of(context);
    try {
      // 画面上にローディングアニメーションを表示する
      ref.watch(overlayLoadingProvider.notifier).state = true;

      // 引数でもらった処理を実行する
      await action();

      // 成功時のメッセージをスナックバーに表示する
      Utils.instance.showSnackBar(
        scaffoldMessenger,
        message: successMessage,
        mode: SnackBarMode.success,
      );
    } catch (e) {
      // 例外が発生した場合はメッセージをスナックバーに表示する
      Utils.instance.showSnackBar(
        scaffoldMessenger,
        message: 'failure: ${e.toString()}',
        mode: SnackBarMode.failure,
      );
    } finally {
      // 画面上に表示されているローディングアニメーションを停止する
      ref.watch(overlayLoadingProvider.notifier).state = false;
    }
  }
}
```

#### ポイント

- ボタンタップ時に実行したい主となる処理を呼び出し元から渡してもらう
- 成功時にスナックバーに表示したメッセージを呼び出し元から渡してもらう
- それ以外の処理に関しては呼び出す側、つまり、Page 側からは気にする必要がない

### 定義した Mixin を実装

上記で定義した `PageMixin` を Page 側に実装します。
すると、次のようなコードになります。
Mixin を使用しない場合に比べて大幅なコード短縮となったことがわかるかと思います。

```dart
/// ホームページ
class HomePage extends ConsumerWidget with PageMixin {
  const HomePage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('🍹 Flutter Mixin Sample 🍹'),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            // ボタン ①
            ElevatedButton(
              child: const Text('1 のボタン'),
              onPressed: () async {
                execute(
                  context,
                  ref,
                  action: () async => await Utils.instance.function1(),
                  successMessage: '1 のボタンがタップされました',
                );
              },
            ),

            // ボタン ②
            ElevatedButton(
              child: const Text('2 のボタン'),
              onPressed: () async {
                execute(
                  context,
                  ref,
                  action: () async => await Utils.instance.function2(),
                  successMessage: '2 のボタンがタップされました',
                );
              },
            ),

            // ボタン ③
            ElevatedButton(
              child: const Text('3 のボタン'),
              onPressed: () async {
                execute(
                  context,
                  ref,
                  action: () async => await Utils.instance.function3(),
                  successMessage: '3 のボタンがタップされました',
                );
              },
            ),
          ],
        ),
      ),
    );
  }
}
```

#### ポイント

- PageMixin を with 句 で実装する
- ボタンタップ時のイベントハンドラーとして PageMixin の `execute` を呼び出す
  - この時に、`PageMixin.execute` のような呼び出しは不要、
    `execute` だけで Mixin 内のメソッドを呼び出すことができる

## さいごに（単純な継承（extends）や utility クラスでもよいのでは？）

ここまで Mixin で冗長なコードを抜き出して再利用性の高いコードを実現するための方法を説明してきましたが、
これらの内容は extends や utility クラスでも実現できるのでは？
という疑問も浮かんでくるかと思います。

結論から言うと、yes となります。

しかし、今回のような Page に関するクラスで同じようなことを実現しようとすると、
Flutter においては、StatefulWidget、StatelessWidget、ConsumerWidget 等を継承する必要があるため、
他のクラスを継承したいことがあった場合、extends 句による多重継承は許されない仕様となっています。

そのような場合に、Mixin を使用することで多重継承を実現することができます。

さらに、utility クラスと比較した場合には、**Mixin を用いることによるメソッド呼び出しの単純さ** がメリットに感じています。
utility クラスを使用したメソッド呼び出しの場合、`utility.hoge` のような呼び出し方法になります。
一方、Mixin を使用したメソッド呼び出しの場合、`hoge` のような呼び出しで済みます。

以上、再利用性の高いコードを実現するための 1 つの方法として Mixin を使用する方法を紹介しました。

## サンプルコード

今回の全コードを GitHub にて公開しています。
https://github.com/nozomi-koborinai/flutter-mixin-sample
