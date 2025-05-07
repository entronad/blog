在数据可视化中，交互是很重要的。Flutter 图表库 [Graphic](https://github.com/entronad/graphic) 拥有一套精心设计的交互系统，以应对各种各样的可交互图表。

这套系统建立在若干个概念之上，只要理解了这些概念，如何处理 [Graphic](https://github.com/entronad/graphic) 中的交互就变得简单而灵活。这些概念中有些是创新的，但它们都是直观而易于理解的。

这篇文章将介绍这些概念，以帮助你通过 [Graphic](https://github.com/entronad/graphic) 创建可交互的 Flutter 图表。

# 手势（Gesture）

作为一个触控优先的 GUI 框架，Flutter 中的交互是建立在手势之上的。

手势系统有两个层级。第一层包含原始的指针事件，它们描述指针（如触控，鼠标，触控笔）在屏幕上的位置和移动。第二层包含手势，它们描述由一个或多个指针移动构成的语义行为。注意手势不仅包含触控，也包括跨平台的各种指针类型。

由于 [Graphic](https://github.com/entronad/graphic) 是一个组件级别的可视化库，我们选择手势层作为交互系统的基础。[Gesture](https://pub.dev/documentation/graphic/latest/graphic/Gesture-class.html) 类包含关于手势的信息，它主要在 [GestureSignal](https://pub.dev/documentation/graphic/latest/graphic/GestureSignal-class.html) 中使用。

在 Flutter 中被广泛用来处理手势的组件是 [GestureDetector](https://api.flutter.dev/flutter/widgets/GestureDetector-class.html)。它在回调参数（比如 [onTap](https://api.flutter.dev/flutter/widgets/GestureDetector-class.html)）中定义了所有的手势类型，开发者对它们很熟悉。所以 [Graphic](https://github.com/entronad/graphic) 继承了这种分类。[Graphic](https://github.com/entronad/graphic) 中的 [GestureType](https://pub.dev/documentation/graphic/latest/graphic/GestureType.html) 与它们在 GestureDetector 中的对应参数有着相同的名字（除了 `on` 前缀）和含义，例如 `GestureType.tap` 和 `GestureDetector.onTap`。这使得 [Graphic](https://github.com/entronad/graphic) 与 Flutter 的手势系统保持一致，并且对开发者友好。

# 信号（Signal）

用来表示交互的有两种抽象层级：信号和选取（selection）。这两种概念来自 Vega，但是在 [Graphic](https://github.com/entronad/graphic) 有些变化。

信号在有些其他系统中又被称为”事件“。它们在用户或外界变化与图表交互时产生。它们包含交互的信息。信号主要用在更新器（updater）中，比如 [RectCoord.horizontalRangeUpdater](https://pub.dev/documentation/graphic/latest/graphic/RectCoord/horizontalRangeUpdater.html)，或者在内部触发选取。

虽然名字一样， [Graphic](https://github.com/entronad/graphic) 中的信号与 [Vega 中的信号](https://vega.github.io/vega/docs/signals/)有着不同的含义。在 Vega中，信号是可视化参数中的动态变量，也就是说不管有没有交互发生，它们持续有值提供。但是在 [Graphic](https://github.com/entronad/graphic) 中，信号是交互的化身，所以它们只在触发时出现，并且携带交互的完整信息，而不仅仅是一个变量值。

除了为用户交互的 [GestureSignal](https://pub.dev/documentation/graphic/latest/graphic/GestureSignal-class.html)，还有为外界变化影响图表的 [ChangeDataSignal](https://pub.dev/documentation/graphic/latest/graphic/ChangeDataSignal-class.html) 和 [ResizeSignal](https://pub.dev/documentation/graphic/latest/graphic/ResizeSignal-class.html)，外界变化广义上来讲也是”交互“。

不管是什么类型或什么发起的信号，在内部将被一个集中器统一广播给所有信号更新器。这使得开发者可以自由的决定更新器对哪个信号做出反应。

# 选取（Selection）

选取是由手势驱动的数据查询。它们是信号的结果。当一个选取被触发时，每条数据将处于要么选中要么未选中的状态，如果定义了 `Attr.onSelection` 的话将导致对应的具象属性变化。

[Graphic](https://github.com/entronad/graphic) 的选取规则主要来自 [Vega-Lite 的选取](https://vega.github.io/vega-lite/docs/selection.html)，所以也分为 [IntervalSelection](https://pub.dev/documentation/graphic/latest/graphic/IntervalSelection-class.html) 和 [PointSelection](https://pub.dev/documentation/graphic/latest/graphic/PointSelection-class.html)。

不同于其它的可视化库，[Graphic](https://github.com/entronad/graphic) 中的选取是在数据值空间中检测的，而不是通过图形的相交。指针的坐标将被转换为各维度上的值，通过这些值在数据中查找结果。这种方法更”数据驱动“，并且在大数据中比图形相交检测效果更好。

总的来讲，信号更底层更灵活，选取更简洁并注重数据。

# 更新器（Updater）

在 [Graphic](https://github.com/entronad/graphic) 中，关于交互如何作用于图表的基本理念是，它们不直接向图表提供值，而是更新图表中已有的参数值并反应式的重新渲染。计算参数值和更新是在不同的算子（operator）中，初始值（通常在定义中指定）将被保存：

![1]()

更新器是根据交换更新属性值的回调函数，例如 [RectCoord.horizontalRangeUpdater](https://pub.dev/documentation/graphic/latest/graphic/RectCoord/horizontalRangeUpdater.html) 或 [Attr.updaters](https://pub.dev/documentation/graphic/latest/graphic/Attr/updaters.html)。由于有两种层级的交互，更新器也对应有两类： [SignalUpdater](https://pub.dev/documentation/graphic/latest/graphic/SignalUpdater.html) 和 [SelectionUpdater](https://pub.dev/documentation/graphic/latest/graphic/SelectionUpdater.html)：

```
SignalUpdater<V> = V Function(
  V initialValue,
  V preValue,
  Signal signal
)

SelectionUpdater<V> = V Function(
  V initialValue
)
```

可以看出这种结构的好处是开发者可以通过算子中保存的初始值和前值更好的控制值的状态。

# 交互通道（Interaction Channel）

对于一般的交互情形，以上的特性已经足够了。不过我们引入交互通道以处理高级应用。

交互通道是与图表进行双向通信的途径。可以通过它输入输出信息。也就是说，可以手动向图表发射信号或选取，以及当信号或选取发生时得到通知。它使得我们能更灵活和更精确的控制交互。

我们考虑通过函数反应式编程（Functional Reactive Programming, FRP）来实现它。幸运的是，Dart 语言有内置的[异步流系统](https://dart.dev/tutorials/language/streams)，是 FRP 的一种简单实现。[StreamController](https://api.dart.dev/stable/2.18.3/dart-async/StreamController-class.html) 类可以承担交互通道的角色。

展示交互通道优势的一个领域是图表耦合。考虑有两个不同的图表，耦合的意思是当与其中的一个交互时，另一个也做出同样的反应，反之亦然。

例如，两个图表分别展示一支股票的价格和成交量，当点击一个图表显示辅助线时，另一个上也展示同样的辅助线：

![2]()

只需要将这两个图表共享同一个手势信号通道，它们就会共享所有的手势了，不需要任何额外的输入输出参数：

```
final priceVolumeChannel = StreamController<GestureSignal>.broadcast();

// the price chart
Chart(
  ...
  gestureChannel: priceVolumeChannel,
)

// the volume chart
Chart(
  ...
  gestureChannel: priceVolumeChannel,
)
```

另一个例子是两个图表总是选中同一天：

![3]()

只需要共享同一个选取通道即可：

```
final heatmapChannel = StreamController<Selected?>.broadcast();

// the above chart
Chart(
  ...
  elements: [PolygonElement(
    selectionChannel: heatmapChannel,
  )]
)

// the below chart
Chart(
  ...
  elements: [PolygonElement(
    selectionChannel: heatmapChannel,
  )]
)
```

完整的示例代码见[这里](https://github.com/entronad/graphic/blob/main/example/lib/pages/interaction_channel_dynamic.dart)。

[英文版本](https://itnext.io/how-to-build-interactive-charts-in-flutter-e317492d5ba1)