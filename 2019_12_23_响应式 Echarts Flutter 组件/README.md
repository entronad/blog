> 本文介绍了一种响应式 Echarts Flutter 组件：[flutter_echarts](https://github.com/entronad/flutter_echarts) 的开发思路
>
> [repository](https://github.com/entronad/flutter_echarts) 
>
> [pub.dev](https://pub.dev/packages/flutter_echarts) 

Flutter 随着自身的发展，逐渐被应用到较为大型的应用中，复杂的数据可视化图表日益成为一个重要的需求。虽然 Flutter 有强大的 Painter、Canvas 用于图形绘制，但不幸的是，目前 Flutter 生态圈中还没有功能强大又易用的杀手级可视化库。

今年以来，Flutter 开发组推出了官方的内联 WebView 组件：[webview_flutter](https://pub.dev/packages/webview_flutter) ，它基于 Flutter 新的 Platform View，使得我们可以将 Web 的内容，和其它 Widget 一样，无缝的嵌入到 Flutter 中。因此我们可以将 Web 中那些成熟完善的可视化库引入到我们的 Flutter 应用中。

说到成熟完善、强大易用的可视化库，[Echarts](https://www.echartsjs.com/zh/index.html) 无疑是一个很好的选择。它的优点就不再赘述了，将 Echarts 应用到 Flutter app 中，不仅可以实现 Echarts 支持的各种丰富的图表类型，满足各种需求，而且能够复用 Web 端现成的配置代码，减少工作量。

因此我们封装了 Flutter 组件： [flutter_echarts](https://github.com/entronad/flutter_echarts) ，兼顾易用性和扩展性原则，争取做到既方便 Flutter 开发人员使用，又尽可能的发挥 Echarts 的功能。

# 特性

在此之前，在 React Native 中封装 Echarts 的 [实践](https://github.com/entronad/react-native-echarts-demo) 里，我们总结了一些在响应式前端框架中使用可视化库的经验，结合对 Flutter 的理解，为 flutter_echarts 设计了如下特性：

**响应式更新**

Flutter 和所有响应式前端框架一样，最重要的特点之一就是可以根据数据的变化自动更新视图，这为开发带来了极大的便利。Echarts 是一个独立于任何 UI 框架的可视化库，但是 ECharts 架构设计上是由数据驱动，数据的改变驱动图表展现的改变。

因此只需要将 Echarts 的动态数据驱动与 Flutter 组件的视图更新联系起来，就可以实现响应式更新。ECharts 动态数据驱动图表更新的实现非常简单，所有数据的更新都通过 `setOption` 实现，你只需要定时获取数据，`setOption` 填入数据，而不用考虑数据到底产生了那些变化，ECharts 会找到两组数据之间的差异然后通过合适的动画去表现数据的变化。Flutter 中，当容器组件更新，传给子组件的数据发生变化时，会触发 StatefulWidget 子组件的 `State.didUpdateWidget` 的方法，在其中调用 `setOption` 就可以通知 Echarts 更新图表，使得整个子组件用起来像 StatelessWidget 一样简便。

**双向通信**

图表与外部逻辑的相互通信是必不可少的，在 flutter_echarts 中，图表的 JavaScript 程序和 组件的 Dart 程序之间采用类似父子组件之间" props down, events up "的方式进行通信。

外部对图表的所有设定、更新通过 `option` 和 `extraScript` 这两个参数，以 JavaScript 字符串的形式传给 WebView 执行；而 WebView 中触发的事件则通过 JavascriptChannel 传递给外部的 onMessage 函数处理，以此实现外部 JavaScript 程序和内部 Dart 程序的双向通信。

**配置扩展**

Echarts 有很丰富的 [扩展](https://echarts.apache.org/en/download-extension.html) ，包括图表、地图、WebGL 等，在 Web 开发中，它们可以以脚本的形式引入代码，从而扩展 Echarts 的功能。为满足开箱即用， flutter_echarts 内置了最新版的 Echarts 脚本，无需额外引入，同时提供了 `extensions` 参数，方便使用者引入所需的扩展脚本。 `extensions` 参数类型为字符串数组，使用者可直接拷贝脚本作为字符串到源码中，避免了文件读写操作和繁琐的 asset 目录。

# 组件参数

封装功能性的组件，其易用性往往比完备性更重要，要让任意水平的开发者都能开箱即用。Echarts 本身在设计时也是遵循易用性的原则，尽可能的将所有配置工作，交给 `option` 这一个参数 去完成（ [详见论文](http://www.cad.zju.edu.cn/home/vagblog/VAG_Work/echarts.pdf) ）。因此 flutter_echarts 在设计时也尽量简化组件参数：

**option**

*String*

字符串形式的 JavaScript Echarts Option。Echarts 图表主要就是通过这个参数配置的。你可以通过 dart:convert 中的 `jsonEncode()` 来转换 Dart 对象类型的数据：

```
source: ${jsonEncode(_data1)},
```

由于 JavaScript 没有`'''` 符号，你可以使用它来包裹字符串，以省掉一些引号的转义：

```
Echarts(
  option: '''
  
    // option string
    
  ''',
),
```

**extraScript**

*String*

在 `Echarts.init()` 和任意 `chart.setOption()` 之间执行的 JavaScript 脚本。在组件中我们已经内置了一个 名为 `Messager` 的 JavascriptChennel，所以你可以使用这个标识符来进行 JavaScript 向 Flutter 的通信：

```
extraScript: '''
  chart.on('click', (params) => {
  if(params.componentType === 'series') {
  	Messager.postMessage('anything');
  }
  });
''',
```

**onMessage**

*void Function(String)*

处理 `extraScript` 中 `Messager.postMessage()` 发送的消息的函数。

**extensions**

*List<String>*

从 Echarts 扩展中拷贝的脚本字符串组成的数组，比如各种组件、WebGl、语言等。可以从 [这里](https://echarts.apache.org/en/download-extension.html) 下载。将它们作为原始字符串（raw string）引入：

```
const liquidPlugin = r'''

  // copy from liquid.min.js

''';
```

---

目前仅有以上 4 个参数，控制更新等由内部机制完成，争取做到用起来就像个简单的表现型 StatelessWidget，只要使用者熟悉 Echarts 本身而不需要额外的学习成本。

当然，如果有建议或要求，请发起 [issue](https://github.com/entronad/flutter_echarts/issues) 。

# 源码解析

**html 的加载**

对于跨平台的开发方案，由于不同的底层操作系统，文件资源目录一直是个麻烦的事情，在 React Native 中有时甚至必须手动将 html 拷贝到 Android 对应的目录。Flutter 虽然有了完善的 asset 系统，但也需要额外的依赖和配置。直接将本地 html 作为源码中的文本字符串加载是解决这些问题的好办法，webview_flutter 的官方示例也比较推荐用这种办法处理本地 html 。

因此我们将模板 html 、Echarts 脚本、扩展脚本、初始化逻辑等在组件初始化时拼接成字符串，作为 uri 资源供 WebView 加载：

```
  @override
  void initState() {
    super.initState();
    _htmlBase64 = 'data:text/html;base64,' + base64Encode(
      const Utf8Encoder().convert(_getHtml(
        echartsScript,
        widget.extensions ?? [],
        widget.extraScript ?? '',
      ))
    );
    _currentOption = widget.option;
  }
  
  ...
  
  @override
  Widget build(BuildContext context) {
    return WebView(
      initialUrl: _htmlBase64,
      
      ...
    );
  }
```

值得注意的是，作为 uri 资源的字符串，是有一些特殊字符限制的，因此加载时我们将字符串转为 Base64 编码。

这里有一个小技巧由于 JavaScript 中没有 `'''` 这个符号，因此在 Dart 中用 `'''` 包裹 JavaScript 脚本字符串可以减少很多转义工作。

**图表更新**

响应式更新基本的实现机制就是在 State.didUpdateWidget 方法中通过`setOption` 通知 Echarts 更新图表：

```
  void update(String preOption) async {
    _currentOption = widget.option;
    if (_currentOption != preOption) {
      await _controller?.evaluateJavascript('''
        chart && chart.setOption($_currentOption, true);
      ''');
    }
  }

  @override
  void didUpdateWidget(Echarts oldWidget) {
    super.didUpdateWidget(oldWidget);
    update(oldWidget.option);
  }
```

这其中比较麻烦的是在组件刚刚初始化的时候。

我们知道 WebView 加载 html 和外部数据的获取都是异步的，事先并不知道谁会先完成。WebView 初始化时生命周期的顺序是：

```
onWebViewCreated --> 加载html --> onPageFinished
```

而 WebViewController 一般是在 onWebViewCreated 中获取的。换言之，当组件拿到 WebViewController 时，并不能确保 WebView 中的 html 已经加载完成，所以 `didUpdateWidget` 不能仅依据是否已经拿到 WebViewController 决定是否可以更新了。

解决办法是将“外部数据更新时更新图表”解耦为“外部数据更新时更新内部 \_currentOption ” 和 ”当需要更新图表时调用 \_currentOption “两步，从而确保 html 加载完成前获取的数据也能被记录更新：

```
  String _currentOption;
  
  void init() async {
    await _controller?.evaluateJavascript('''
      chart.setOption($_currentOption, true);
    ''');
  }

  void update(String preOption) async {
    _currentOption = widget.option;
    ...
  }
  
  @override
  Widget build(BuildContext context) {
    return WebView(
      ...
      onPageFinished: (String url) {
        init();
      },
      ...
    );
  }
```

**内置信道**

webview_flutter 提供了 javascriptChannels 参数，可以设置多路命名信道。不过为了使不熟悉 webview_flutter 的使用者也能快速上手， flutter_echarts 并没有暴露这个参数来管理通信，而是内置建立了一个名为“ Messager ”的信道：

```
  @override
  Widget build(BuildContext context) {
    return WebView(
      ...
      javascriptChannels: <JavascriptChannel>[
        JavascriptChannel(
          name: 'Messager',
          onMessageReceived: (JavascriptMessage javascriptMessage) {
            widget?.onMessage(javascriptMessage.message);
          }
        ),
      ].toSet(),
    );
  }
```

使用者如果有多种事件需要通信，可以像 redux action 那样进行设置：

```
chart.on('click', (params) => {
  if(params.componentType === 'series') {
    Messager.postMessage(JSON.stringify({
      type: 'select',
      payload: params.dataIndex,
    }));
  }
});
```

# 展望

当然了，通过 WebView 借用 Echarts 进行数据可视化确实是权宜之计，在性能和兼容性上终究不能做到完美。

近年来，前端数据可视化也有了长足的进步， [图形语法(Grammar of Graphics)](https://www.springer.com/us/book/9780387245447) 的理念逐渐被引入到前端的数据可视中，诞生了像阿里的 [G2](https://g2.antv.vision/zh) ，微软的 [Chart Parts](https://microsoft.github.io/chart-parts/) 的实践。

后续我正在做一个基于图形语法的 Flutter 原生可视化库： [Graphic](https://github.com/entronad/graphic) ，欢迎关注。