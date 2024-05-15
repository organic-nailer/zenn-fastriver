---
title: "Pythonの.pycファイル入門"
emoji: "🌐"
type: "tech"
topics: ["python"]
published: true
---

こんにちは、fastriver(@fastriver_org)です。今回は皆さん一度は夢に見るであろう、Pythonのバイナリファイルの解析・扱い方について紹介します。

## .pycファイルとは

![](/images/python-pyc-binary-2023-05-07-15-04-18.png)

Pythonはインタプリタ型言語ですが、直接実行するのではなく一旦バイトコードに変換してからそれをPythonVM上で実行する、ということを行っています。PythonVMはJVMなど他の一般的なVM同様にスタックマシンであり、スタック上に値を積んだり変換したりを繰り返して計算を進めます。

> 便宜上バイトコードと呼んでいますが、PythonVMのバイトコードは1命令あたり2バイトの大きさです。

Pythonはキャッシュとして、バイトコードを一時的に.pycファイルに書き出すことがあります。

## .pycの生成と利用

.pycは普通にPythonを使っていても見ないファイルかもしれません。実は、Python実行時に生成されている`__pycache__`フォルダ内で書き出されています。
実行はせず.pycファイルだけ生成したい、という場合は以下のコマンドを使うことができます。

```shell
$ python -m compileall example.py
```

このコマンドにより、`example.py`から`__pycache__/example.cpython-39.pyc`が生成されます(バージョン部分は環境に依る)。
また、Pythonスクリプトからも.pycにコンパイルすることが可能です。

```python
import py_compile
py_compile.compile("example.py", cfile="example.pyc")
```

こちらの場合は、`cfile`引数に渡した好きな場所に.pycファイルを書き出すことができます。

.pycファイルは、Pythonスクリプトと同様にコマンドから実行することができます。また拡張子を.pyとして指定しても、同名のファイルがない場合は.pycが実行されます。

```shell
$ python example.pyc
```

Pythonスクリプトと同じように振る舞うので、他のファイルでimportして使うことも可能です。

```python
import example
```

しかし、後述のように.pycファイルはPythonのバージョンなどによって仕様が異なるため、生成したときの環境と同じ環境でしか動かせず可搬性はありません。

## .pycのファイル構造

> 以下すべて環境はPython 3.9を想定しています。他の環境だとかなり相違あるかもしれません。

.pycはPythonスクリプトからコンパイルして生成し、同じように振る舞うことからバイトコードが中に書かれているのだろうということは予想がつきますが、実際どのようなファイル形式になっているのでしょうか？
.pycはバイナリ形式で保存されているため、テキストエディタで開いて中身をみることはできません。別途バイナリエディタを使うかVS CodeのHex Editor(ms-vscode.hexeditor)を用意しましょう。

とりあえず足し算を出力する以下のプログラムをコンパイルしたものが以下になります。

```python: example.py
print(1+2)
```

![](/images/python-pyc-binary-2023-05-07-15-33-36.png)

