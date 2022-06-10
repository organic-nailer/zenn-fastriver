---
title: "Flutter Animationの裏側"
emoji: "🎥"
type: "tech"
topics: ["flutter", "animation"]
published: true
published_at: "2022-04-03 11:23"
---

Flutter、アニメーション効果を作るのが非常に簡単です。最低StatefulWidget上でAnimated系Widgetを使うだけで美しいアニメーションを実装できます。
また凝ったアニメーションを実装する場合でも、AnimationControllerやTweenを使うことで複雑なものも実現可能です。

これらのアニメーションはなぜ動くのか？AnimationControllerの実装から裏側を少し覗いて見ましょう。

# Animationの使い方

Animationを実装する場合の一つの方法として、Transition系のWidgetとAnimationControllerを使うというものがあります。

https://medium.com/flutter-jp/transition-9c57528c84b8

透明度のアニメーションをFadeTransitionを使って組む例を以下に示します。

```dart
class _MyHomePageState extends State<MyHomePage> with TickerProviderStateMixin {
  late final AnimationController controller;
  bool visible = false;
  
  @override
  void initState() {
    super.initState();
    controller = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 1)
    );
  }
  
  @override
  void dispose() {
    controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: FadeTransition(
          opacity: controller,
          child: Container(
            width: 200, height: 200,
            color: Colors.green
          )
        )
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () {
          if(visible) {
            controller.reverse();
            visible = false;
          }
          else {
            controller.forward();
            visible = true;
          }
        },
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

![](https://storage.googleapis.com/zenn-user-upload/4c39da54d12b-20220331.gif)

事前に(initStateで)初期化しておいたAnimationControllerをFadeTransitionに渡し、ボタンが押されたタイミングでcontroller.forward()かreverse()を呼び出すことでアニメーションすることができます。

ここで使われているAnimationControllerがアニメーション操作の入り口になるわけです。

# アニメーション実行の流れの図

![](https://storage.googleapis.com/zenn-user-upload/8cdf5e371a20-20220403.png)

上がFramework側、下がEngine側です。アニメーションは上位・下位レイヤが協調してやっと動作していることがわかります。

# 登録処理

## AnimationControllerの作成

https://api.flutter.dev/flutter/animation/AnimationController-class.html

```dart
  AnimationController({
    double? value,
    this.duration,
    this.reverseDuration,
    this.debugLabel,
    this.lowerBound = 0.0,
    this.upperBound = 1.0,
    this.animationBehavior = AnimationBehavior.normal,
    required TickerProvider vsync,
  }) : assert(lowerBound != null),
       assert(upperBound != null),
       assert(upperBound >= lowerBound),
       assert(vsync != null),
       _direction = _AnimationDirection.forward {
    _ticker = vsync.createTicker(_tick);
    _internalSetValue(value ?? lowerBound);
  }
```

AnimationControllerはDurationや最小最大値などの設定を利用して行うアニメーションを操作するクラスです。開発者はここからアニメーションの開始・停止を制御することができます。

AnimationControllerを作成すると、コンストラクタでvsync.createTicker()を呼び出してTickerを生成しています。

## TickerProvider

コンストラクタで渡されているvsyncとは？ということですが、vsyncといえば

```dart
class _MyHomePageState extends State<MyHomePage> with TickerProviderStateMixin {
  @override
  void initState() {
    super.initState();
    controller = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 1)
    );
  }
```

の部分で、半ばおまじないのように書いていたTickerProviderStateMixin(またはSingleTickerProviderStateMixin)になります。

TickerProviderStateMixinはStateに対するmixinですが、中核の機能としては独立して動作するためStateと切り離して考えても大丈夫です。

さてAnimationControllerのコンストラクタで呼び出したcreateTickerの中身は以下のようになっています。

```dart
  @override
  Ticker createTicker(TickerCallback onTick) {
    //...
    _tickers ??= <_WidgetTicker>{};
    final _WidgetTicker result = _WidgetTicker(onTick, this, debugLabel: kDebugMode ? 'created by ${describeIdentity(this)}' : null)
      ..muted = !_tickerModeNotifier!.value;
    _tickers!.add(result);
    return result;
  }
