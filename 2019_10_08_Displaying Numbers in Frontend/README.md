> How to display numbers friendly in frontend? In this article, we summarized some practical principles, introduced a tool library - Number Display, and analyzed it's source code.
>
> Number Display has both JavaScript version for Web, and Dart version for Flutter.
>
> [JavaScript version](https://github.com/entronad/number-display)
>
> [Dart version](https://github.com/entronad/number_display)

Displaying numbers is a very common requirement in frontend development. Unlike those in the statistic or scientific reports, which pay most attention to precision and standardization, numbers in frontend care more about friendly user experience. It's important to let the user perceive the number at first glance, and to keep the page clean and neat.

Weather in a mobile app, or in a backstage management system, or on a popular data visualization screen, displaying numbers has some common demands:

- Restricted by the geometric space, the number string can't overflow a certain length.
- strings like null, NaN or undefined are forbidden, there should better be a placeholder like "--" in exceptions.
- For big integers,  add commas as group separators (1,234,222) , or convert with units (1.23k).
- No scientific notations (1.23e+4).
- No trailing decimal zeros.

That will be really nice if there would be a tool function which, with only simple configurations, could convert input numbers to required strings. Especially when the geometric space is limited, it could automatically trim the number string by transforming the unit or reduce the decimal:

```
-254623933.876  =>  -254.62M
-1.2345e+5  =>  -123,450
12.000  => 12 
NaN  =>  --
```

Number Display was built for such demands:

```
rstStr = display(-254623933.876)    // result: -254.62M
```

# Usage

In principle of separating configuration and usage, Number Display exports a high order function named `createDisplay` . By passing parameters as configuration to `createDisplay` , we can custom the real `display` function used in widgets.

Usage in Web:

```
import createDisplay from 'number-display';

const display = createDisplay({
  length: 8,
  decimal: 0,
});

<div>{display(data)}</div>
```

Usage in Flutter:

```
import 'package:number_display/number_display.dart';

final display = createDisplay(
  length: 8,
  decimal: 0,
);

print(display(data));
```

This makes it quite concise to use  `display` , and possible to batch the configuration. Note that the configuration is passed as a object in JavaScript but named parameters in Dart.

Now Number Display includes 5 configurations:

- `length` : The max length the result would be. length should no less then 5 so that any number can display ( say -123000 ) after trim.
- `decimal` : The max decimal length. Note that this is only a constraint. The final precision will be calculated by length, and less than this param. There will be no decimal trailing zeros.
- `placeholder` : The result when the input is neither string nor number, or the input is NaN, Infinity or -Infinity. It will be sliced if longer than length param.
- `allowText` : Allow *Text* ( String that cant convert to number) as input and result. It will be sliced within length param. If false , result of text will be placeholder. Note that some special form will be regarded as text like 'NaN', '-1.2345e+5'. Note that the Dart version has no this param.
- `comma` : Whether the locale string has commas ( 1,234,222 ), if there are rooms.

# Trimming Principles

While displayed on the page, number strings are limited by the geometric space.

Theoretically, While designing the size of the widgets, it's value length should be considered. But actually, it can't be so comprehensive, and the response always falls to the frontend developers.

The form of the number can be flexible, but the string is absolutely unacceptable to overflow outside the widget. That is the highest priority. And that is the main function of Number Display. When the length is limit, Number Display will trim the number string by the following oder:

- omit the locale commas
- slice the decimal by the room left
- trim the integer with number units ( k, M, G, T, P )
- if the `length` is >= 5, any number can be trimmed within it. If it's less than 5 and input number is too long, display will throw an error.

# Source Code

Behind the hood, Number Display mainly uses regular expression to handle the number strings. Steps are as follows:

**1. Legal judgment**

For JavaScript, which a quite dynamic language, the input "number" can either be a Number or a String in practice:

```
if (
  (type !== 'number' && type !== 'string')
  || (type === 'number' && !Number.isFinite(value))
) {
  return placeholder;
}
```

While for static typed Dart, we announced the input type as num:

```
if (value == null || !value.isFinite) {
  return placeholder;
}
```

**2. Slicing the sign, integer and decimal parts**

The sign, integer and decimal parts are identified by the separator "-" and ".". Refered to the [code](https://github.com/ant-design/ant-design/blob/master/components/statistic/Number.tsx) of Ant Design Statistic component, with String.match function in JavaScript, we can match multiple patterns and get all the three parts as an array in just one line:

```
const cells = value.match(/^(-?)(\d*)(\.(\d+))?$/);
```

But in Dart we have no such trick:

```
final negative = RegExp(r'^-?').stringMatch(valueStr) ?? '';
final integer = RegExp(r'\d+').stringMatch(valueStr) ?? '';
final deciRaw = RegExp(r'(?<=\.)\d+$').stringMatch(valueStr) ?? '';
```

In fact the regular expression in dart is optimizing, we noticed that there are changes of regular expression in all recent versions (2.1-2.4) .

**3. Trimming the decimal part**

After omitting the comma separators, if the space is still limited, the decimal length will be reduced. Note since that the decimal point only appears when there were decimal part, there would be two boundaries, 0 and 2, for the length. Meanwhile, we will subtract the trailing zeros by the way:

(For the left part, JavaScript and Dart code are quite similar, so we only take JavaScript code for example)

```
let space = length - negative.length - int.length;
if (space >= 2) {
  deciShow = deci.slice(0, space - 1).replace(/0+$/, '');
  return `${negative}${int}${deciShow && '.'}${deciShow}`;
}
if (space >= 0) {
  return `${negative}${int}`;
}
```

**4. Trimming the integer part**

If the string still overflows, we have to convert the integer part with units like k or M. As we have added the group separator before:

```
const localeInt = int.replace(/\B(?=(\d{3})+(?!\d))/g, ',');
```

We can use that to calculate which unit we need:

```
const sections = localeInt.split(',');
const units = ['k', 'M', 'G', 'T', 'P'];
const unit = units[sections.length - 2];
```

At last, just the same as trimming the decimal part, reduce the trailing part to the room left:

```
space = length - negative.length - mainSection.length - 1;
if (space >= 2) {
  const tailShow = tailSection.slice(0, space - 1).replace(/0+$/, '');
  return `${negative}${mainSection}${tailShow && '.'}${tailShow}${unit}`;
}
  if (space >= 0) {
  return `${negative}${mainSection}${unit}`;
}
```

# Discussion

Displaying numbers is often ignored in the development, and there are no deep technologies in it indeed. But if we can't think thoroughly, there would be a lot of trouble.

The principles above seek more in common as possible. But there are always unique demands, so we tried to make it configurable. Number Display has experienced two major updates, and simplified the configurations. For sake of better performance, we changed some loops and judgments with regular expression operations. Hopefully that you could give some advice:

[Issues (JavaScript)](https://github.com/entronad/number-display/issues) 

[Issues (Dart)](https://github.com/entronad/number_display) 