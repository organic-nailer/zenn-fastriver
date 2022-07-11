---
title: "Day4: レイアウト, 画面遷移(Navigator)"
free: false
---

# 流れと目標

目標: 画面の自由な位置にWidgetを配置できるようになる

## 今日の流れ

1. 画面のレイアウト
   - Widgetのレイアウトを学ぶ
2. 画面遷移
   - 複数の画面に移動できるようになる

## 使うテンプレートコード

今回使うテンプレートは前回と同様(main.dartにコピペしてね)

```dart
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
        body: Container()
    );
  }
}
```

# 画面のレイアウト

![](/images/2022-05-28-17-01-29.png)

レイアウトとは、表示要素(Widgetなど)を画面上に配置することを指す。Webサイトでもアプリでもゲームでも、見た目や使いやすさに直結するため非常に重要かつ難しい問題である。

FlutterではButtonやTextなどの何かを表示するWidgetとは別にCenterなどの子をどの位置に置くかを決める専用のWidgetが用意されている。

レイアウトでは大体以下のようなことを行っている。

- 並べる
  - 縦に並べたり、横に並べたり、上に積んだり
- 端から距離を空ける
  - 上下左右で何ピクセルの空白を空けるか
- 大きさを合わせる
  - 親の幅と同じになったり、余っている場所を埋めるように変形したり

## Widgetを並べる(Row, Column)

Row, ColumnというWidgetを使うことで複数のWidgetを横並びにしたり縦並びにしたりすることができる。

### 横に並べる(Row)

Rowのchildrenに子をリスト形式で渡すとそれらを横に並べて表示してくれる。

```dart
class _MyHomePageState extends State<MyHomePage> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
        body: Row(
      children: [
        Container(
          color: Colors.green,
          height: 200.0,
          width: 200.0,
        ),
        Container(
          color: Colors.yellow,
          height: 300.0,
          width: 200.0,
        ),
        Container(
          color: Colors.red,
          height: 100.0,
          width: 300.0,
        ),
      ],
    ));
  }
}
```

https://dartpad.dev/?id=12808c8f5fb02b5cc4314b6c037bc2d4

![](/images/2022-05-28-17-13-02.png)

並べ方の方法を指定することもできる。

- MainAxisAlignment: Start or End or Center or SpaceBetween or SpaceAround or SpaceEvenly
  - 主軸(並べる方向)で子をどのように寄せるか
- MainAxisSize: Max or Min
  - 主軸側の自身のサイズの決定方法
- CrossAxisAlignment: Start or Center or End or Stretch or Baseline
  - 交差軸(主軸と直交)で子をどのように寄せるか
- VerticalDirection: Up or Down
  - 主軸の並べる向き

```dart
        body: Row(
      mainAxisAlignment: MainAxisAlignment.spaceEvenly,
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
```

![](/images/2022-05-28-17-29-21.png)

> 幅が足りないとエラーが出る
> ![](/images/2022-05-28-17-34-44.png)

### 縦に並べる(Column)

Rowと同じ要領でColumnに置き換えると、並べる方向が縦に変化する。使い方は同じ。

```dart
class _MyHomePageState extends State<MyHomePage> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
        body: Column(
      children: [
        Container(
          color: Colors.green,
          height: 200.0,
          width: 200.0,
        ),
        Container(
          color: Colors.yellow,
          height: 300.0,
          width: 200.0,
        ),
        Container(
          color: Colors.red,
          height: 100.0,
          width: 300.0,
        ),
      ],
    ));
  }
}
```

https://dartpad.dev/?id=ef53d1326d9d22c22e83e6d37dc2bf7e

![](/images/2022-05-28-17-30-55.png)

### 余った部分を埋める(Expanded)

A,B,CというWidgetを横に並べるとする。AとBは幅が決まっているが、余った部分はCで埋めてしまいたい。このようなときはCをExpandedというWidgetで囲うとCを最大幅に広げて無駄な空間をなくしてくれる。

```dart
class _MyHomePageState extends State<MyHomePage> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
        body: Row(
      children: [
        Container(
          color: Colors.green,
          height: 100.0,
          width: 100.0,
        ),
        Container(
          color: Colors.yellow,
          height: 100.0,
          width: 100.0,
        ),
        Expanded( // 追加！
          child: Container(
            color: Colors.red,
            height: 100.0, // 幅は指定しないようにしておく
          ),
        ),
      ],
    ));
  }
}
```

https://dartpad.dev/?id=cffc2bdeb854ff9a9e1cec503a35a217

![](/images/2022-05-28-17-41-18.png)

## 自由な位置に配置する・重ねる(Stack)

子を自分の好きな位置に表示したい場合、また重ねて表示させたい場合はStackというWidgetが使える。

StackはRow/Columnと同様にchildrenを持つことのできるWidgetで、AlignやPositionedというWidgetを子として追加することで自由な位置に子を置くことができる。

Align Widgetはalignmentというものを引数に取り、親からの相対的な位置(左上や右下など)を指定できる。また、子はリストの後ろにある方が手前に表示される。

> 後ろの子が手前に表示されるのは、前のWidgetから順に画面に描いているから

```dart
class _MyHomePageState extends State<MyHomePage> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
        body: Center(
      child: Container(
        width: 300.0,
        height: 300.0,
        color: Colors.grey,
        child: Stack(
          children: [
            Align(
              alignment: Alignment.topLeft,
              child: Container(
                color: Colors.green,
                height: 200.0,
                width: 200.0,
              ),
            ),
            Align(
              alignment: Alignment.center,
              child: Container(
                color: Colors.yellow,
                height: 200.0,
                width: 200.0,
              ),
            ),
            Align(
              alignment: Alignment.bottomRight,
              child: Container(
                color: Colors.red,
                height: 200.0,
                width: 200.0,
              ),
            ),
          ],
        ),
      ),
    ));
  }
}
```