```

https://github.com/flutter/flutter/blob/c860cba910319332564e1e9d470a17074c1f2dfd/packages/flutter/lib/src/widgets/ticker_provider.dart#L285

TickerModeという概念が度々書かれていますが、これはWidgetツリーの一部のアニメーションを一括制御できるようにするための仕組みの一環なので今回は無視します。
それを除くとここはシンプルで、コールバックを渡すとそこからWidgetTickerを作成しそのWidgetTickerを返り値としているだけになります。

TickerProviderStateMixinはその名の通りTickerを提供するためのクラスであり、コールバックの処理は更に奥で行われています。

## WidgetTicker: Tickerの作成

https://github.com/flutter/flutter/blob/c860cba910319332564e1e9d470a17074c1f2dfd/packages/flutter/lib/src/widgets/ticker_provider.dart#L384

WidgetTickerはTickerを一部拡張したものなのでTickerを追うことにします。

```dart
  Ticker(this._onTick, { this.debugLabel }) {
    assert(() {
      _debugCreationStack = StackTrace.current;
      return true;
    }());
  }
```

https://github.com/flutter/flutter/blob/c860cba910319332564e1e9d470a17074c1f2dfd/packages/flutter/lib/src/scheduler/ticker.dart#L65

先程AnimationControllerから渡したコールバックはこのクラスが保持していることがわかります。
作成時としてはそれだけです。

# アニメーションの開始

ここまで初期化時の処理を見てきました。実際にコールバックが作用するのはアニメーションを開始したときになります。順方向アニメーションを開始するAnimationController.forward()は以下のような実装になっています(reverse(), repeat()もあまり変わりません)。

```dart
TickerFuture forward({ double? from }) {
  //...
  _direction = _AnimationDirection.forward;
  if (from != null)
    value = from;
  return _animateToInternal(upperBound);
}
//...
TickerFuture _animateToInternal(double target, { Duration? duration, Curve curve = Curves.linear }) {
  //...
  return _startSimulation(_InterpolationSimulation(_value, target, simulationDuration, curve, scale));
}
//...
TickerFuture _startSimulation(Simulation simulation) {
  assert(simulation != null);
  assert(!isAnimating);
  _simulation = simulation;
  _lastElapsedDuration = Duration.zero;
  _value = simulation.x(0.0).clamp(lowerBound, upperBound);
  final TickerFuture result = _ticker!.start();
  //...
  return result;
}
```

https://api.flutter.dev/flutter/animation/AnimationController/forward.html

forward()が呼び出されると、最大値に向けて順方向にアニメーションするためにその設定で_animateToInternal()が呼ばれます。そしてその最後にSimulationを作って_startSimulation()を呼びます。
Simulationというのは時間を入力するとその時間での位置xと速度dxを算出してくれるクラスです(Simulation自体はabstract)。_animateToInternal()内で作られている_InterpolationSimulationは指定された区間と時間で線形補間するようになっています。

https://api.flutter.dev/flutter/physics/Simulation-class.html

> 他にはGravitySimulationやSpringSimulationなどが用意されています

Simulationは後で使うので一旦置いておくとして、先を見ると_startSimulation()内でTicker.start()が呼ばれていることがわかります。

## SchedulerBindingへのコールバックの登録

```dart
TickerFuture start() {
  //...
  if (shouldScheduleTick) {
    scheduleTick();
  }
  //...
}

