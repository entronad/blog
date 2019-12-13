对于一些功能比较复杂的函数，需要将很多配置项作为参数传入，这时候传统的位置参数表就不太方便了，因为对于配置项参数，我们往往会设置默认值，希望使用者无需按顺序传入所有参数，而只要指明哪几个参数需要特别配置。

比如一个简单的字符串格式化函数，除了必需的传入值 `value` 外，有三个配置项：

- `indent` ：缩进

- `caseMode` ：大小写

- `callback` ：转换完成后的回调

前端代码中应该如何定义呢？

---

我们先看看用 JavaScript 的情况。

如果我们按照最原始的定义方法：

```
function transform(value, indent, caseMode, callback) {
  ...
}
```

那么使用者就必须记住所有配置参数的位置，要是只想设置 `caseMode` ，需要写成如下尴尬的形式：

```
transform('someStr', undefined, 'upper')
```

事实上很多 JavaScript 内置函数就是采用的这种方式，使用者很难记住所有的位置参数，因此`[1, 2, 3].map(parseInt)` 才会成为“经典”面试题。

很多现代编程语言，比如 Dart 或 Python ，增加了“命名参数”这一特性，解决了参数与参数名对应的问题：

```
transform('someStr', caseMode: 'upper')
```

JavaScript 没有命名参数这一特性，早期实践中往往通过“配置对象参数“的方式，达到类似的效果：

```
function transform(value, cfg) {
  ...
  if(cfg.caseMode) ...
}
```

对象参数的缺点是源码中 `cfg` 的配置项不够直观，默认值设置比较麻烦，类型检查约束性不够。

现在配合上 ES6 的解构赋值和函数默认值，直观性和默认值可以解决了：

```
function transform(value, {
  indent = 2,
  caseMode = 'upper',
  callback,
} = {}) {
  ...
}
```

不过约束性还是不够，不能实现“必选参数”、“可选参数”的功能。

---

这时候就要 TypeScript 上场了。

除了类型检查外，TypeScript 可以对必选参数进行空值检查，然后再通过`?:` 手动设置可选参数：

```
function transform(value: string, {
  indent = 2,
  caseMode = 'upper',
  callback,
}:{
  indent?: number,
  caseMode?: string,
  callback?: (rst: string) => void,
} = {}) {
  ...
}
```

这就是 TypeScript 中比较完整的一个列子，实现了命名参数、可选参数，参数默认值的功能，使用时：

```
transform('someStr', { caseMode: 'upper' })
```

---

在以上的定义中，“命名参数”用上了 `{}: {}={}` 的结构进行定义，格式上要分成两段，另外由于本质上是个对象参数，外面还是不能忘了大括号的，写法略有些冗长，但是 TypeScript 的团队 [表示](https://github.com/Microsoft/TypeScript/issues/467) 由于保留位置限制、表达式解析等种种原因，目前不会加入真正的“命名参数”特性，因此目前只能这样写。