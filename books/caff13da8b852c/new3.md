---
title: "Day3: Flutterの概要, Widget, State"
free: false
---

# 流れと目標

目標: Flutterを使って1からカウンターアプリを実装する

## 今日の流れ

1. VSCodeでプロジェクトを開く
   - VSCodeでのFlutterのプロジェクトの見方を学ぶ
2. Hello, world!の説明
   - 最初に実行したプログラムの概要を学ぶ
3. 全てはWidgetである
4. 木構造とWidgetツリー
   - Flutterで画面を作るときの概念を理解する
5. 各Widgetについて
6. StatefulWidgetとsetState()とメンバ変数
   - 1からコードを書いて画面の作り方を理解する

# VSCodeでプロジェクトを開く

まずはVSCodeで作ったプロジェクトを開いてコードを見る・編集できるようにしよう

1. VSCodeを起動
2. 左上の[File] -> [Open Folder]から前回作ったフォルダ(flutter_aic)を開く
   1. 日本語だとまた変わるかも
   2. 信頼しますか？とか言われたらとりあえず信頼すること

![](/images/2022-05-20-14-48-03.png)

![](/images/2022-05-20-14-51-07.png)

3. 左側に表示されているファイル一覧から[lib] -> [main.dart]をクリックして表示

![](/images/2022-05-20-14-53-37.png)

4. 左上の[View] -> [Terminal]を押してターミナルを表示

![](/images/2022-05-20-14-55-51.png)

5. `flutter run -d chrome`か`flutter run -d edge`と書いてEnterを押すと実行できる
   1. 終了する場合はブラウザを閉じるかターミナル内で`q`を押す

![](/images/2022-05-20-14-57-47.png)

# Hello, world!の説明

