> Dart 官方文档学习笔记，记录 Dart 特点和区别于其它语言之处

#基本特点

- 任何变量都是对象，包括基本类型，所有对象都继承自 Object 类
- 强类型，自带类型推断，但也可指定类型
- 有泛型
- 支持最外层函数定义，也支持类的静态方法与对象的实例方法，函数中可定义内部函数
- 支持最外层变量定义，也支持类的静态成员和对象字段
- 没有表示私有公用的关键字，以“_”开头的标识符表示库中私有
- 标识符以字母或“_”开头，后面可随意组合
- 语句末尾要加分号


#变量与类型


- 不指定类型定义变量的关键字是var，会根据初始化的值推断类型，但一旦确定后不可赋给其它类型的值
- 如需要定义**可变类型**的变量，可使用关键字dynamic或指定类型为Object
- 任何类型变量没有初始化，值都为null，表示“没有”的只有null
- 表示常量的关键字是final和const，他们可以代替var关键字或写在类型之前
- const是编译时常量，在编译时确定，不可定义为实例变量；final是运行时常量，在第一次使用时确定；一个常量是const类型也暗示必然是final类型
- const也可定义为其它const的算式结果，在类中定义需与static一起
- const可写在变量或构造函数前用来定义常值，常量表示引用关系不可变，常值表示值本身不可变
- 数值类型num，有两个子类int，double，int可进行按位运算，没有基本类型的概念，由于int，double都是num的子类，定义为num的变量可以在int、double间多态
- 数值转字符串用int.parse('1')，double.parse('1.1')，字符串转数值用1.toString()，3.14159.toStringAsFixed(2)
- 字符串字面量中可用\${}引用表达式，或直接\$引用标识符
- 两个字面相同的字符串相等
- 可用连续三个单引号或双引号表示多行字符串
- 字符串前加字母 r 则不处理转义
- 布尔类型bool，所有需要判断的地方只能传入严格的bool结果
- List类型，在定义时会推断泛型
- Map类型，key可为任意类型，在定义时会推断泛型，用{}字面量定义的会推断为Map类型
- 在用构造函数新建对象时，可不用new关键字
- 当用map[key]取值时，如没有此key，会返回null
- UTF-16字符用\uxxxx表示，其它长度编码的UTF字符（不是4位）要加{}：\u{xxxxx}
- 可通过\#获取标识符的Symbol，压缩会改变标识符的名称但不会改变它的Symbol


#函数


- 函数的返回值、参数可设定类型也可不设，无返回值的函数返回类型可设为void
- 只有一个表达式的函数可写成箭头函数
- 函数的可选参数分为命名参数和位置参数
- 命名参数的定义：

```
void enableFlags({bool bold, bool hidden}) {...}
```

- 命名参数的调用：

```
enableFlags(bold: true, hidden: false);
```

- 命名参数可用在参数很多很复杂的情况，定义时时，可通过@required确定其为必须的，需引入package:meta/meta.dart或package:flutter/material.dart包：

```
const Scrollbar({Key key, @required Widget child})
```

- 在函数定义时，形参表最后可用[]获取位置参数：

```
String say(String from, String msg, [String device]) {
  var result = '$from says $msg';
  if (device != null) {
    result = '$result with a $device';
  }
  return result;
}
```

- 可选参数可通过 = 设置默认值，不设置默认值为null
- 应用必须有一个main函数：

```
void main() {
  
}

或

void main(List<String> arguments) {
  
}
```

- 函数定义式（包括前面带上名字的箭头函数）相当于定义了一个闭包类型的变量，等价于定义并将一个匿名函数赋值给一个变量，但不可以同时定义命名函数又赋值给变量
- 作用域遵循词法作用域，作用域是静态确定的，函数可使用其定义时的变量
- 任何函数都有返回值，如为显示指明则返回null，故函数调用式一定是表达式



- 多元运算符的版本由最左边一个参数决定
- ~/：整除（向下取整）
- ==规则：如果涉及到null，必须两边都为null则返回true，否则返回false；如果不涉及null，返回左边参数的*x*.==(*y*)方法结果
- is；is!：判断一个对象是（不是）一个类的实现（包括子类）
- as：将一个对象强转为一个类，如果该对象为null或不是该类将报错
- ??=：如果为null就赋值，否则不动
- ^：按位异或
- ~：按位取反
- ??：条件运算符，如果左边不为null返回左边，否则返回右边
- ..：级联运算符，表示把左边对象中的右边成员取出进行某项操作再返回该对象
- ?.：条件获取成员，如果左边为null也不报错而是返回null


#结构语句

- else if中间分开一个空格
- 可以用forEach()和for-in遍历可迭代对象，比如Map，List
- switch分支比对的目标必须是编译时常量，被比较实例必须是同一类（不可以是子类），不可重写==



- 异常分为Exception和Error两类以及它们的子类
- 方法无需进行异常声明和检查
- 可以throw任意对象，当然不建议这样做
- throw式是一个表达式
- on指明捕捉的异常类型，catch指明异常参数，两者可配合使用，有了on可省略catch
- catch有第二个参数StackTrace
- 在catch块中，可用rethrow关键字继续抛出该异常
- 就算遇到未捕获的异常，也会先执行完finally中的异常再抛出


#面向对象


