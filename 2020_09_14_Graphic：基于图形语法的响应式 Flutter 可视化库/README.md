背景

Flutter 随着自身的发展，逐渐被应用到较为大型的应用中，复杂的数据可视化图表日益成为一个重要的需求。虽然 Flutter 有强大的 Painter、Canvas 用于图形绘制，但不幸的是，目前 Flutter 生态圈中还没有功能强大又易用的杀手级可视化库。目前的 Flutter 可视化库都存在一些不尽如人意的地方，例如：

[charts_flutter](https://pub.dev/packages/charts_flutter) 是由 Google 内部的一个小组开发的可视化库，代码质量很高。但是它支持的图表种类很少，仅有最常见的线图、柱状图、饼图等几种，且没有曲线平滑等常用的功能。此外它似乎是一个实验性质的项目，没有任何的文档或介绍，虽然源代码放在了GitHub上，但明确表示不会处理任何的 issue 和 PR，似乎也好久没有更新特性了。

[fl_chart](https://pub.dev/packages/fl_chart) 是 Pub 上目前人气最高的可视化库，它有较为酷炫的设计和动画，是一位 [伊朗帅哥](https://github.com/imaNNeoFighT) 开发的。但是它的个人风格太明显了，不太适用于统计或严肃的场景。

[syncfusion_flutter_charts](https://pub.dev/packages/syncfusion_flutter_charts) 是一个较为完善和专业的可视化库，但它是一个闭源的商业软件，仅提供收费的使用许可，对于很多开发者来说并不适合。

...

为了构建复杂的可视化图表，我们开发过 [flutter_echarts](https://github.com/entronad/flutter_echarts) ，通过 Webview 的方式将 JS 的 Echarts 引入到 Flutter App 中，满足很多复杂的可视化需求，比如 3D 图表、GIS 可视化等，但是 Webview 在性能和兼容性等方面还是存在诸多问题，对于性能和稳定性要求高的一般图表，还是需要一个 Flutter 原生的可视化方案。



特性

图形语法（Grammar of Graphics）

配置式创建图表

可自定义的 Shape 函数

示例

展望