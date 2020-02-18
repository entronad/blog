> This article records a performance optimization of the WebView based Flutter data visualization library: [echarts_flutter](https://github.com/entronad/flutter_echarts) .

For any widgets based on WebView, the loading of pages is always a crucial part of performance. [echarts_flutter](https://github.com/entronad/flutter_echarts), whose foundation is to render local pages of [echarts](https://echarts.apache.org/en/index.html) with WebView, is no exception.

The contents to load of [echarts_flutter](https://github.com/entronad/flutter_echarts) can be divided into these parts:

- template HTML
- echarts script
- echarts extension scripts
- logic code of the chart

The template HTML and the logic code of the chart is rather small, so the key point is the loading of echarts script and echarts extension scripts.

One of echarts' best features is that it has many awesome extensions, such as WebGL 3D charts, GIS maps, etc. As data visualization requirements getting more and more complex, these extensions have become no less important than echarts itself. So it is a must to allow users to import extensions conveniently. Besides, to avoid troublesome assets management, we hope to handle both HTML and JavaScript as strings, thus the WebView will load all sources as URI.

There would be some questions:

- Should the scripts be inside the HTML or injected afterwards?
- The URI has some char limits, it needs a safe encoding form.

# Original Approach

In the beginning, we thought that, in general idea, we'd better put all things in the HTML and load them together. Considering there are a lot of illegal URI chars in JavaScript, we should convert the HTML into Base64 after composing. For we don't know what extension scripts the user will import, the encoding will be executed by functions dynamically:

```
String _getHtml(
  String echartsScript,
  List<String> extensions,
  String extraScript,
) {
  ... // Compose and return all HTML and scripts
}


  @override
  void initState() {
    super.initState();
    // Convert to Base64 in init
    _htmlBase64 = 'data:text/html;base64,' + base64Encode(
      const Utf8Encoder().convert(_getHtml(
        echartsScript,
        widget.extensions ?? [],
        widget.extraScript ?? '',
      ))
    );
    _currentOption = widget.option;
  }


  @override
  Widget build(BuildContext context) {
    return WebView(
      // Load all of them
      initialUrl: _htmlBase64,
      ...
    );
  }
```

# Performance Test

Let's take a simple performance test for feather analyses. The test case has three charts, including a WebGL 3D chart and a liquid animation chart:

![example](example)

With the Flutter Dev Tool, we can get the flame chart of CPU time occupation:

![1](1)

# Optimization

Echarts and it's extensions are of large volumes. So it will take a lot of time to compose and convert the strings in runtime. But these are necessary steps to get legal URI strings, so how to solve this problem?

How about abandon the idea "load everything together", and inject the dynamic part by `evaluateJavascript` and only put the static part in HTML? this may save some converting work.

To make sure of the feasibility, let's take an experiment first: only move out all scripts from HTML and inject them with `evaluateJavascript`, and check the performance:

```
  @override
  void initState() {
    super.initState();
    _htmlBase64 = 'data:text/html;base64,' + base64Encode(
      const Utf8Encoder().convert(_getHtml(
        // remove all scripts form the convert function
        // echartsScript,
        // widget.extensions ?? [],
        // widget.extraScript ?? '',
      ))
    );
    _currentOption = widget.option;
  }
  
  
  void init() async {
    final extensionsStr = this.widget.extensions.length > 0
    ? this.widget.extensions.reduce(
        (value, element) => (value ?? '') + '\n' + (element ?? '')
      )
    : '';
    await _controller?.evaluateJavascript('''
      // inject after the page is loaded
      $echartsScript
      $extensionsStr
      const chart = echarts.init(document.getElementById('chart'), null);
      ${this.widget.extraScript}
      chart.setOption($_currentOption, true);
    ''');
  }
```

The result is:

![2](2)

We can see that the time of loading HTML is reduced, while the time of `onPageFinished`, which contains the injection of scripts grew. The total time is reduced.

So it seems that converting large strings is quite costing. Using `evaluateJavascript` instead is a right way.

So we then remove all the dynamic converting part, and load template HTML as a const string. Since the HTML is static and short now, we can escape the illegal chars manually and input UTF-8 string directly, which needs no dart:convert library and looks more plain:

```
const htmlUtf8 = 'data:text/html;UTF-8,<!DOCTYPE html><html><head><meta charset="utf-8"><style type="text/css">body,html,%23chart{height: 100%;width: 100%;margin: 0px;}div {-webkit-tap-highlight-color:rgba(255,255,255,0);}</style></head><body><div id="chart" /></body></html>';


  @override
  void initState() {
    super.initState();
    _currentOption = widget.option;
  }
  
  @override
  Widget build(BuildContext context) {
    return WebView(
      initialUrl: htmlUtf8,
      ...
    );
  }
```

The test result is:

![3](3)

We can see that time is further reduced, especially in loading.

Thus, compared to the original version, the performance improved a lot.



Echarts script is also static, what if we convert it previously and put it in the HTML:

```
const echartsHtmlBase64 = '...';

  
  @override
  Widget build(BuildContext context) {
    return WebView(
      initialUrl: echartsHtmlBase64,
      ...
    );
  }

```

The result is:

![4](4)

On the contrary, it takes more time.

So we can see that "putting scripts in HTML" is not necessarily better than "injecting by `evaluateJavascript`", and even takes more time for some encoding reasons.

 # Conclusion

In summary, the final optimization solution is: load template HTML in UTF-8 URI string and inject all scripts and logic code with `evaluateJavascript` .



---

*Note: the webview_flutter has an issue that `onPageFinished` won't work in IOS, so the optimization above has not applied to release version for now. You can see the source code of it in [this commit](https://github.com/entronad/flutter_echarts/tree/db0a452b5f6652d2b9070aa0daeae995da13cb3e) .*

