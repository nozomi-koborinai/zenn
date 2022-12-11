---
title: "【Flutter × GoFデザインパターン】Commandを実装してみた"
emoji: "🕹️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter", "Dart", "デザインパターン", "command"]
published: true # true：公開記事、false：非公開記事
published_at: 2022-12-22
---
本記事はFlutter大学アドベントカレンダー22日目の記事です。
https://qiita.com/advent-calendar/2022/flutteruniv
はじめましてcoboといいます。
私はソフトウェアの設計理論を学ぶことに興味があるため、今回はGoFデザインパターンの中からCommandパターンを紹介させていただきます。

# 本記事のゴール
- Commandパターンが理解できる
- Commandパターンを実装してサンプルアプリを作成することができる

## 説明すること/しないこと
### すること
- デザインパターンの概要
- Commandパターンの概要
- Commandパターンの実装方法
### しないこと
- デザインパターンの種類とそれらの説明
- 状態管理(Stateless, Stateful, Riverpodなど)

それでは本題に入っていきます。

# Commandパターンとは?
`処理の呼び出し（命令）と、処理の内容を分離するデザインパターン`のことです。
## そもそもデザインパターンとは？
過去のソフトウェア設計者が発見し編み出した設計ノウハウを蓄積し、名前をつけ、再利用しやすいように特定の規約に従ってカタログ化したものです。
ソフトウェア開発における`設計カタログ`だと思っていただければイメージがつきやすいかと思います。
今回説明するCommandパターンは数多くある設計カタログ中の1種類ということになります。

# 実装
今回はCommandパターンを実装して以下のサンプルアプリを作成していきます。
![](/images/command_sample.gif =450x)

## Commandパターンの概要
![](/images/command_pattern_class1.png =550x)
### それぞれのクラスの役割
| クラス | 役割 |
| ---- | ---- |
| Command<interface> | 操作を実行するためのインタフェースを宣言 |
| Command | Command<interface>の具象 |
| Receiver | Commandの受取手、Command処理の対象 |
| Invoker | 画面上の操作をトリガーとしてCommand処理を実行 |
| Client | Commandの生成 及び Receiverの設定 |
| CommandHistory | 実行したCommandの履歴管理 |

### Command<interface>
インタフェースとして、抽象的なコマンドを定義します。
```dart:command.dart
/// コマンドを抽象的に定義
abstract class Command {
  /// コマンドの実行
  void execute();

  /// コマンド名の取得
  String getTitle();

  /// コマンドの実行を元に戻す
  void undo();
}
```

### Command
抽象として定義したCommandを具象として定義します。
Commandインタフェースを実装するところがポイントです。
```dart:change_color_command.dart
/// 色変更コマンド
///
/// Commandインタフェースを実装して具体的な処理を記述する
class ChangeColorCommand implements Command {
  // 色変更対象のShapeオブジェクト
  Shape shape;
  // 変更前の色
  late Color previousColor;
  final random = math.Random();

  /// コンストラクタ
  ///
  /// 引数にてClientからReceiverを受け取る
  /// そのReceiverに対して変更を加えていくイメージ
  /// 今回の例で言うとReceiverはShapeオブジェクトとなる
  ChangeColorCommand(this.shape) {
    previousColor = shape.color;
  }

  /// コマンド処理の実体
  /// -> shapeにランダムな色を設定する
  @override
  void execute() {
    shape.color = Color.fromRGBO(
      random.nextInt(255),
      random.nextInt(255),
      random.nextInt(255),
      1.0,
    );
  }

  @override
  String getTitle() {
    return 'Change color';
  }

  /// 加えた変更を無かったことにする処理
  @override
  void undo() {
    shape.color = previousColor;
  }
}
```
:::message
本記事では詳しくは説明しませんが、
具象が抽象に依存することを`依存関係逆転の原則`といいます。
この原則を活用することで一般的によいとされている`疎結合なプログラム`を設計することができます。
:::

### Receiver
Command処理の対象となるオブジェクトを定義します。
上記CommandクラスのコンストラクタにReceiver（Shape）を渡すことで、Receiverに対して様々な変更を加えることが可能になります。
```dart:shape.dart
/// Commandの受取手、Command処理の対象オブジェクト
class Shape {
  late Color color;
  late double height;
  late double width;

  Shape.initial() {
    color = Colors.black;
    height = 150.0;
    width = 150.0;
  }
}
```

