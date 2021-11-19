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

For numbers, The default [LinearScale](https://pub.dev/documentation/graphic/latest/graphic/LinearScale-class.html) will determine the range by input data, so the range minimum may not be 0. For a bar chart, this makes the chart focus on height differences of bars. But it is not fit for the rose chart, because people tend to regard radius ratios as value ratios.

So, the range minimum of [LinearScale](https://pub.dev/documentation/graphic/latest/graphic/LinearScale-class.html) should be set to 0 manually:

```dart
'sales': Variable(
  accessor: (Map map) => map['sales'] as num,
  scale: LinearScale(min: 0),
),
```

# Aesthetic Attribute

The second problem is, the sectors are adjacent so that their colors should be distinguishable. And people prefer to use labels, not axis, to annotate the rose chart.

Attributes for perceiving graphics, like color or label, are called aesthetic attributes. In [Graphic](https://github.com/entronad/graphic) they are:

- `position`
- `shape`
- `color`
- `gradient`
- `elevation`
- `label`
- `size`

Except `position`, each of them are defined in [GeomElement](https://pub.dev/documentation/graphic/latest/graphic/GeomElement-class.html) by corresponding [Attr](https://pub.dev/documentation/graphic/latest/graphic/Attr-class.html) class. According to definition properties, they can be specified in these ways:

- Indicates directly by `value`.
- Indicates corresponding `variable`, and target attribute `values` and `stopes`. The variable values will be interpolated or mapped to attribute values. These kind of attributes are called [ChannelAttr](https://pub.dev/documentation/graphic/latest/graphic/ChannelAttr-class.html)s.
- Indicates how a tuple is encoded to an attribute value by `encoder`.

In this example, we specify colors and labels for every sectors:

```dart
elements: [IntervalElement(
  color: ColorAttr(
    variable: 'category',
    values: Defaults.colors10,
  ),
  label: LabelAttr(
    encoder: (tuple) => Label(
      tuple['category'].toString(),
    ),
  ),
)]
```

Thus we get a better rose chart:

![]()

But how to switch the rose chart to a pie chart?

# Transpose Coordinate

Variables of data often have a function relation: `y = f(x)`. We call the x is in the domain dimension, and the y is in the measure dimension. Customarily, for a plane, the rectangle coordinate assigns domain dimension to the horizontal and measure dimension to the vertical; while the polar coordinate assigns domain dimension to the angle and measure dimension to the radius.

A rose pie displays values with radiuses, while a pie chart displays values with angles. So the first step is to switch the correspondence of dimensions. This is called transposing:

```dart
coord: PolarCoord(transposed: true)
```

Then the graphics transform to a **racing chart**:

![]()

It seems to get closer to a pie chart.

# Variable Transform

The sectors in a pie chart compose a whole circle, the ratios of arcs to the perimeter is the ratios of values to the sum. But the sum of arcs of the above chart is obviously larger than the perimeter.

One solution is to set the scale range of `sales` between 0 and the sum of all `sales`s, then the scaled `sales` values are the ratios to the sum. But for dynamic data, we usually don't know the values when defining the chart.

Another solution is that if the measure dimension variable is the proportion of `sales`, then we only need to set the scale range to `[0, 1]`.

That is why we need [VariableTransform](https://pub.dev/documentation/graphic/latest/graphic/VariableTransform-class.html). It can apply statistical transforms to current variables, modify the tuples or create new variables. Here we use [Proportion](https://pub.dev/documentation/graphic/latest/graphic/Proportion-class.html), which calculates the proportions of `sales` values and assign them to a new `percent` variable which has a scale of `[0, 1]` range:

```dart
transforms: [
  Proportion(
    variable: 'sales',
    as: 'percent',
  ),
]
```

# Graphics Algebra

A new problem occurs after we applied the transform. The tuple had only two variables `category` and `sales` before, and they happens can be assigned to the two dimensions respectively. Nothing need to set. But now, an additional variable `percent` is added. How to assign three chestnuts to two monkeys? There needs a clear specification.

To define the relation between variables and dimensions, we need the graphics algebra.

The graphics algebra specifies the variables relations and how they are assigned to dimensions with an expression that connects [Varset](https://pub.dev/documentation/graphic/latest/graphic/Varset-class.html)s (variable set) with operators. There are tree operators:

- `*`: cross, which assigns two operands to two dimensions in order.
- `+`: blend, which assigns two operands to a same dimension in order.
- `/`: nest, group all tuples according to the right operand.

We need to assign `category` and transformed `percent` to domain dimension and measure dimension respectively. Benefited form the operator overriding of Dart, [Graphic](https://github.com/entronad/graphic) implements all graphics algebra by the [Varset](https://pub.dev/documentation/graphic/latest/graphic/Varset-class.html) class. So we define `position` as:

```dart
position: Varset('category') * Varset('percent')
```

After variable transform and graphics algebra are set, the graphics become:

![]()

# Grouping and Modifier

The arc length of sectors are settled, then we should "splice" them. The first step of splicing is to adjust their positions to end to end.

This position adjusting is specified by [Modifier](https://pub.dev/documentation/graphic/latest/graphic/Modifier-class.html)s. The object of the adjusting is not single tuples, but tuple groups. So we should group the tuples by `category`. Thus for the example data, each group will have a single tuple. The grouping is specified by nest operator of the graphics algebra. Then we can set the [StackModifier](https://pub.dev/documentation/graphic/latest/graphic/StackModifier-class.html):

```dart
elements: [IntervalElement(
  ...
  position: Varset('category') * Varset('percent') / Varset('category'),
  modifiers: [StackModifier()],
)]
```

Since we have made the total arc length equals to the perimeter, the sectors become end to end after stacked, which can be regarded as a **sunrise chart**.

![]()

# Coordinate Dimensions

Since the angles of sectors are in position, there needs only one final step: to inflate the radiuses so that the sectors make a hole pie.

Let's look into the radius dimension. We have just assigned the `category` variable to it by algebra, so the sectors fall into different "tracks" respectively. But in fact, we don't want they differ in radius dimension and only vary in angles. In another word, we prefer the polar coordinate is a 1D coordinate.

We just need to indicate the coordinate dimension count to 1, and remove `category` form the algebra expression:

```dart
coord: PolarCoord(
  transposed: true,
  dimCount: 1,
)
...
position: Varset('percent') / Varset('category')
```

Then the sectors inflates the circle radius, and we finished the **pie chart**:

![]()

---

The complete specification is:

```D
Chart(
  data: data,
  variables: {
    'category': Variable(
      accessor: (Map map) => map['category'] as String,
    ),
    'sales': Variable(
      accessor: (Map map) => map['sales'] as num,
      scale: LinearScale(min: 0),
    ),
  },
  transforms: [
    Proportion(
      variable: 'sales',
      as: 'percent',
    ),
  ],
  elements: [IntervalElement(
    position: Varset('percent') / Varset('category'),
    groupBy: 'category',
    modifiers: [StackModifier()],
    color: ColorAttr(
      variable: 'category',
      values: Defaults.colors10,
    ),
    label: LabelAttr(
      encoder: (tuple) => Label(
        tuple['category'].toString(),
        LabelStyle(Defaults.runeStyle),
      ),
    ),
  )],
  coord: PolarCoord(
    transposed: true,
    dimCount: 1,
  ),
)
```

In this process, we transformed the graphics incessantly by changing the specifications such as the coordinate, scales, aesthetic attributes, variable transforms, and modifiers. And we got a bar chart, a rose chart, a racing chart, a sunrise chart, and a pie chart of traditional chart typologies.

![]()

We can conclude that the Grammar of Graphics jumps out of the constraint of traditional chart typologies, and can generates more visualization graphics with better flexibility and extensibility. More importantly, It reveals the intrinsic relations of different visualization graphics, and provides a theory foundation for data visualization science.

