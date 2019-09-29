在前端开发中，数值展示是一个常见的需求。不同于统计或实验报表等对精确性和规范性的注重，前端展示数值时更注重用户友好性，让人一眼能读出数字，且保持页面简洁、整齐。

无论是移动端的App，还是管理后台型的页面，或者流行的数据大屏，以下一些需求是展示数值时常常遇到的，具有一定的共性：

- 受布局空间的限制，数值字符串不能超过某个长度
- 不能出现null、NaN、undefined之类的情况，出现异常时希望有“--”之类的占位符
- 数字过大时希望加上千位分隔符1,234,222或加上单位表示1.23k
- 不要出现科学计数法1.23e+4
- 小数末尾不要有一串0

如果能有一个函数，自动的将输入的数值转换成符合要求的字符串就好了，特别是当长度有限时，能通过变换单位，压缩小数位的方式确保字符串不溢出长度限制，比如：

```
-254623933.876  =>  -254.62M
-1.2345e+5  =>  -123,450
12.000  => 12 
NaN  =>  --
```

Number Display就是为这样的需求而创建的：

```
display(-254623933.876)    // result: -254.62M
```

# 使用

本着配置和使用分离的原则，在2.*版本中number-display暴露的是名为`createDisplay`的高阶函数，通过传入参数给`createDisplay`定制组件中实际解析数值字符串的`display`函数：

```
import createDisplay from 'number-display';

const display = createDisplay({
  length: 8,
  decimal: 0,
});

<div>{display(data)}</div>
```

```
import 'package:number_display/number_display.dart';

final display = createDisplay(
  length: 8,
  decimal: 0,
);

print(display(data));
```

这样实际组件中实用`display`的代码就更为简洁，且方便批量配置。值得注意的是，为便于键值匹配和使用默认值，在JavaScript中配置项是对象的方式传入，Dart中是以命名参数的方式传入。

在2.*版本中number-display简化了配置项的数量，目前只包括以下5个：

- **length**: 数值字符串的长度限制。如果你要确保任何数值都可以被压缩到此范围内，你必须确保它大于等于5，这样最极端的情况（-123000）也能被压缩
- **decimal**: 最大小数位数。注意当空间不够时，实际小数位数会比这个值小，另外小数末尾的0会去掉
- **placeholder**: 当值为非法数字时输出的占位符
- **allowText**: 如果输入不是数字而是一段文本，是否原样输出
- **comma**: 是否显示千位分隔符。千位分隔符的意义见这里

# 限长压缩原则

数值在显示时，会受到组件尺寸的约束，理论上在组件设计尺寸时就必须考虑到值的范围，但实际不可能沟通的这么完美，保证数值不溢出组件往往还是落在开发的事上。当长度不够时，Number Display 按照以下优先级对数值进行压缩：

1. 去掉千位分隔符；
2. 按剩余空间减少小数位数；
3. 采用数值单位( k, M, G, T, P )压缩整数部分
4. 按以上原则，理论上任何数字都可以压缩到5个字符以内，如果设置的`length` 小于5并且数值无法压缩，将抛出异常

# 代码实现

Number Display 处理数值的过程分为以下几步

**1. 判断合法性**

对于JavaScript，由于其类型的灵(sui)活(yi)性，实际开发中传过来的“数值”的type往往既有可能是number，也有可能是string：

```
if (
  (type !== 'number' && type !== 'string')
  || (type === 'number' && !Number.isFinite(value))
) {
  return placeholder;
}
```

而对于静态类型语言Dart，我们限定输入参数类型为num，比较符合开发规范：

```
if (value == null || !value.isFinite) {
  return placeholder;
}
```



**2. 截取符号、整数、小数部分**

截取一个数值的符号、整数、小数三部分基本原理是依靠"-"和"."进行拆分匹配。借鉴Ant Design 的Statistic组件源码，我们利用JavaScript中String.match函数一次匹配多个模式并返回数组的特性，只用一行代码便可将三部分都获取到：

```
const cells = value.match(/^(-?)(\d*)(\.(\d+))?$/);
```

当然Dart语言就没有这种技巧了，三部分要分别匹配获取

```
final negative = RegExp(r'^-?').stringMatch(valueStr) ?? '';
final integer = RegExp(r'\d+').stringMatch(valueStr) ?? '';
final deciRaw = RegExp(r'(?<=\.)\d+$').stringMatch(valueStr) ?? '';
```

值得一提的是，Dart语言的正则部分正在逐步完善中，我注意到最近的几个版本更新（2.1-2.4）每次都有正则语法的改动。

**3. 压缩小数部分**

限长首先会去掉千位分隔符，当空间还不够时，就会按剩余空间缩减小数部分，注意由于小数点必须与小数部分共存亡，剩余空间需要以0和2两个边界分段：

（以下代码JavaScript和Dart基本类似，仅以JavaScript为例）

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

**4. 压缩整数部分**

如果完全去掉了小数部分空间仍然不够，就要尝试将整数部分用k、M等数值单位进行压缩了。之前我们曾给整数部分加上过千位分隔符：

```
const localeInt = int.replace(/\B(?=(\d{3})+(?!\d))/g, ',');
```

这里刚好可以通过分隔成了几段获知单位该是多少：

```
const sections = localeInt.split(',');
const units = ['k', 'M', 'G', 'T', 'P'];
const unit = units[sections.length - 2];
```

最后和压缩小数部分类似，我们要再根据剩余的空间确定转换数值单位后的小数位数：

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



https://github.com/ant-design/ant-design/blob/master/components/statistic/Number.tsx