大雑把に(// から始まる行はコメントなので省略)

![](/images/2022-05-28-15-20-08.png)

![](/images/2022-05-28-15-20-57.png)

1. main関数
2. runApp()
3. MyApp.build()
4. MyHomePage.createState()
5. MyHomePageState.build()

のような順番で呼ばれていく

## ファイル構成

![](/images/2022-05-28-15-27-31.png)

dartのコードはlib/に書く

web/にあるものはindex.htmlなど普通のWebサイトに必要な物たち

android/ ios/などはそれぞれのプラットフォーム専用のコードが書かれている。Flutterでは固有の呼び出しをうまく隠蔽してくれているためほとんど弄る必要がない。

### [余談]その他のファイルについて

- .dart_tool/ : Dart言語のプロジェクトでの設定など
- .idea/ : IDE(統合開発環境)を使うときの設定
- .gitignore : Gitでバージョン管理の対象から外すファイルを指定
- .metadata : Flutterプロジェクトを更新するときに使われる情報
- .packages : ライブラリのコードの保存場所を示す
  - .から始まるファイル・フォルダは隠しファイル・フォルダと呼ばれる。だいたいプログラムが勝手に作って勝手に使うので弄ってはいけない
- analysis_options.yaml: プログラムのミスなどを指摘してくれるlinterというツールの設定ファイル
- [プロジェクト名].iml: IDE(統合開発環境)を使うときの設定
- pubspeck.lock: pubspec.yamlで追加したライブラリを詳細に管理するためのファイル(自動生成)
- README.md: プログラムの説明などを書くためのファイル

## [余談]Webはどのように動くのか

デバッグ時は特殊なので置いておくとして、実際にWeb公開したい場合には`flutter build web`というコマンドを実行することでWebアプリとしてまとめてくれる。

![](/images/2022-05-20-15-00-55.png)

頑張って書いたDartのコードはmain.dart.jsというファイルにJavaScriptに変換されている。ユーザがWebアプリにアクセスすると、

1. index.htmlにアクセス
2. flutter.jsというスクリプトを読み込み
3. flutter.js内からmain.dart.jsを呼び出し
4. JavaScriptを使ってindex.htmlから作られたDOMを書き換える
   1. これでFlutterの画面が表示、更新される

という流れになっている。main.dart.jsの中身を人間に読むのは不可能。

# 全てはWidgetである

Flutterの画面を構成する全ての部品はWidgetというパーツである。Widget同士は可換であるため、自由なレイアウトが実現できる。



## 下のページはCardというWidgetを並べてできている

![](https://www.mediafire.com/convkey/253d/cvboy07jv9589y96g.jpg)



## Cardの中に配置されている画像や文字、ボタンはWidgetである

![](https://www.mediafire.com/convkey/dd45/gepccwdzumiwe5n6g.jpg)



## マクロに見るとCardたちのレイアウトもGrid Widgetであり、さらに画面全体もScaffold Widgetでできている

![](https://www.mediafire.com/convkey/2f97/62k8lm0411lncun6g.jpg)

> Flutterの呼び出し自体もMaterialAppというWidgetだったりする



## これはCard Widget

![](https://www.mediafire.com/convkey/7864/zkkg2znmqh6ys7h6g.jpg)



## 右下に配置するのにAlign Widgetで囲んでいる

![](https://www.mediafire.com/convkey/2152/kviie1wrkrbjs296g.jpg)



## 端から少し余白をとるのにもWidgetでMarginをとる

![](https://www.mediafire.com/convkey/9786/35kfqmc5yttmoyb6g.jpg)



## 中に入っているアイコンや文字もWidgetである

![](https://www.mediafire.com/convkey/485d/x927ombjcy7jc4t6g.jpg)

このようにFlutterの世界では文字や画像からレイアウトや余白まで全てがWidgetでできている

もちろん自分で新しいWidgetを作ることも可能



# 木構造とWidgetツリー

![](https://docs.flutter.dev/assets/images/docs/ui/layout/sample-flutter-layout.png)

> https://docs.flutter.dev/development/ui/layout より

前節で画面がWidgetで埋め尽くされていることがわかった。これらのWidgetたちはどのような形で与えられ、管理されて画面を構成するのだろうか。

サンプルアプリのコードと実行したときの画面を解析してみると、以下のような構造になっている。

![](/images/2022-05-20-22-50-41.png)

画面の方は複数の層に分かれた四角形が積み木のように積み上がっていることがわかる。コードの方もインデントに注目すると、同様に積み上がっている。

> インデント: コードの可読性のために行の初めが段々になるよう空白を入れること

もう少し注目すると、積み上がった各Widgetには親子関係を見出すことが可能である。親は自身を積み上げるときの土台となるWidgetで、子は自身の上に積み重なるものとなる(生物の親子とは少し異なり親は高々1個で、子は0個以上持つことができる)。
このような親子関係で表せるデータは、**木構造**というデータ構造で表現することが可能である。

木構造でサンプルアプリのWidgetの関係を表現すると以下のようになる。

![](/images/2022-05-20-23-07-31.png)

まず木構造の木は現実のそれとは異なり、根が上部に存在して下に成長する。このツリーの根はMaterialAppである(コードの上の方で定義している)。根を親として先程の親子関係が木構造で表現できていることがわかるだろう。

このようにGUIは、構成要素を木構造で表現することで複雑な画面を作っている。

Flutterでは、開発者がWidgetでWidgetツリーを作って画面をデザインする。

> ちなみにFlutter内ではWidgetツリーだけでなく様々な用途に木構造を使っている。例えばタッチイベントを処理するときにも使われている。

# 各Widgetについて

- [Widget Catalog(英語)](https://flutter.dev/docs/development/ui/widgets)
- https://iotmanablog.hatenablog.com/entry/2021/03/09/063111
- https://minpro.net/widget-05

使う雛形(これでmain.dartの中身を入れ替える)

```Dart
// ignore_for_file: library_private_types_in_public_api, use_key_in_widget_constructors, prefer_const_constructors, avoid_print

import 'package:flutter/material.dart';

void main() => runApp(MyApp());

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Web Training',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: MyHomePage(),
    );
  }
}

class MyHomePage extends StatefulWidget {
  const MyHomePage({Key? key}) : super(key: key);

  @override
  _MyHomePageState createState() => _MyHomePageState();
}

class _MyHomePageState extends State<MyHomePage> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Text("Hello, Flutter!",
          style: TextStyle(fontSize: 32.0, color: Colors.green)),
    );
  }
}

```

## Text

```Dart
Text(
	"Hello, Flutter!",
	style: TextStyle(
		fontSize: 32.0,
		color: Colors.green
	)
),
```

https://dartpad.dev/?id=4bc52c2fd6cce009d4df1333ea3df7ec

最初に表示するテキストを引数に取る。

文字の装飾をしたい場合は名前付き引数のstyleにTextStyleを渡す

[TextStyle class - Flutter](https://api.flutter.dev/flutter/painting/TextStyle-class.html)

## Button

```Dart
ElevatedButton(
  onPressed: () { // ボタンが押されたときに呼ばれる部分
    print("Button Clicked!");
  },
  // ボタンに子を指定するとその中に表示される
  child: Text("Hello, Flutter!"),
),
```

https://dartpad.dev/?id=a4ca7fca7a138adbd6fdcc0e3443468a

child引数に中に入れたいWidgetを書く

onPressed引数に、押されたときの処理を書く

* ElevatedButton: 立体的な色付きのボタン
* TextButton: 文字だけのボタン
* OutlinedButton: 外枠のみのボタン

## Center

```Dart
Center(
	child: ElevatedButton(
		//...
	)
)
```

https://dartpad.dev/?id=1bfc64c7843cfb6f3c47a3dc6047a06d

child引数に取ったWidgetを中心にくるようにする

## Container

```Dart
Container(
    width: 300.0,
    height: 200.0,
    color: Colors.orange,
    margin: EdgeInsets.all(8.0),
    padding: EdgeInsets.all(8.0),
    child: RaisedButton(
        onPressed: () {
            print("Button Clicked!");
        },
        child: Text(
            "Hello, Flutter!",
            style: TextStyle(
                fontSize: 32.0,
                color: Colors.green
            ),
        ),
    ),
),
```

https://dartpad.dev/?id=9a96f2decc523387fa2eca048e508d49

中のWidgetの大きさ・背景色・余白等を管理する

![](https://docs.flutter.dev/assets/images/docs/ui/layout/margin-padding-border.png)

[レイアウトの話](https://docs.flutter.dev/development/ui/layout)

# StatefulWidgetとsetState()とメンバ変数

Hello, worldでは詳しく触れなかったが、Flutterは画面更新の仕組みの一つとしてStatefulWidgetというものを用意している。

```Dart
class MyHomePage extends StatefulWidget {
```

extendsはこのクラスがStatefulWidgetの役割を持つことを定めている(継承)。

StatefulWidgetはその名の通りState(状態)を持つWidgetである。

> アプリ界隈の人とかはライフサイクル管理を知っているだろうが、FlutterではStatefulWidgetでライフサイクルを管理する
> 
> ライフサイクル: 画面の要素(Widget)が作られてから破棄されるまでの流れ

## setStateの呼び出しと画面更新

StatefulWidgetは自分の子となるWidgetたちを任意のタイミングで再レイアウト・再描画する機能を持っている。開発者はsetState()という関数を呼ぶことで画面の表示を更新することができる。

```Dart
setState(() {
	title = "タイトルを変更！";
})
```

setStateを呼んだときに画面を表示するときに使われる情報の一部を書き換えておくと、画面もその情報を使って書き換わる。

例

```Dart
class _MyHomePageState extends State<MyHomePage> {
  int count = 0; // 追加

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: Container(
          width: 400.0,
          height: 100.0,
          color: Colors.blue.shade50,
          child: Center(
            child: ElevatedButton(
              onPressed: () {
                print("Button Clicked!");
                // 追加
                setState(() {
                  count++;
                });
              },
              child: Text("You clicked $count times!"), //変更
            ),
          ),
        ),
      ),
    );
  }
}
```

https://dartpad.dev/?id=8ff92d28843bb234e4c3c058b9c4e706

便利

## 何が起こっているのか

1. 変数を書き換える
2. setStateを呼ぶ
3. 呼ばれたStatefulWidgetは変数の変更を知る
4. build関数を実行して子を再描画する

いちいち子を作り直していては重いのでは？と思うかもしれないが、ここがFlutterの真骨頂。よしなに更新するべきところだけ更新してくれる。

詳しく知りたい人は「[FlutterのWidgetツリーの裏側で起こっていること](https://medium.com/flutter-jp/dive-into-flutter-4add38741d07)」を読む(別に知らなくてもアプリは作れる)

# おしまい

StatefulWidgetを適当に使うだけでも画面をよしなに変更できるので、簡単なアプリならもう作れる。レッツチャレンジ

例: どうぶつしょうぎ(拙作): 500行くらいで作った

https://doubutsu.fastriver.dev/#/

## 課題(物足りない人向け)

* 他のWidgetも使いながらカウンタをアレンジしてみる
* ボタンを並べてキーボードみたいなものを作ってみる