### Invoker, Client
#### Invoker
Client(画面)上の操作をトリガーとしてCommand処理を呼び出します。
今回は以下のようにViewの中にInvoker用のメソッドを定義しています。
#### Client
Commandの生成(インスタンス化)、Receiverの設定、画面上の操作によってInvokerを呼び出します。
```dart:command_example_page.dart
/// Commandサンプルアプリページ
class CommandExamplePage extends StatefulWidget {
  const CommandExamplePage({super.key});

  @override
  _CommandExamplePageState createState() => _CommandExamplePageState();
}

class _CommandExamplePageState extends State<CommandExamplePage> {
  // undoを実現するためのCommand履歴
  final CommandHistory _commandHistory = CommandHistory();
  // Receiver(shape)の初期設定
  final Shape _shape = Shape.initial();

  /// Invoker(色変更ボタン押下時に呼ばれるメソッド)
  void _changeColor() {
    // CommandのReceiverとして_shapeを渡す
    final command = ChangeColorCommand(_shape);
    // 実行したコマンドをコマンド履歴に追加
    _executeCommand(command);
  }

  /// Invoker(高さ変更ボタン押下時に呼ばれるメソッド)
  void _changeHeight() {
    // CommandのReceiverとして_shapeを渡す
    final command = ChangeHeightCommand(_shape);
    // 実行したコマンドをコマンド履歴に追加
    _executeCommand(command);
  }

  /// Invoker(幅変更ボタン押下時に呼ばれるメソッド)
  void _changeWidth() {
    // CommandのReceiverとして_shapeを渡す
    final command = ChangeWidthCommand(_shape);
    // 実行したコマンドをコマンド履歴に追加
    _executeCommand(command);
  }

  void _executeCommand(Command command) {
    setState(() {
      command.execute();
      _commandHistory.add(command);
    });
  }

  void _undo() {
    if (_commandHistory.isEmpty) return;
    setState(() {
      _commandHistory.undo();
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(
          'Command Design Pattern',
          style: TextStyle(color: Colors.grey[700]),
        ),
      ),
      body: SingleChildScrollView(
        padding: const EdgeInsets.symmetric(
          horizontal: 10,
        ),
        child: Column(
          children: <Widget>[
            SizedBox(
              height: 160.0,
              child: Center(
                child: AnimatedContainer(
                  duration: const Duration(milliseconds: 500),
                  height: _shape.height,
                  width: _shape.width,
                  decoration: BoxDecoration(
                    color: _shape.color,
                    borderRadius: BorderRadius.circular(10.0),
                  ),
                  child: const Icon(
                    Icons.star,
                    color: Colors.white,
                  ),
                ),
              ),
            ),
            const SizedBox(height: 15),
            ElevatedButton(
              onPressed: _changeColor,
              child: const Text('Change color command'),
            ),
            const SizedBox(height: 5),
            ElevatedButton(
              onPressed: _changeHeight,
              child: const Text('Change height command'),
            ),
            const SizedBox(height: 5),
            ElevatedButton(
              onPressed: _changeWidth,
              child: const Text('Change width command'),
            ),
            const Divider(),
            ElevatedButton(
              onPressed: _undo,
              child: const Text('Undo'),
            ),
            const SizedBox(height: 15),
            CommandHistoryColumn(
              commandList: _commandHistory.commandHistoryList,
            ),
          ],
        ),
      ),
    );
  }
}
```

### CommandHistory
実行したCommandを履歴管理するためのオブジェクトを定義します。
これにより、undo/redoなどの機能が実現可能となります。
```dart:command_history.dart
/// 実行したCommandを履歴管理するためのオブジェクト
class CommandHistory {
  final List<Command> _commandList = <Command>[];

  bool get isEmpty => _commandList.isEmpty;

  /// 実行したコマンド名を文字列のリストで返却
  List<String> get commandHistoryList =>
      _commandList.map((c) => c.getTitle()).toList();

  void add(Command command) {
    _commandList.add(command);
  }

  /// Command履歴リストの末尾要素を削除
  /// Command履歴リストの末尾要素のundo処理を実行して一つ前の状態に戻す
  void undo() {
    if (_commandList.isNotEmpty) {
      final command = _commandList.removeLast();
      command.undo();
    }
  }
}
```

