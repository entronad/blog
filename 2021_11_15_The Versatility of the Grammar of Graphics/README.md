> The new version of Flutter visualization library [Graphic](https://github.com/entronad/graphic) optimized its declarative specification grammar, so that it can better represent the nature of the Grammar of Graphics.
>
> In this article, we change the specification of a [Graphic](https://github.com/entronad/graphic) chart, thus transform a bar chart to a pie chart step by step. This work displays the flexibility and diversity of the Grammar of Graphics, and also show beginners the basic concepts of the Grammar of Graphics.
>
> If you have never learned the Grammar of Graphics before, it does not affect reading this article. It is also a starting handbook of [Graphic](https://github.com/entronad/graphic).

Bar charts and pie charts are both very common in data visualization. They look quite the different at first sight, but they have the same nature. Why? Let's transform a bar chart to a pie chart step by step, to look into the intrinsic reasons.

Let's start with a simple bar chart. The data is the same as the [starting example](https://echarts.apache.org/examples/editor.html?c=doc-example/getting-started) of ECharts:

```dart
const data = [
  {'category': 'Shirts', 'sales': 5},
  {'category': 'Cardigans', 'sales': 20},
  {'category': 'Chiffons', 'sales': 36},
  {'category': 'Pants', 'sales': 10},
  {'category': 'Heels', 'sales': 10},
  {'category': 'Socks', 'sales': 20},
];
```

# Declarative Specification

[Graphic](https://github.com/entronad/graphic) adopts a declarative specification. All grammars of visualization is in the constructor of the [Chart](https://pub.dev/documentation/graphic/latest/graphic/Chart-class.html) widget:

```dart
Chart(
  data: data,
  variables: {
    'category': Variable(
      accessor: (Map map) => map['category'] as String,
    ),
    'sales': Variable(
      accessor: (Map map) => map['sales'] as num,
    ),
  },
  elements: [IntervalElement()],
  axes: [
    Defaults.horizontalAxis,
    Defaults.verticalAxis,
  ],
)
```

# Data and Variable

The data of a chart is imported by the `data` property. It can be a list of any type. Inside the chart, the data items are converted to standard [Tuple](https://pub.dev/documentation/graphic/latest/graphic/Tuple.html) objects. How [Tuple](https://pub.dev/documentation/graphic/latest/graphic/Tuple.html) fields are extracted from data items is defined by [Variable](https://pub.dev/documentation/graphic/latest/graphic/Variable-class.html)s.

We can see from the example code that the grammar of the specification is concise, yet the `variables` definition takes half the length. Since Dart is a strict typed language, in order to allow any input datum type, detailed [Variable](https://pub.dev/documentation/graphic/latest/graphic/Variable-class.html) definitions are quite necessary.

# Geometry Element

The greatest idea of the Grammar of Graphics is indicating the difference between abstract *graph* and perceivable *graphic*.

For instance, whether a datum is an *interval* of values or a *point* of a value, is called the *graph*; while whether the item on canvas is a bar or a triangle, the width, and the height, is called the *graphic*. The steps to create a graph and a graphic are called geometry and aesthetic, respectively.

The concepts of graph and graphic reach the intrinsic relationship between data and graphics. They are the key for the Grammar of Graphics to get free from the constraints of chart typologies.

The [GeomElement](https://pub.dev/documentation/graphic/latest/graphic/GeomElement-class.html) is where these two concepts are defined. Its types determine the graph type, and they are:

- [PointElement](https://pub.dev/documentation/graphic/latest/graphic/PointElement-class.html): points.
- [LineElement](https://pub.dev/documentation/graphic/latest/graphic/LineElement-class.html): a line connecting points.
- [AreaElement](https://pub.dev/documentation/graphic/latest/graphic/AreaElement-class.html): an area between lines.
- [IntervalElement](https://pub.dev/documentation/graphic/latest/graphic/IntervalElement-class.html): intervals between two points.
- [PolygonElement](https://pub.dev/documentation/graphic/latest/graphic/PolygonElement-class.html): polygons partitioning a plane.

The height of a bar in a bar chart represent the interval between 0 and the datum value, so we choose [IntervalElement](https://pub.dev/documentation/graphic/latest/graphic/IntervalElement-class.html). Thus we get a very common **bar chart**:

![]()

Let's get back to the beginning quest. The angles of a pie chart also represent intervals, so we should also choose the [IntervalElement](https://pub.dev/documentation/graphic/latest/graphic/IntervalElement-class.html). But why a bar chart renders bars, while a pie chart renders sectors?

# Coordinate

A coordinate assigns variables into different dimensions in the plane. For rectangle coordinates ([RectCoord](https://pub.dev/documentation/graphic/latest/graphic/RectCoord-class.html)), the dimensions are horizontal and vertical; and for polar coordinates ([PolarCoord](https://pub.dev/documentation/graphic/latest/graphic/PolarCoord-class.html)), dimensions are angle and radius.

The example above doesn't indicate the `coord` property, so a default rectangle coordinate is set. Since the pie chart represent intervals with angles, it should be a polar coordinate. We add a line of definition to indicate that:

```dart
coord: PolarCoord()
```

Then the chart becomes a **rose chart**:

![]()

It seems getting close to a pie chart. But the graphic switched by "a single button" looks imperfect. It needs some fixing.

# Scale

The first problem is, the proportions of the sector radiuses seem not equal to the proportions of the `salse` values.

This problem revolves an important concept of the Grammar of Graphics: the [Scale](https://pub.dev/documentation/graphic/latest/graphic/Scale-class.html).

The original data values could be numbers, strings, or time. Even only of numbers, the values may range several orders of magnitude. So before used in the chart, they should be normalized. This step is called scaling.

The continuous data, such as numbers and time, should be normalized to `[0, 1]`; while the discrete data, such as strings, should be mapped to natural number indexes like 0, 1, 2, 3...

Every variable has a responsive scale, which is set in [Variable](https://pub.dev/documentation/graphic/latest/graphic/Variable-class.html)'s `scale` property. The variable values in [Tuple](https://pub.dev/documentation/graphic/latest/graphic/Tuple.html)s can be one of `num`, `DateTime`, or `String`, so the scale is classified by the input value types:

- [LinearScale](https://pub.dev/documentation/graphic/latest/graphic/LinearScale-class.html): normalizes a range of numbers to `[0, 1]`, continuous.
- [TimeScale](https://pub.dev/documentation/graphic/latest/graphic/TimeScale-class.html): normalizes a range of time to `[0, 1]` numbers, continuous.
- [OrdinalScale](https://pub.dev/documentation/graphic/latest/graphic/OrdinalScale-class.html): maps strings to natural number indexes in order, discrete.

