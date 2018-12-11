> 源码地址：[entronad/crypto-es](https://github.com/entronad/crypto-es)

无论是前端还是后端，信息的加解密、摘要校验是常常碰到的需求，开发中一旦涉及到敏感数据，什么 MD5 、 Base64 、 AES 算法基本上都是要来上一套的。

在 JavaScript 的各种加密算法工具库中， [CryptoJS](https://code.google.com/archive/p/crypto-js/) 以其全面的功能、良好的通用性，一直是首选。它诞生较早，主仓库的代码还是托管在 Google Code 上，虽然后续也被移植到了 npm 上，持续有维护和更新，但由于历史原因，还是有几点在现在看来不太合时宜：

- 对象采用一套自己实现的原型继承（ prototypal inheritance ）系统
- 通过名为 Wordarray 的自定义类以 32 位整数的方式进行位操作

在 ES6 之前的年代里，这两点还是很巧妙和创新的，规避了 JavaScript 语言本身的缺点，同时也保证了浏览器兼容性和开箱即用。不过既然新的 ECMAScript 规范已经添加了类定义和 ArrayBuffer ，解决了原本的问题，我想尝试利用最新的 ECMAScript 特性对 CryptoJS 进行实验性的重写。

重写的项目定名为 [CryptoES](https://github.com/entronad/crypto-es) ，既然是实验性的，生产应用和兼容性等就不多作考虑，对使用场景的定义为：满足 ECMAScript 的最新标准（ 2018 ），比如模块采用 ECMAScript Module 而非 CommonJs ；成员变量定义还是提案就先不用。在 Babel 、 loader hook 的帮助下代码已经能在各种环境下了，未来随着 ES 规范的普及，可直接使用的场景会更多。

此外，既然是重写，还要保证所有使用接口不变。

# ECMAScript 类与继承

CryptoJS 在核心架构中扩展了 JavaScript 的原型链，自己实现了一套原型继承体系，具有 extend 、 override 、 mixin 等功能，使其比较符合通用的面向对象变成的习惯，可以说与 ECMAScript 类异曲同工，我们重写的第一步就是直接应用 ECMAScript 类与继承改造它，去除冗余和变扭之处，使其更符合规范。再这之前，先看一下 ECMAScript 类中我们会用到的一些关键点：

**constructor**

我们知道，ES6 中以 class 为关键字的类定义只是一个语法糖，它本质上还是一个构造函数，因此可以通过实例的 constructor 属性，获取该实例的类。这有什么用呢，我们可以在实例方法中通过 `new this.constructor()` 的方式创建一个和该实例属于同类的新实例。

**this**

与 JavaScript 传统的原型链继承不同，ECMAScript 类的继承是先调用父类的构造函数，添加父类实例对象的属性和方法到 this 上面，然后再调用子类的构造函数修改 this，这就使得我们在子类中定义的方法可以正确的添加和覆盖。

值得注意的是，实例方法中的 this 指向的是实例，而静态方法中的 this 则指向类，因此类定义中可以通过静态方法里的 `new this()`  实现工厂模式。

**super**

在类的定义中，加了括号的 super()  指父类的构造函数，不加括号的 super 类似 this ，在静态方法中指父类，在实例方法中指父类的原型对象，因此在子类重写实例方法中，可以通过 `super.overridedMethod.call(this)` 的方式先调用一下父类的该方法

**prototype 与  \_\_proto\_\_**

类本质上是构造函数，因此类有 prototype 属性指向类的原型对象。

类本身也是一个对象，它也有 \_\_proto\_\_ 属性，指向父类。而类的原型对象的 \_\_proto\_\_ 则指向父类的原型对象，这样就实现了原型链。

实例的 \_\_proto\_\_ 指向类的原型对象。

通过这些属性可以获取类或实例的继承关系。

# CryptoJS 类风格改造

CryptoJS 目前比较常用的 npm 版是托管在 GitHub 上的：[brix/crypto-js](https://github.com/brix/crypto-js)， [CryptoES](https://github.com/entronad/crypto-es) 也以此为参考基准。其核心的对象定义在 core.js 文件中。

CryptoJS 通过名为 Base 的对象实现原型继承：

```
var Base = C_lib.Base = (function () {
  return {
    extend: function (overrides) {
      // Spawn
      var subtype = create(this);

      // Augment
      if (overrides) {
        subtype.mixIn(overrides);
      }

      // Create default initializer
      if (!subtype.hasOwnProperty('init') || this.init === subtype.init) {
        subtype.init = function () {
        subtype.$super.init.apply(this, arguments);
      };
    }

    // Initializer's prototype is the subtype object
    subtype.init.prototype = subtype;

    // Reference supertype
    subtype.$super = this;

    return subtype;
    },

    create: function () {
      var instance = this.extend();
      instance.init.apply(instance, arguments);

      return instance;
    },

    init: function () {
    },
  };
}());
```

具体来说，继承是通过给调用“父”对象的 extend 方法传入需要重写的成员变量和方法，生成新的“子”对象。通过这些对象的 create 方法来返回真正的实例，而 init 方法则在实例创建时被调用，起到构造函数的作用。这种方式存在的缺点是不能像习惯的那样使用 new 关键字来新建实例，而且每个对象会在 $super 中递归的保存继承链中所有”父“对象的实例，信息冗余。

这些功能，通过 ECMAScript 类的 extend 关键字和构造函数就可以很好实现，无需额外的代码，且避免以上问题。不过为了保存使用接口不变，我们还是保留 create 作为一个静态方法，以便通过 `ClassName.create()` 的方式创建实例，同时使用 rest 运算符和解构赋值，将传递给 create 方法的参数赋给真正的构造函数：

```
export class Base {
  static create(...args) {
    return new this(...args);
  }
}
```

Base 中提供的一个基本功能是 mixin，在 CryptoJS 的原型继承体系中，类似 Base 这样的“类”对象和实例对象的界限并不明晰，mixin 是可以可通过“类”对象来调用的：

```
mixIn: function (properties) {
  for (var propertyName in properties) {
    if (properties.hasOwnProperty(propertyName)) {
      this[propertyName] = properties[propertyName];
    }
  }

  // IE won't copy toString using the loop above
  if (properties.hasOwnProperty('toString')) {
    this.toString = properties.toString;
  }
},
```

但从逻辑和实际的使用情况来看，mixin 应该是一个实例方法，其实就是实现了 Object.assign() 的功能：

```
mixIn(properties) {
  return Object.assign(this, properties);
}
```

最后是实例拷贝的功能，按照原型继承思路，它的实现是比较变扭的：

```
clone: function () {
  return this.init.prototype.extend(this);
}
```

我们按比较直白的方式来，通过 `new this.constructor()` 使其不需指明类名，适用于任意对象的实例：

```
clone() {
  const clone = new this.constructor();
  Object.assign(clone, this);
  return clone;
}
```

完成了 Base 类后，所需要的各种核心类都可以通过类继承的方式来获取这些基本方法。并且使用了 ECMAScript 类后，很多调用方式可以更为规范，比如原先 Wordarray 定义中构造新实例的两个方法：

```
var WordArray = C_lib.WordArray = Base.extend({        
  init: function (words, sigBytes) {
    words = this.words = words || [];

    if (sigBytes != undefined) {
      this.sigBytes = sigBytes;
    } else {
      this.sigBytes = words.length * 4;
    }
  },
  
  random: function (nBytes) {
    ...
    return new WordArray.init(words, nBytes);
  }
});
```

现在可以通过构造函数和类的静态方法实现：

```
export class WordArray extends Base {
  constructor(words = [], sigBytes = words.length * 4) {
    super();

    this.words = words;
    this.sigBytes = sigBytes;
  }

  static random(nBytes) {
    ...
    return new WordArray(words, nBytes);
  }
}
```

# 单元测试

在单元测试中，我们额外要测试一下类的继承关系是否正确实现了，即

- 子类的 \_\_proto\_\_ 指向父类
- 子类实例的原型对象与父类原型对象在原型链中是上下级

```
data.Obj = class Obj extends C.lib.Base {
};

data.obj = data.Obj.create();

it('class inheritance', () => {
  expect(data.Obj.__proto__).toBe(C.lib.Base);
});

it('object inheritance', () => {
  expect(data.obj.__proto__.__proto__).toBe(C.lib.Base.prototype);
});
```



> 题图：韩国电视剧《继承者们》