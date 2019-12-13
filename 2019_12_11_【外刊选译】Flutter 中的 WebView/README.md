> 原文：[The Power of WebViews in Flutter](https://medium.com/flutter/the-power-of-webviews-in-flutter-a56234b57df2) 
>
> 一年前调研的时候，发现 Flutter 还没有内联 WebView 这样一个很关键的组件。当时记得在一个盖的很高的 issue 里，开发组表示为了多端表现一致，他们很想让 Flutter 自带一个统一的 web 解析模块，而不甘心仅仅是分别调用 ios 和 Android 各自原生的 WebView，毫无疑问这个技术方案遇到很大的困难。现在看来是梦想向现实妥协了，现在官方推出的内联 WebView 是以插件的形式调用原生组件。

你是否想要在 app 开发这样的功能：无需打开手机自带的浏览器就可以展示网页？或者你已经在网站上实现了一套安全的支付流程，不想再在移动端重写一遍——毕竟付款是个敏感的业务，你可不想一半的钱最后都意外落到“保护[海怪](https://en.wikipedia.org/wiki/Kraken)基金会”了。只有我一个人这样想吗？终于， Flutter 团队创建了一个 [非常好用的插件](https://pub.dev/packages/webview_flutter) ，让你能在 app 中引入 WebView 实现这些功能。

“这些功能”我指的是在 Flutter app 中展示网页……不是指保护海怪。

# Flutter WebView 和其它 Widget 一样

在你的 app 中引入 WebView 插件非常简单。它用起来就和其它的 Widget 一样：`WebView(initialUrl: ‘https://flutter.io')` 。你也可以通过 `javascriptMode` 参数启用或禁用 JavaScript。默认情况下你的 WebView 中的 JavaScript 是禁用的，所以要想启用的话你需要这样创建 WebView ：

```
WebView(
  initialUrl: 'https://flutter.io',
  javascriptMode: JavascriptMode.unrestricted,
)
```

几乎所有获取 WebView 信息以及控制 WebView 的功能是通过这样东西实现的： WebViewController 。当 WebView 完全创建好后，它通过一个回调函数返回：

```
WebViewController _controller;
WebView(
  initialUrl: 'https://flutter.io',
  onWebViewCreated: (WebViewController webViewController) {
    _controller = webViewController;
  },
);
//...later on, probably in response to some event:
_controller.loadUrl('http://dartlang.org/');
```

WebViewController 是你通过 Flutter 程序修改 WebView 或者获取其参数（比如当前 URL ）的门票。为了说明这实践上是怎样操作的，我写了一个简单的维基百科浏览 app ，它能让你记录并查看书签，这样完美主义者就永远不会因为忘了上次看过某篇文章而掉进“ [维基兔子洞](https://en.wikipedia.org/wiki/Wiki_rabbit_hole) ”了。

![维基百科浏览 app ，通过 Flutter WebView 实现。可以点赞并收藏文章以便日后查看](1.gif)

Wiki-rabbit-hole-browser 完整的代码参见 [GitHub](https://github.com/efortuna/wiki_browser) 。

WebView 和其它所有的 Flutter Widget 一样，上面可以添加覆盖其它 Widget。值得注意的是点赞按钮就是一个普通的 `FloatingActionButton` 悬浮在 WebView 上面，它有常见的悬浮阴影效果。另外，当app bar上的下拉菜单打开时，它会和对其它 Widget 一样部分遮住 WebView。

如果你看代码，会发现在示例中，我非常多的使用 `Compliter` 类和 `FutureBuilder` 。将 `_controller` 实例变量声明为 Completer 类似于为 WebViewController 设置了一个占位符。我们可以通过调用 _controller.isCompleted （意思是当我们有可用的 WebViewController 时状态为“完成”）或者 [通过 controller.future 使用 FutureBuilder](https://github.com/efortuna/wiki_browser/blob/master/lib/main.dart#L40) 来检查是否有可用的 WebViewController 。使用 FutureBuilder 能让我们创建的 UI 组件比如 FloatingActionButton 仅在有可用的 WebViewController 时添加点赞（否则的话程序在保存点赞时将无法得到 `currentUrl` ）。

WebView 的另外两个特性稍微有点复杂，所以我们将在下面两部分详细看一下。

---

# WebView 也可以捕获特殊的手势

作为 Flutter Widget，WebView也可以参与到 Flutter 的手势消歧协议（ [被称为 Gesture Arena](https://flutter.dev/docs/development/ui/advanced/gestures#gesture-disambiguation) ）中。默认情况下，WebView 只会对没有被其它 widget 声明的手势做出响应。不过你也可以通过指定 `gestureRecognizers` 来使它主动声明一个手势。

如果你的 WebView 在其它会对手势做出反应的 Widget 中，比如一个 ListView，你可能想要指定 app 如何对手势做出响应。当用户手指拖动屏幕时，是应该滚动 ListView 还是 WebView ？如果你希望两个 widget 都能滚动，WebView widget 可以 “捕获” 拖动的手势，这样当用户拖动 WebView 时它会滚动，而不是 ListView 滚动。你可以通过 `gestureRecognizers` 参数指定哪些手势传递给 WebView widget 。这个参数为所有你想捕获的 GestureRecognizer 的 Set 。不要被那个工厂对象吓着了，它只不过是一个美化的 builder 方法。要捕获垂直滚动事件，可以这样写：

```
WebView(
  initialUrl: someUrl,
  gestureRecognizers: Set()
    ..add(Factory<VerticalDragGestureRecognizer>(
      () => VerticalDragGestureRecognizer())),
)
```

或者：

```
var verticalGestures = Factory<VerticalDragGestureRecognizer>(
  () => VerticalDragGestureRecognizer());
var gestureSet = Set.from([verticalGestures]);
return WebView(
  initialUrl: someUrl,
  gestureRecognizers: gestureSet,
);
```

如果你完整的看过 [Boring Flutter Development Show](https://www.youtube.com/playlist?list=PLOU2XLYxmsIK0r_D-zWcmJ1plIcDNnRkK) ，你可能