たった10文字のプログラムなのにバイナリのほうが大きいですね。それもそのはず、.pycファイルにはバイトコードだけでなく様々なデータが格納されています。
.pycファイルに格納されているデータ形式の正体は、Pythonのmarshalモジュールで整列化(バイナリ形式に変換)されたCode Objectです。ここでは詳しく書きませんが、文字列や数値、リストや関数などPython内で用いられる多くのオブジェクトはこのmarshalモジュールで整列化することができます。
公式ドキュメントにはCode Objectの仕様に関する記述はありません。これはユーザがこれを使うことを想定していないからです。実際、Pythonのバージョンが少し変わるだけで形式がガラッと変わるなど、かなり面倒な実装になっています。
しかし有志がCode Objectの解析と逆アセンブラとしてpycdc(https://github.com/zrax/pycdc)を公開してくれているので、構造の一部はここから把握することができます。あとはCPythonのコードを読む以外ありません。

Python3.9での.pycファイルの構造は以下のようになっています。Pycから始まる名前はmarshalモジュールで整列化されたObjectです。

![](/images/python-pyc-binary-2023-05-07-16-49-20.png)

最初の16バイトはヘッダのようなものです。Magic Numberはコンパイルに使ったPythonのバージョンごとに異なり、例えばPython 3.9は0x0A0D0D61になります。先程のexample.pycを見ると次のような構造であることがわかります。

![](/images/python-pyc-binary-2023-05-07-17-29-07.png)

続いてCode ObjectであるPycCodeが来ます。

![](/images/python-pyc-binary-2023-05-07-16-50-02.png)

Objectの最初の1バイトはObjectTypeというオブジェクト固有の数字が入ります。

> ObjectTypeの最上位1bitは別の目的のためのフラグになっているため、オブジェクトごとに2つの数値が入る可能性があります。

Code ObjectにはPythonの関数なども表現できるように、引数やローカル変数の設定が存在しています。ファイルのCode Objectには関係ありません。バイトコードに関連する箇所はstackSize、byteCodes、constants、names辺りです。stackSizeはこのバイトコードが最大で使うスタックの長さを指定します。bytecodesにはバイトコードが入ります。バイトコードには命令が固定サイズであるという関係上変数名や定数などを直接書くことができないため、constantsとnamesにそれぞれObjectを記録し、添字でバイトコードからアクセスするようになっています。

![](/images/python-pyc-binary-2023-05-07-17-29-48.png)

byteCodesはPycStringという形式で格納されます。ややこしいのですが、文字列を表現するObjectはPycStringではなくPycAscii、PycSmallAscii、PycUnicodeです。PycStringはバイト列を格納する形式になります。また、constants、namesでは主にPycSmallTupleが用いられます。

![](/images/python-pyc-binary-2023-05-07-17-30-58.png)

PycStirngでは2-5バイト目で値のバイト数を指定します。次のバイナリを見ると、バイト数は0x0C、つまり12バイトのデータを持っていることがわかります。またPycSmallTupleでは2バイト目で子Objectの数を指定します。constantsは2つ、namesは1つ子を持っていますね。

![](/images/python-pyc-binary-2023-05-07-17-33-07.png)

バイトコードはPycString内のバイナリ列として表現されており、1命令辺り2バイトの大きさを持っています。それぞれ1バイト目がopcode(命令の種類)、2バイト目がoperand(演算対象)です。ここから先程のexample.pycのバイトコードを逆アセンブルしてみましょう。

```
1. 0x65, 0x00 -> LOAD_NAME    , 0: print
2. 0x64, 0x00 -> LOAD_CONST   , 0: 3
3. 0x83, 0x01 -> CALL_FUNCTION, 1
4. 0x01, 0x00 -> POP_TOP      , 0
5. 0x64, 0x01 -> LOAD_CONST   , 1: None
6. 0x53, 0x00 -> RETURN_VALUE , 0
```

1バイト目がそれぞれの命令に対応しています。命令ごとにoperandが示す値の意味は異なり、例えばLOAD_NAMEのoperandはnamesの添字、LOAD_CONSTのoperandはconstantsの添字に解釈されます。またCALL_FUNCTIONのoperandは引数の数です。POP_TOPやRETURN_VALUEのようにoperandを使わない命令も存在します(その場合は0で埋められます)。
軽くそれぞれの命令の操作を確認しておきましょう。LOAD_XXXは指定された値をスタックの先頭に積む命令です。逆にPOP_TOPはスタックの先頭から値を一つ下ろす命令になります。CALL_FUNCTIONはスタックの先頭から指定された数だけ積まれている値を引数として処理し、その下の値を関数として実行します。実行後は関数の返り値がスタックの先頭に残ります。RETURN_VALUEはここではプログラムの終了を意味します。ファイルの最後は`return None`とするきまりのようです。

## バイトコードだけでHello, worldしてみる

実際のコードから生成した.pycファイルの構造をなんとなく理解することができたので、次はバイトコードを想像して直接書いてみることをやってみましょう。PythonのHello, worldは次のようなプログラムになると思います。

```python
print("Hello, world!")
```

ではこれをPythonのバイトコードで表現するとどうなるでしょうか？　先程のコードを踏まえると次のように書けます。

```
1. 0x65, 0x00 -> LOAD_NAME    , 0: print
2. 0x64, 0x02 -> LOAD_CONST   , 2: "Hello, world!"
3. 0x83, 0x01 -> CALL_FUNCTION, 1
4. 0x01, 0x00 -> POP_TOP      , 0
5. 0x64, 0x01 -> LOAD_CONST   , 1: None
6. 0x53, 0x00 -> RETURN_VALUE , 0
```

文字列をconstantsの2番目に置きましょう。するとconstantsの部分に変更を入れる必要があります。
ASCIIで表現できる256文字以下の文字列は、PycShortAsciiという形式で表現されます。エンコードはもちろんASCIIです。

![](/images/python-pyc-binary-2023-05-07-17-55-56.png)

`Hello, world!`という文字列は13文字、ASCIIでエンコードすると`48 65 6C 6C 6F 2C 20 77 6F 72 6C 64 21`になります。そのため変更箇所は

1. バイトコードの3番目を0x00から0x02に変更
2. constantsのlengthを0x02から0x03に変更
3. constantsの後ろに`7A 0D 48 65 6C 6C 6F 2C 20 77 6F 72 6C 64 21`を挿入

になります。どうにかしてバイナリを編集し、`hello.pyc`として保存したのが次のバイナリです。

![](/images/python-pyc-binary-2023-05-07-18-20-52.png)

実行すると、Hello, world!と表示されます。

```shell
$ python hello.pyc
Hello, world!
```

Pythonスクリプトを経由せずに、PythonVM上でハローワールドを動かすことができました！
夢が広がりますね。
