> Introducing the development work of a reactive Echarts Flutter Widget: [flutter_echarts](https://github.com/entronad/flutter_echarts) .
>
> [repository](https://github.com/entronad/flutter_echarts) 
>
> [pub.dev](https://pub.dev/packages/flutter_echarts) 

With its rapid development, Flutter has been applied to more and more massive projects, and complex data visualization charts has been an important requirement. Although Flutter has powerful classes like Painter or Canvas to do the painting work, unfortunately, there is sill no killer data visualization library in the Flutter ecosystem.

Early this year, the Flutter dev team has published an official inline WebView widget: [webview_flutter](https://pub.dev/packages/webview_flutter) . It is based on the new  Platform View, which makes it possible to embed web contents seamlessly in Flutter, just like other widgets. Thus we can import those mature web data visualization libraries into our Flutter apps.

When talking about mature, powerful and easy-to-use data visualization libraries, [Echarts](https://www.echartsjs.com/zh/index.html) is defiantly a good choice. I would no more repeat its advantages here. If we can add Echarts to our Flutter apps, we can not only implement abundant chart types it supports, but also reuse the ready-made chart codes of the web to reduce the workload.

So we encapsulated a Flutter widget: [flutter_echarts](https://github.com/entronad/flutter_echarts) , taking into account both extensibility and ease to use, helping Flutter developers to fully play the Echarts' functions.

# Feature

Before this, we encapsulated an [Echarts component](https://github.com/entronad/react-native-echarts-demo) in React Native, and got some experiences about how to use data visualization libraries in a reactive UI framework, so when it comes to Flutter, we designed some features for flutter_echarts:

**Reactive Updating**

One of the most important features of Flutter is, like all other reactive UI frameworks, that it automatically update the view according to the change of data, which brings a lot of convenience for development. Echarts is independent of any UI framework, but it is designed driven by data, and changes in data drive changes in the chart.

So we just need to connect the data driving method of Echarts with the view updating of Flutter to implement the widget's reactive updating. It's very simple to set dynamic data updating in Echarts. All data updating are through `setOption` . You only need to get data as you wish, fill in data to `setOption` without considering the changes brought by data, ECharts will find the difference between two group of data and present the difference through proper animation.

Meanwhile, in Flutter, when the container widget updates and the data props passed to the child widget changes, the  `State.didUpdateWidget` of this StatefulWidget will be triggered. So calling `setOption` in it would notify Echarts to update the chart. This makes flutter_echarts as easy to use as a simple StatelessWidget.

**Two Way Communication**

The communication between the chart and the outer program is quite necessary. In flutter_echarts, the communication principle between JavaScript and Dart is just as that between father and child widgets: "props down, events up".

All settings and commands from outside is passed to chart by  `option` and `extraScript` and in form of JavaScript code strings. these codes will be executed by the WebView; On the other hand, the events inside the WebView are sent through JavascriptChannel and handled by the onMessage function. This is the two way communication between inside JavaScript and outside Dart.

**Configurable extensions**

Echarts has varies [extensions](https://echarts.apache.org/en/download-extension.html) , including charts, maps and WebGL. In the web ,we could import them as script files to extend the functions of Echarts. For out-of-the-box usage,  flutter_echarts embeded the newest version of Echarts script, no need of extra importing. Meanwhile, we exposed an `extensions` property for users to include any scripts needed.  `extensions` is a List of Strings, and users could directly copy the script strings into the source code, which avoids file reading and complex asset dirs.

# Widget Properties

While encapsulating widgets, ease to use is often more importent than completeness. It should able for all levels of developers to use out-of-the-box. Echarts it self is designed to the principle of ease, and try to put all configrations in `option` ([details in this paper](http://www.cad.zju.edu.cn/home/vagblog/VAG_Work/echarts.pdf)) . So flutter_echarts also simplified the widget properties:

**option**

*String*

The JavaScript Echarts Option for the chart as a string. The echarts is mainly configured by this property. You could use `jsonEncode()` function in dart:convert to convert data in Dart object form:

```
source: ${jsonEncode(_data1)},
```

Because JavaScript don't have `'''` , you can use this operator to reduce some escape operators for quotas:

```
Echarts(
  option: '''
  
    // option string
    
  ''',
),
```

**extraScript**

*String*

The JavaScript which will execute after the `Echarts.init()` and before any `chart.setOption()` . The widget has build a javascriptChennel named `Messager`, so you could use this identifier to send message from JavaScript to Flutter:

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

Function to handle the message sent by `Messager.postMessage()` in `extraScript` .

**extensions**

*List<String>*

List of strings that coyied from Echarts extensions, such as components, WebGL, languages, etc. You can download them [here](https://echarts.apache.org/en/download-extension.html) . Insert them as raw strings:

```
const liquidPlugin = r'''

  // copy from liquid.min.js

''';
```

---

These are the all 4 properties of flutter_echarts. Other things like when to update charts are made by inside mechanisms. This makes flutter_echarts looks just like a simple presentational StatelessWidget. Users just need to be famliar with Echarts and no additional learning.

A full example is here: [flutter_echarts_example](https://github.com/entronad/flutter_echarts/tree/master/example) .

And of course if you have any suggestions or demands, please give an [issue](https://github.com/entronad/flutter_echarts/issues) .

# Source Code Analysis

**loading html**

For cross platform developments, due to the deferent file systems of the OSs, there are always troubles to manage the asset dirs. In React Native sometimes you have even to copy the html file into the Android dirs manually. Flutter has a complete asset system, but it also need extra dependencies and configurations. So loading local htmls as text strings in the source code is a good idea, and the webview_flutter team also recommend this way in its' official examples.

So we put all template html, Echarts scripts, extensive scripts and initial codes into a string in the initialization of the widget, and load it as a uri source:

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

Note that as in the uri string , there are some limited chars, so we encode the string to Base64.

There's a tip: JavaScript dose not have  `'''` , so we can wrap our JavaScript strings with it to reduce some escaping work.

**updating charts**

The basic mechanism of reactive updating is to call `setOption` in the State.didUpdateWidget hook to notify the chart updating:

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

The most troublesome part is in the initialization of the widget.

We know that both the loading of html in the WebView and the fetching of data are asynchronous, and we do not  know which will finish earlier. the order of lifecycles in the initialization of WebView is:

```
onWebViewCreated --> loading html --> onPageFinished
```

And WebViewController can only be reached in onWebViewCreated. In another word, when the widgetd get a WebViewController, we can not tell whether the html has already been loaded, so in the `didUpdateWidget` , we can not tell whether it's ready to update by testing the WebViewController.

Our solution is to decouple "data props changing triggers chart updating" into two steps: "data props changing causes \_currentOption changing" and "update the chart according to \_currentOption", which makes sure that any data are record, even before the html is loaded.

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

**message channel**

The webview_flutter provides a javascriptChannels property to set multiple named channels. But considering the users who don't know about webview_flutter, flutter_echarts dose not expose this property. Instead, we build only one channel named "Messager":

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

If multiple types of events need to be sent, users can create actions like in redux:

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