void scheduleTick({ bool rescheduling = false }) {
  assert(!scheduled);
  assert(shouldScheduleTick);
  _animationId = SchedulerBinding.instance!.scheduleFrameCallback(_tick, rescheduling: rescheduling);
}
```

https://api.flutter.dev/flutter/scheduler/Ticker/start.html

https://api.flutter.dev/flutter/scheduler/Ticker/scheduleTick.html

Ticker.start()は内部でscheduleTick()を呼んでいます。そしてその中でSchedulerBindingのscheduleFrameCallback()を呼び出していることがわかります。
SchedulerBindingはWidgetsFlutterBindingの一部で、Engineからのフレーム同期信号を元にフレーム作成や各種コールバックの処理を行う場所です。

https://zenn.dev/fastriver/articles/65a1b96911c86e

----------------------

```dart
int scheduleFrameCallback(FrameCallback callback, { bool rescheduling = false }) {
  scheduleFrame();
  _nextFrameCallbackId += 1;
  _transientCallbacks[_nextFrameCallbackId] = _FrameCallbackEntry(callback, rescheduling: rescheduling);
  return _nextFrameCallbackId;
}
```

https://api.flutter.dev/flutter/scheduler/SchedulerBinding/scheduleFrameCallback.html

これでTicker内の_tick()メソッドがtransientCallbacksに登録されました。

# コールバックの処理

先程SchedulerBinding.scheduleFrameCallback()を呼んだとき、よく見ると内部でscheduleFrame()呼んでいることがわかります。これを呼ぶとなんやかんやあって次のフレームの作成要求のタイミングでSchedulerBinding.handleBeginFrame()が呼ばれるようになっています。

> 暇であれば以下を読んで下さい

https://zenn.dev/fastriver/scraps/04fec681080265

```dart
void handleBeginFrame(Duration? rawTimeStamp) {
  //...
  try {
    // TRANSIENT FRAME CALLBACKS
    _frameTimelineTask?.start('Animate', arguments: timelineArgumentsIndicatingLandmarkEvent);
    _schedulerPhase = SchedulerPhase.transientCallbacks;
    final Map<int, _FrameCallbackEntry> callbacks = _transientCallbacks;
    _transientCallbacks = <int, _FrameCallbackEntry>{};
    callbacks.forEach((int id, _FrameCallbackEntry callbackEntry) {
      if (!_removedIds.contains(id))
        _invokeFrameCallback(callbackEntry.callback, _currentFrameTimeStamp!, callbackEntry.debugStack);
    });
    _removedIds.clear();
  }//...
}
```

https://api.flutter.dev/flutter/scheduler/SchedulerBinding/handleBeginFrame.html

この関数が呼ばれると、蓄えられたtransientCallbacksが1つずつ実行され、使い終わったコールバックは全て破棄されます。

# コールバックの伝搬

transientCallbacksに登録されていたコールバックはTicker._tick()でした。

```dart
void _tick(Duration timeStamp) {
  assert(isTicking);
  assert(scheduled);
  _animationId = null;

  _startTime ??= timeStamp;
  _onTick(timeStamp - _startTime!);

  // The onTick callback may have scheduled another tick already, for
  // example by calling stop then start again.
  if (shouldScheduleTick)
    scheduleTick(rescheduling: true);
}
```

https://github.com/flutter/flutter/blob/c860cba910319332564e1e9d470a17074c1f2dfd/packages/flutter/lib/src/scheduler/ticker.dart#L232

中ではTickerの作成時に渡されたコールバック_onTick()を呼び出しています。
さらにアニメーションが続けられる場合は、scheduleTick()を呼んで再びコールバックをtransientCallbacksに登録するようになっています。少し面倒な気もしますがこれにより安全にアニメーションの動作を管理することができるわけです。
また_onTick()の引数にはアニメーションが開始してから(start()が呼ばれてから)の経過時間を渡すようになっています。

-------------

Ticker._onTick()に登録されていたのはAnimationController._tick()です。

```dart
void _tick(Duration elapsed) {
  _lastElapsedDuration = elapsed;
  final double elapsedInSeconds = elapsed.inMicroseconds.toDouble() / Duration.microsecondsPerSecond;
  assert(elapsedInSeconds >= 0.0);
  _value = _simulation!.x(elapsedInSeconds).clamp(lowerBound, upperBound);
  if (_simulation!.isDone(elapsedInSeconds)) {
    _status = (_direction == _AnimationDirection.forward) ?
      AnimationStatus.completed :
      AnimationStatus.dismissed;
    stop(canceled: false);
  }
  notifyListeners();
  _checkStatusChanged();
}
```

https://github.com/flutter/flutter/blob/c860cba910319332564e1e9d470a17074c1f2dfd/packages/flutter/lib/src/animation/animation_controller.dart#L819

この関数が呼ばれると、まずSimulationに経過時間を入力し、現在の値を計算、更新します。そしてnotityListeners()を呼び出してリスナが存在すればそれを呼びます。

# Transition系Widget内での対応

AnimationControllerを実際にListenして画面に反映するのはWidget側の仕事です。一例としてFadeTransitionの内部を見てみましょう。

```dart
class FadeTransition extends SingleChildRenderObjectWidget {
  /// Creates an opacity transition.
  ///
  /// The [opacity] argument must not be null.
  const FadeTransition({
    Key? key,
    required this.opacity,
    this.alwaysIncludeSemantics = false,
    Widget? child,
  }) : assert(opacity != null),
       super(key: key, child: child);
  //...
  final Animation<double> opacity;
  //...
  @override
  RenderAnimatedOpacity createRenderObject(BuildContext context) {
    return RenderAnimatedOpacity(
      opacity: opacity,
      alwaysIncludeSemantics: alwaysIncludeSemantics,
    );
  }
  //...
}
```

https://api.flutter.dev/flutter/widgets/FadeTransition-class.html

Widget自体の実装は非常にシンプルです。AnimationControllerはopacityに渡され、そのまま対応するRenderObjectであるRenderAnimatedOpacityに渡していることがわかります。
RenderObjectは実際にレイアウトや描画などを担当するオブジェクトで、Widgetツリーと似通ったツリー構造になっています。

https://api.flutter.dev/flutter/rendering/RenderAnimatedOpacity-class.html

ここで少しややこしいのですが、RenderAnimatedOpacityの実装のほとんどはRenderAnimatedOpacityMixinに記述されています。mixin側を見ると

```dart
  @override
  void attach(PipelineOwner owner) {
    super.attach(owner);
    opacity.addListener(_updateOpacity);
    _updateOpacity(); // in case it changed while we weren't listening
  }

  @override
  void detach() {
    opacity.removeListener(_updateOpacity);
    super.detach();
  }

  void _updateOpacity() {
    final int? oldAlpha = _alpha;
    _alpha = ui.Color.getAlphaFromOpacity(opacity.value);
    if (oldAlpha != _alpha) {
      final bool? didNeedCompositing = _currentlyNeedsCompositing;
      _currentlyNeedsCompositing = _alpha! > 0;
      if (child != null && didNeedCompositing != _currentlyNeedsCompositing)
        markNeedsCompositingBitsUpdate();
      markNeedsPaint();
      if (oldAlpha == 0 || _alpha == 0)
        markNeedsSemanticsUpdate();
    }
  }
