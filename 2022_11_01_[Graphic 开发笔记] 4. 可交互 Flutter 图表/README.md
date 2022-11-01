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

不同于其它的可视化库，[Graphic](https://github.com/entronad/graphic) 中的选取是在数据值空间中检测的，而不是通过图形的相交。