- 类只可以有一个父类，子类可使用任意级父类的内容
- 对于定义了常量构造函数的类，可用const关键字调用，调用相同常量构造函数（包括参数）构造的对象是同一个实例
- 在一个常量作用域内（比如一个常量Map）都是常量，不需要再重复写const了
- 对象类信息的字段是runtimeType
- 类中声明了初始值的字段，字段会在constructor之前初始化
- 在类中只有命名冲突时才需要this.xx
- 可以以如下的形式定义命名构造函数

```
Point.origin() {
    x = 0;
    y = 0;
  }
```

- 任何构造函数都不会被继承
- 如果没有特别指明，子类的构造函数中第一步会调用父类的默认构造函数
- 子类构造函数的执行顺序是：初始化字段表->父类构造函数->子类构造函数
- 定义了构造函数会覆盖掉默认构造函数，如父类没有默认构造函数，子类构造函数需在函数体大括号前用冒号手动指明调用父类构造函数

```
class Employee extends Person {
  // Person does not have a default constructor;
  // you must call super.fromJson(data).
  Employee.fromJson(Map data) : super.fromJson(data) {
    print('in Employee');
  }
}
```
- 构造函数大括号前还可用冒号指明初始化字段表，常用来设置final字段：

```
// Initializer list sets instance variables before
// the constructor body runs.
Point.fromJson(Map<String, num> json)
    : x = json['x'],
      y = json['y'] {
  print('In Point.fromJson(): ($x, $y)');
}
```

- 没有函数体，而是用冒号指向调用其他构造函数，称为重定向构造函数：

```
class Point {
  num x, y;

  // The main constructor for this class.
  Point(this.x, this.y);

  // Delegates to the main constructor.
  Point.alongXAxis(num x) : this(x, 0);
}
```

- 固定不变的对象可设为编译时常量，通过类定义中的常量构造函数实现：

```
class ImmutablePoint {
  static final ImmutablePoint origin =
      const ImmutablePoint(0, 0);

  final num x, y;

  const ImmutablePoint(this.x, this.y);
}
```

- 如果不希望构造函数总是新建一个实例，添加了factory关键字成为工厂构造函数，工厂构造函数不可访问this：

```
class Logger {
  final String name;
  bool mute = false;

  // _cache is library-private, thanks to
  // the _ in front of its name.
  static final Map<String, Logger> _cache =
      <String, Logger>{};

  factory Logger(String name) {
    if (_cache.containsKey(name)) {
      return _cache[name];
    } else {
      final logger = Logger._internal(name);
      _cache[name] = logger;
      return logger;
    }
  }

  Logger._internal(this.name);

  void log(String msg) {
    if (!mute) print(msg);
  }
}
```

- 所有实例的字段都有getter，非final的有setter，也可通过get、set关键字设置
- 没有定义函数体的方法称为抽象方法，抽象方法只可出现在抽象类中
- 通过abstract关键字可定义抽象方法，除非定义了工厂构造函数，抽象类不能被实例化
- 任何类都是一个接口，一个类要实现一个接口（类），用关键字impliments，可以实现多个接口
- 子类通过@override注解重写父类方法
- 可通过operator关键字重写运算符（重写 == 运算符需同时重写 hashCode的getter）
- 调用一个实例的未实现方法，将会报错，触发1.实例类型为dynamic；或2.实例定义了该方法（为抽象）同时重写了noSuchMethod()
- 枚举类型定义形式为：


```
enum Color { red, green, blue }
```

- 枚举类型中的每一个值都有一个index属性，枚举类的values属性可获得所有的值
- 不可以继承、混入、实现枚举类型，也不可以实例化枚举类型
- 创建mixin的要求是继承Object，不声明构造函数，不调用super
- 使用mixin在类定义后使用关键词with，和混入多个mixin
- 可用static关键字定义类变量和类方法，类变量在第一次使用时才会初始化；类方法不可使用this；对于公有功能，建议使用函数而不是类方法
- 除了限制类型，泛型还能起到指代类型，减少代码的作用：

```
abstract class Cache<T> {
  T getByKey(String key);
  void setByKey(String key, T value);
}
```

- 可在集合的字面量定义中使用泛型

```
var names = <String>['Seth', 'Kathy', 'Lars'];
var pages = <String, String>{
  'index.html': 'Homepage',
  'robots.txt': 'Hints for web robots',
  'humans.txt': 'We are people, not machines'
};
```

- 集合类型机制是reified（不同于Java的erasure），会在运行时保留

```
print(names is List<String>);
```

#异步处理

- 处理异步过程的类是Future和Stream，函数是async/await函数
- async关键字写在参数表与函数体之间
- 可将main函数定义为async函数
- 不需要返回值的async函数返回类型可设为Future\<void\>
- 可用await for等待遍历stream的所有值，但要小心使用：

```
await for (varOrType identifier in expression) {
  // Executes each time the stream emits a value.
}
```

- 同步generator用sync\*关键字定义，返回Iterable，异步generator用async\*关键字定义，返回Stream
- 递归generator函数中调用自身时可用yield*关键字提高性能

#其它

- 当给类实现了call()方法，类的实例就可以像函数一样被调用了


- 通过isolate处理多线程，isolate有单独的内存堆，相互之间不可访问
- 通过typedef关键字，可定义函数的具体（参数、返回值）类型：

```
typedef Compare<T> = int Function(T a, T b);

int sort(int a, int b) => a - b;

void main() {
  assert(sort is Compare<int>); // True!
}
```

- 文档级注释用///或/**开头，编译器会忽略文档级注释，但其中用[]括起来的内容仍然可以链接