## 新たに機能を追加してみる
現時点でのサンプルアプリの機能は
- 色変更
- 高さ変更
- 幅変更

の3種類となっており、それぞれの操作がCommandとして定義されています。
これらに対してさらに`アイコン変更`Commandを追加してみることにします。

:::details Shapeオブジェクトに新規プロパティの追加
アイコンを変更したいので、まずはShapeオブジェクトにiconプロパティを新たに定義します。
```dart:shape.dart
/// Commandの受取手、Command処理の対象オブジェクト
class Shape {
  late Color color;
  late double height;
  late double width;
  late IconData icon;   // 追加

  Shape.initial() {
    color = Colors.black;
    height = 150.0;
    width = 150.0;
    icon = Icons.star;  // 追加、デフォルトは星アイコン
  }
}
```
:::

:::details アイコン変更Commandを新規追加
色変更コマンド同様にCommandインタフェースを実装したアイコン変更コマンドを新たに定義します。
```dart:change_icon_command.dart
/// アイコン変更コマンド
class ChangeIconCommand implements Command {
  // アイコン変更対象のShapeオブジェクト
  Shape shape;
  // 変更前のアイコン
  late IconData previousIcon;

  ChangeIconCommand(this.shape) {
    previousIcon = shape.icon;
  }

  @override
  void execute() {
    shape.icon = Icons.star_border;
  }

  @override
  String getTitle() {
    return 'Change icon';
  }

  @override
  void undo() {
    shape.icon = previousIcon;
  }
}
```
:::

:::details アイコン変更CommandInvokerを新規追加
こちらも色変更コマンドのInvokerと同様に新たにアイコン変更CommandInvokerを定義します。
```dart:command_example_page.dart
一部抜粋
  /// Invoker(アイコン変更ボタン押下時に呼ばれるメソッド)
  void _changeIcon() {
    // CommandのReceiverとして_shapeを渡す
    final command = ChangeIconCommand(_shape);
    // 実行したコマンドをコマンド履歴に追加
    _executeCommand(command);
  }
```
:::

:::details 画面のアイコン変更ボタンによりInvokerを呼び出す
ユーザの新規操作としてアイコン変更を実現するために、新たにアイコン変更ボタンを定義します。
```dart:command_example_page.dart
一部抜粋
            ElevatedButton(
              onPressed: _changeColor,
              child: const Text('Change color command'),
            ),
            const SizedBox(height: 5),
            ElevatedButton(
              onPressed: _changeHeight,
              child: const Text('Change height command'),
            ),
            const SizedBox(height: 5),
            ElevatedButton(
              onPressed: _changeWidth,
              child: const Text('Change width command'),
            ),

            // 追加 ここから
            const SizedBox(height: 5),
            ElevatedButton(
              onPressed: _changeIcon,
              child: const Text('Change icon command'),
            ),
            // 追加 ここまで
            const Divider(),
            ElevatedButton(
              onPressed: _undo,
              child: const Text('Undo'),
            ),
```
:::

### 完成！！
これでアイコン変更コマンドを実装したサンプルアプリが出来上がりました。
お疲れ様でした！
![](/images/command_sample_update.gif =450x)


# FlutterでCommandパターンを実装してみての感想
今回の記事で初めてGoFデザインパターンのCommandをFlutterで実装しました。
ユーザ操作をCommandとして定義することで、

- 操作履歴を管理することが容易になる
- 新たなユーザ操作を新規Commandとして定義することで、既存処理の影響少なくしたままコードを書いてける

といった点が魅力的で、コードを書いていてとても楽しかったです。
また、デザインパターンを理解することで、言語に依存しない設計知識を身につけることができるため、今後も定期的にデザインパターンを学習/紹介していきたいと思いました。

# 参考
https://kazlauskas.dev/flutter-design-patterns-12-command/
https://www.hanachiru-blog.com/entry/2021/03/15/120000
https://shiraberu.tech/2021/11/26/command-pattern/