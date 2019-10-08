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