https://dartpad.dev/?id=28de3edb59b00ad86d67c1c85fb3d039

![](/images/2022-05-28-20-03-01.png)

Alignの代わりにPositionedを使うと、より細かく位置を設定することができる。

Positionedはleft,top,right,bottomという属性を持っていて、指定したピクセルだけ親の境界からずらして表示される。

これを利用すると、例えば格子状にWidgetを表示できたりする。

```dart
class _MyHomePageState extends State<MyHomePage> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
        body: Center(
      child: Container(
        width: 300.0,
        height: 300.0,
        child: Stack(
          children: [
            Positioned(
              top: 0.0,
              left: 0.0,
              child: Container(
                color: Colors.green.withOpacity(0.1),
                height: 100.0,
                width: 100.0,
              ),
            ),
            ...
            Positioned(
              top: 200.0,
              left: 200.0,
              child: Container(
                color: Colors.green.withOpacity(0.9),
                height: 100.0,
                width: 100.0,
              ),
            ),
          ],
        ),
      ),
    ));
  }
}
```

https://dartpad.dev/?id=e90109f070f77e16b37ae0d37a198e46

![](/images/2022-05-28-20-10-18.png)

### PositionedとAlignの違い

- Positioned: 親の境界からの距離をピクセルで指定
- Align: 親からの相対的な位置を指定

## 余白を指定する(Padding)

前の3つは複数の子をどのように配置するのかというWidgetだったが、前回やったCenterのように1つの子に対して配置を適用するWidgetもいくつかある。

Paddingは子に余白を設けるWidgetである。試しにRowで作った3つの四角形をPaddingで囲ってみると

```dart
class _MyHomePageState extends State<MyHomePage> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
        body: Row(
      children: [
        Padding(
          padding: const EdgeInsets.all(8.0),
          child: Container(
            color: Colors.green,
            height: 100.0,
            width: 100.0,
          ),
        ),
        Padding(
          padding: const EdgeInsets.all(8.0),
          child: Container(
            color: Colors.yellow,
            height: 100.0,
            width: 100.0,
          ),
        ),
        Padding(
          padding: const EdgeInsets.all(8.0),
          child: Container(
            color: Colors.red,
            height: 100.0,
            width: 100.0,
          ),
        ),
      ],
    ));
  }
}
```

間が少し(8pxずつ)生まれた。

![](/images/2022-05-28-20-52-51.png)

# 画面遷移

![](/images/2022-05-28-20-59-11.png)

スマホアプリでは、全く別の情報を画面に表示したい場合、画面遷移という方法で表示を切り替える。例えばゲームを作る場合、スタートメニューとゲーム画面は全く異なるため画面遷移を利用するのが望ましい。
Webではハイパーリンクという形式で画面遷移を実装することが多い。

Flutterでは別の画面に移動する方法としてNavigatorという仕組みが使われる。画面遷移の具体的な実装として次の画面のオブジェクトを指定する方法とハイパーリンクのようなものを使う方法の2つが存在するが、前者の方が簡単なのでそちらを紹介する。

## 新しいファイルに新しいページを書く

lib/下にsecond.dartを作る

libを右クリック -> [New File] -> second.dartと入力

![](/images/2022-05-28-21-45-23.png)

中身を以下のように書く。

```Dart
// ignore_for_file: library_private_types_in_public_api, use_key_in_widget_constructors, prefer_const_constructors, avoid_print

import 'package:flutter/material.dart';

class SecondPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: Text(
          "次のページだよ!",
          style: TextStyle(
            fontSize: 40.0
          ),
        ),
      ),
    );
  }
}
```

> 注意: ページとして扱うWidgetの一番上はScaffoldにすること！

## 画面遷移処理を書く

まず、second.dart内のコードにアクセスできるようにインポートする

```dart
import 'package:flutter/material.dart';

import 'second.dart';
```

ボタンを用意してこんな感じに書く

```Dart
class _MyHomePageState extends State<MyHomePage> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
        body: Center(
      child: ElevatedButton(
        onPressed: () {
          Navigator.of(context)
              .push(MaterialPageRoute(builder: (_) => SecondPage()));
        },
        child: Text("Go to Next Page"),
      ),
    ));
  }
}
```

ボタンを押すと次のページが表示される。

![](/images/2022-05-28-21-46-31.png)

少し呪文めいているが、`Navigator.of(context).push(MaterialPageRoute(builder: (_) => 次のページのクラス))`を呼ぶとそのクラスが画面に表示される。

## 前の画面に戻る

次の画面に行った後、ブラウザの戻るボタンをクリックすると元の画面に戻れる。しかしアプリ内の操作で戻りたい場合は、戻りたいときに`Navigator.of(context).pop()`を呼ぶと前の画面に戻る。

second.dartを以下のように書き換える。

> ColumnのMainAxisSize.minは縦方向を中央揃えしてくれる魔法

```dart
class SecondPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Text(
              "次のページだよ!",
              style: TextStyle(fontSize: 40.0),
            ),
            TextButton(
                onPressed: () {
                  Navigator.of(context).pop();
                },
                child: Text("戻る"))
          ],
        ),
      ),
    );
  }
}
```

https://dartpad.dev/?id=f67ebfbf5502d92de6bf1fff80f34ea5

戻るボタンを押すと前のページに戻ることができる。

![](/images/2022-05-28-21-48-39.png)

# おしまい

## 課題(任意)

* ボタンを押すとその色の見本を見せてくれるアプリを作る
  * Flutter標準で使える色たち: https://materialui.co/colors/

![](/images/2022-05-28-22-04-10.png)

![](/images/2022-05-28-22-04-19.png)
