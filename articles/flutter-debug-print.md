---
title: "[Flutter] debugPrint()はReleaseビルドでも出力されてしまう"
emoji: "🖨️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter"]
published: true
---

# 結論

debugPrint()はprint()と同様、[公式ドキュメント](https://api.flutter.dev/flutter/foundation/debugPrint.html)に書いてあるとおり、Debugモードのみで使うようにする

```dart
if (kDebugMode) {
  debugPrint('A useful message');
}
```

# debugPrint()の罠

皆さん、printデバッグしてますか？　その手軽さから使う方は多いと思います。しかし標準出力はアプリ利用者からも簡単に確認できてしまうため、配布版では出力しないよう注意を払う必要があります。

Flutterには標準で3種類の出力方法が存在しています。

* [print()](https://api.flutter.dev/flutter/dart-core/print.html) [dart:core]
* [debugPrint()](https://api.flutter.dev/flutter/foundation/debugPrint.html) [flutter/foundation.dart]
* [log()](https://api.flutter.dev/flutter/dart-developer/log.html) [dart:developer] 

print()はdart:coreに用意された標準出力用の関数で、これはアプリのビルドモードに関係なく出力されます。リリースモードでの出力を避けるため、そのままの使用は避けkDebugModeの条件で囲うことがデフォルトのLinter Ruleで規定されています([avoid_print](https://dart.dev/tools/linter-rules/avoid_print))。

![](/images/flutter-debug-print-2024-05-16-00-01-35.png)

![](/images/flutter-debug-print-2024-05-16-00-01-55.png)

avoid_printルールでは、「プロダクションコードではロギングフレームワークを使うこと」「FlutterではdebugPrintを使うor printをkDebugModeで囲うこと」と書かれています。

> For production code, consider using a logging framework. If you are using Flutter, you can use debugPrint or surround print calls with a check for kDebugMode

しかし紛らわしいことに、**debugPrint()はただのprint()のラッパーであり、特にデバッグモードのみ出力するといった機能は搭載されていません。** リリースモードでも容赦なく出力が垂れ流されます。

そのため結論でも書きましたが、リリースモードでの表示を避けるためには、debugPrint()もkDebugModeで囲うことが要求されます。

この辺りの説明については、最近(2024年1月)になってやっと[debugPrint()のドキュメント](https://api.flutter.dev/flutter/foundation/debugPrint.html)に追加されました。

![](/images/flutter-debug-print-2024-05-16-00-17-26.png)

なぜdebugPrint()という名前なのにデバッグオンリーではないのか、という疑問については[追加時のPR](https://github.com/flutter/flutter/pull/141595)で議論されていました。

> In general, that is the convention in the framework: methods starting with debug* should only be invoked from a debug context. debugPrint is no exception there.

debug接頭辞のついた関数は、デバッグ時のみにおいて使うべきという命名規則らしいです。ややこしいね。

# debugPrint()の用途について、実装を追ってみる

モードによる挙動の変化でなければ、print()とdebugPrint()を利用する際の差分は何になるのでしょうか。

各出力方法については、[このIssue](https://github.com/flutter/flutter/issues/147141)にまとまっています。

![](/images/flutter-debug-print-2024-05-16-00-24-59.png)

print()では大量のメッセージを一気に出力しようとしたとき、OS側の事情で一部のメッセージが省略されてしまう可能性があります。
debugPrint()では内部にスロットリングの仕組みを持っており、メッセージの流量を調整することでメッセージの省略を防ぐ役割を持っています。
つまりはdebugPrint()のほうが正確にメッセージを出力できるようになっているというわけです。

動きを確かめるため、debugPrint()の実装を追ってみましょう。デフォルトの実装はdebugPrintThrottled()関数です。

https://github.com/flutter/flutter/blob/39651e84ea1296a0b100f519a7771273ff46fe4f/packages/flutter/lib/src/foundation/print.dart#L46

https://github.com/flutter/flutter/blob/39651e84ea1296a0b100f519a7771273ff46fe4f/packages/flutter/lib/src/foundation/print.dart#L62-L103

debugPrintThrottled()内では、メッセージが行ごとに分割されて_debugPrintBufferに格納されます。そして_debugPrintTask()が呼ばれます。
_debugPrintTask()の中核はwhile文です。ここで_debugPrintBufferが1つづつ、1024*12バイトを超えるまでprint()で出力されます。
ここに至るまで特にデバッグモードなどの分岐はなかったので、リリースモードでも関係なく出力されることがわかると思います。

関数の他の部分では出力の間隔調整のためのStopwatchを動かしています。_kDebugPrintPauseTimeは1秒に指定されており、最後に実行されてから1秒間は_debugPrintTask()の実行がブロックされるようになっています。
これによって1秒間に1024*12バイトのみが出力されるように制御しているというわけです。

ということで実装からもdebugPrint()がリリースモードで出力してしまうことを確認できました。うっかり機密データを流してしまうことのないよう、気をつけましょう。
