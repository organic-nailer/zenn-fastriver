---
title: "Day1: Dartの基礎1(変数・四則演算等)"
free: false
---

# Dart入門

まずはFlutterでアプリを作る前にプログラミング言語の勉強をしましょう。

## 環境を用意しよう

[dartpad.dev](https://dartpad.dev/)にアクセス！

![](https://www.mediafire.com/convkey/53f9/0vuxmj0c97t8pg36g.jpg)

* エディタ(左): ここにDartのコードを書く
* コンソール(右上): プログラムの出力結果や、エラーが出る
* ドキュメント(右下): カーソルを合わせた部分の説明が出る
* RUNボタン: プログラムを開始する

プログラムはmain() {}の中括弧の中に書く

print([ここに何かを書くと結果を右のコンソールに出力してくれる]);

//(スラッシュ2つ)の後の文字はコメントになる(無視される)

```Dart
void main() { //ここから
	print(5); // -> 5が出力される
//ここまでがプログラムを書ける範囲
}
```

[DartPadで見る](https://dartpad.dev/8730beeb18db60eb18c81e2bc6a882b8)

## プログラムの基本

入力→(なんらかの処理)→出力

> 入力は無いことも多い

最初の方はフローチャートを考えながらやると便利

## 算術演算

Dartでは数字の計算をするときに、演算子を使います。

```Dart
2 + 3 // -> 5 足し算
4 - 14 // -> -10 引き算
2 * 4 // -> 8 掛け算
12 / 3 // -> 4 割り算
17 % 3 // -> 2 余り
```

複数の演算子を同時に使うことも可能

```Dart
2 * 3 + 4 + 5 // -> 15
```

演算子には優先順位があり、高いほうが先に計算され、同じ場合は左から計算が行われる

`* = / = % > + = -`

基本的に普通の四則演算と同じ

計算の順序を変えたい場合は()を使って囲む

```dart
void main() {
	print(2+3*(4-2));
}
```

[DartPadで見る](https://dartpad.dev/4cfa955c67d4bd51e96e9789f002669c)

## 変数・代入

プログラミング言語では、数字や文字列・論理値(true false)など(値と呼ぶ)を名前をつけた変数に"代入"ができる。

```Dart
void main() {
	//ageという変数を作り数字を代入する
	int age = 19;
	//nameという変数を作り文字列を代入する
	String name = "hayakawa";

	//出力する
	print("年齢: $age 名前: $name");
}
```

DartはC言語やJavaなどと同じように

[変数の種類] [変数名] = [代入する値];

という形で変数を宣言・代入する。代入する値(=X)がない場合、変数はnullで初期化される(後述)。

変数の種類(型)は

* int: 32bit整数型。数字をそのままタイプする
* String: 文字列型。""ダブルクオーテーションで文字を囲う
* bool: 論理値型。trueとfalseのみが入る

等がある。

変数名には半角の英数字と一部の記号が使えるが、先頭を数字にすることはできない。

変数の種類と変数名の間には半角スペースを空ける(くっついていると一つの単語だと解釈されてしまう)

変数の右側に存在する"="は代入演算子と呼ばれ、数学におけるイコールとは意味が違うので注意が必要。代入演算子は右にある値を左にある変数に代入するという意味がある。代入に関しては、「age」と書かれた箱の中に19という数字を入れる、のようなイメージでとりあえずよい。

変数の宣言時に代入をしない場合は以下の通り。

```Dart
//mainは省略
int age;
print(age);
```

この場合はageを評価すると中に何も入っていないことを表す「null」が出力される。

式の最後には;セミコロンを付けなくてはならない。逆にこれをつければ式の終わりと解釈してくれるので、改行せずにそのまま次の式を書いてもOK。

最後の行に関しては、文字列の中(""の中)では、$ドルマークの後ろに変数の名前を付けたものを書くとその変数の中身を文字に変換して出力してくれる(便利！)

## 文字列操作

2つ以上の文字列をくっつけたい場合にも"+"が使える。

```Dart
void main() {
	String lastName = "Hayakawa";
	String firstName = "Yuki";

	print(lastName + " " + firstName); // -> Hayakawa Yuki
}
```

> 文字列の変数と普通の文字列は同列に扱うことができる

## 条件分岐

「if文」を使って条件によって異なる動作をすることができる

```Dart
void main() {
	int age = 19;
	if(age < 20) {
		print("$age 歳はお酒NG");
	}
	else {
		print("$age 歳はお酒OK");
	}
}
```

![](https://www.mediafire.com/convkey/19b0/vb5gdnur31d7l7i6g.jpg)

[DartPadで見る](https://dartpad.dev/8c403db0784bebc1f13a25fbbf935750)

if([条件式]) { [条件式がtrueだったときの処理] }

else { [条件式がfalseだったときの処理] }

else以下は無くてもよい。もし条件式がfalseで、そこからまた条件分岐したい場合は

else if([条件式2]) { ... }

のように続けることができる

### 条件式

比較演算子を使うことで条件に適合しているかどうか調べることができる。
比較演算子は、左と右の値を比較してtrueもしくはfalseを返す。

* `==`: 左右が同じ値のときにtrue
* `!=`: 左右が違う値のときにtrue
* `>`: 左のほうが大きいときにtrue
* `<`: 左のほうが小さいときにtrue
* `>=`: 左右同じか左のほうが大きいときにtrue
* `<=`: 左右同じか左のほうが小さいときにtrue

```Dart
"Hiyoshi" == "Hiyoshi" // -> true
"Hiyoshi" != "Yagami" // -> true
10 > 12 // -> false
10 < 10 // -> false
13 >= 13 // -> true
15 <= 18 // -> true
```

## [コラム] 式と文

プログラムを書くときには、文法的に「式」と呼ばれる部分と「文」と呼ばれる部分が存在する。先程のif文は文であるし、条件式は式である。

- 式(Expression): 中にある数式や関数を評価して、最後に何かの値を結果として出すもの
  - 算数の式のように最後まで解くことができる
  - 例: 条件式(結果は真偽値), 算術演算式(結果は数字等)
- 文(Statement): プログラムで行う手続きを記述するもの
  - セミコロンで終わる行や、中括弧{}で括られた部分が1つの文
  - 例: if文, 変数宣言, 代入

多くの言語(Dart含む)では式と文がはっきり区別されていて、これによりプログラムの要素が書ける場所、というのが制限される。例えばif文の()の中は式を入れなければならないため、そこに代入文を書くことはできない。

## 繰り返し

「for文」を使って繰り返し同じ処理を行うことができる

```Dart
void main() {
	for(int i = 0; i < 10; i = i + 1) {
		print("$i 回目のHello");
	}
	print("おしまい");
}
```

![](https://www.mediafire.com/convkey/27ee/vk1bdd7ul26yjqn6g.jpg)

[DartPadで見る](https://dartpad.dev/04526557b91b56026bbd9c510ef5b9e9)

for([初期化式]; [条件式]; [更新式]) { [繰り返しやる処理] }

初期化式: for文に入ったときに一度だけ実行される。繰り返し処理が何回目かを表す変数を用意することが多い

条件式: 毎回繰り返し処理を行う前に評価される。ここがtrueになるときのみ繰り返し処理が行われる。falseになった場合はfor文から抜ける

更新式: 繰り返し処理を行った後に毎回実行される。初期化式で用意した変数をここで更新することが多い

更新式の`i = i + 1`に違和感のある人もいるかも知れないが、これは正しい。代入演算子は右から左に評価していくので、まず`i + 1`が計算されてそれが`i`に代入される(つまりiを1だけ加算している)ことになる。決して`=`は等号ではない

### 変数のスコープについて

{}の内側で宣言された変数はその内側でしか使えない。if文やfor文の条件式部分は中のカッコに含まれる。

# おまけ エラー集

謎の波線に悩まされた時にお使いください

`int hoge = 10` → 最後にセミコロン;が無い

`Stringhoge="fuga";` →型名と変数名の間にスペースがない

`string hoge="piyo";` →大文字小文字をミスしている

```Dart
void hogehoge() {
	if(true) {
		fuga();
	}

```
→カッコの数が対応していない

```Dart
int a = 2;
String a = "hoge";
```
→既に使われている変数名を使っている

```Dart
int a = 2;
String b = a;
```
→違う型の変数に代入しようとしている

# 演習問題

 * level変数(int)を作って、50レベル以上なら「つよい」と表示するプログラムを作る
   * つよいと言われるのにあと何レベル必要か教えてあげる
 * FizzBuzz(1から順に表示して、3で割り切れる場合は「Fizz」、5で割り切れる場合は「Buzz」、15で割り切れる場合は「Fizz Buzz」と表示する)
   * 1, 2, Fizz, 4, Buzz, Fizz, 7, 8, Fizz, Buzz, 11, Fizz, 13, 14, Fizz Buzz, 16 ...
* フィボナッチ数を表示する

解答例は[こちら](https://gist.github.com/organic-nailer/7a0bfee910f1467da9e1c92ac51baffb)

# 宿題

Day2を読む前に必要な事項です

* Flutterをインストールする
  * [公式ドキュメント](https://flutter.dev/docs/get-started/install)
  * [簡単インストーラ](https://github.com/YazeedAlKhalaf/Flutter_Installer/releases/tag/v0.0.8)
    * ページ下のAssetsから自分のOSのzipをダウンロード・実行
    * AndroidStudioはインストールしないほうがよい(古いので)
* VSCodeをインストールする
  * [公式ダウンロードページ](https://azure.microsoft.com/ja-jp/products/visual-studio-code/)
  * Flutterの拡張機能を入れる
* Chromeをインストールする
  * Windowsの人はEdgeでも動くのでやらなくてもよし
  * [HP](https://www.google.com/chrome/)
