> In this article we introduced a Flutter data visualization library based on Grammar of Graphics: [Grapphic](https://github.com/entronad/graphic)
>
> [Repository](https://github.com/entronad/graphic)
>
> [pub.dev](https://pub.dev/packages/graphic)

# Background

Data visualization is a common but important part of application development. A good visualization library always makes it easier to build data visualization charts. But unfortunately, there is not yet a perfect visualization library in Flutter community. The current candidates all have some unsatisfactoriness, such as:

- [charts_flutter](https://pub.dev/packages/charts_flutter) is developed by Google, and has very fine code quality. But it provides very limited chart categories. It lacks some commonly used features like smooth lines. Besides, it seems to be an experimental projects witch has no documents or manuals. Although it's code is uploaded to GitHub, it is not open to any issue or PR. It has no new features updated for a long time.

- [fl_chart](https://pub.dev/packages/fl_chart) is the most popular visualization library on pub.dev. It has cool designs and animations. It is developed by an [Iran handsome](https://github.com/imaNNeoFighT) . But it has too much personal style for some statistics or serious situations.

- [syncfusion_flutter_charts](https://pub.dev/packages/syncfusion_flutter_charts) is very complete and professional. But it's a commercial product and not open sourced. It needs paid licenses. That's not suitable for many developers.

To build complex visualization charts, we developed [flutter_echarts](https://github.com/entronad/flutter_echarts) , Witch imports Echarts (A powerful Web visualization library) into Flutter Apps by Webview. It did help to satisfy some complex demands, but Webview has some weaknesses in performance and compatibility. For fluency and stability, we still need a native Flutter data visualization library.



In recent years, data visualization in frontend development has progressed a lot. Grammar of Graphics theory has been applied to web visualization libraries. this theory is proposed by Leland Wilkinson in the book *The Grammar of Graphics*. It tries to describe grammar rules for all statistic graphics.

Thanks to this theory, the visualization library will no longer be limited to the traditional charts categories like line/bar/pie. Data visualization is abstracted to combination of geometry marks, coordinates, and scales, witch highly increased flexibility and extensibility. Frontend visualization libraries based on Grammar of Graphics are [vega](http://vega.github.io/) , [AntV](https://antv.vision/en) of Alipay, and [chart-part](https://microsoft.github.io/chart-parts/) of Microsoft.



So, there comes [Grapphic](https://github.com/entronad/graphic) : a Flutter data visualization library based on Grammar of Graphics.

# Example

A basic example of Graphic:

```dart
graphic.Chart(
  data: [
    { 'genre': 'Sports', 'sold': 275 },
    { 'genre': 'Strategy', 'sold': 115 },
    { 'genre': 'Action', 'sold': 120 },
    { 'genre': 'Shooter', 'sold': 350 },
    { 'genre': 'Other', 'sold': 150 },
  ],
  scales: {
    'genre': graphic.CatScale(
      accessor: (map) => map['genre'] as String,
    ),
    'sold': graphic.NumScale(
      accessor: (map) => map['sold'] as num,
      nice: true,
    )
  },
  geoms: [graphic.IntervalGeom(
    position: graphic.PositionAttr(field: 'genre*sold'),
    shape: graphic.ShapeAttr(values: [
      graphic.Shapes.rrectInterval(radius: Radius.circular(5))
    ]),
  )],
  axes: {
    'genre': graphic.Defaults.horizontalAxis,
    'sold': graphic.Defaults.verticalAxis,
  },
)
```



More examples please see in [Example App](https://github.com/entronad/graphic/tree/master/example) :

# Features

**Grammar of Graphics**

Data visualization can be roughly summarized to: Mapping data to visual channel attribute(color, shape, size, position...) values by a certain rule, and then render shapes with these attributes. In Grammar of Graphics, this progress can be concrete to:

In Graphic, the design of APIs and class names are mainly refered to AntV. The core concepts are:

- **Geom**: Geometry marks, or shapes to compose the chart.
- **Coord**: Coordinates of the chart, including two types: Cartesian and Polar.
- **Scale**: Rules to scale data into [0, 1], for the usage to map to visual channel attributes.
- **Attr**: Visual channel attributes, including color, shape, size, position... The value is decided by scaled data. they are applied to geometry marks.

These concepts are familiar for those who has already known about Grammar of Graphics. If you have never met them, please refer to the book. We think that Grammar of Graphics is an important part of data visualization, so weather you will use related libraries or not , it worth learning.

**Declarative Chart**

Most visualization libraries in Web is imperative, and plot charts by calling a series of functions. AntV is not an exception. But in Flutter, declarative and reactive views are advocated. The constructor of view is configured by the tree returned in build function. It's worth mentioning that this is also a important deference between Web Canvas and Flutter CustomPaint Widget.

So Graphic APIs is designed declarative. All information are configured by props of graphic.Chart Widget. The chart will rerender when the widget tree updates. By the way, we will optimize the rerender performance in the future.

**Custom Shape**

Why there can be all kinds of various charts? the key is the diversity of shape attribute. so it is the most complex one in all visual channel attributes. The customizability of shape attribute determines the extensibility of visualization libraries.

In Graphic, the shape attribute is defined to functions or high order functions. It receives values of visual channel attributes of geometry marks, and returns the composed render shapes for the render engine. Users can custom their own shape functions:

```
List<graphic.RenderShape> triangleInterval(
  List<graphic.AttrValueRecord> attrValueRecords,
  graphic.CoordComponent coord,
  Offset origin,
) {
  // Implementation of attrValueRecords => renderShapes
}
```

# TODO

Graphic has now mainly finished static charts. We will add features such as interactions, animations, and components like tooltip, legend in the future.

The features and APIs may be unstable for now. So you are welcome for trial, but be cautious to use it in production environment.