```

https://github.com/flutter/flutter/blob/c860cba910319332564e1e9d470a17074c1f2dfd/packages/flutter/lib/src/rendering/proxy_box.dart#L973

RenderObject.attach()はRenderObjectがツリーに接続されたときに呼ばれる関数です。ここでAnimationControllerにリスナとして_updateOpacity()を登録しています。

SchedulerBinding.handleBeginFrame()が実行されると、Ticker・AnimationControllerを経由して最終的にこの関数が呼ばれるというわけです。_updateOpacity()内では変化したcontrollerの値を使って自身のalphaを再計算し、最後にmarkNeedsPaint()という関数を呼び出しています。
これはRenderObjectの再描画を描画パイプラインに対してお願いするもので、次の画面更新のタイミングで新しい透明度に再描画され、画面に表示されることになります。

:::message
実際にはフレーム処理の順番がtransientCallbacksの実行→再レイアウト・再描画となっているため同じフレーム内で画面が更新されます。
:::

# 終わり

~~Flutterのアニメーションは、markNeedsPaint()などを呼び出している点では普通の画面更新とやっていることはほとんど同じですが、setStateによる更新とは異なりWidgetを再構築せず、RenderObjectを直接いじっていることからアニメーションの仕組みに乗るほうが多少パフォーマンスが期待できます。~~
Flutterのアニメーションを少し深く考える手助けになれば幸いです。

> 2022/04/04 追記: 一部表現が不適当だったため修正しました。