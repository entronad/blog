不管是前端还是后端开发，信息的加解密、摘要校验是常常碰到的一个需求，一旦涉及到登录接口、信息保存，什么MD5、Base64、AES算法基本上都是要来上一套的。

在JS的各种加密算法工具库中，[CryptoJS](https://code.google.com/archive/p/crypto-js/)以其全面的功能、良好的通用性，一直是首选。它诞生较早，主仓库的代码还是托管在Google Code上，虽然后续也被移植到了npm上，持续有维护和更新，但由于历史原因，还是有几点在现在看来不太合时宜：

- 对象采用了一套自己实现的继承系统
- 通过名为Wordarray的自定义类以32位整数的方式进行位操作

在ES6之前的年代里，这两点还是很巧妙和创新的，规避了JS语言本身的缺点，同时放在现在也保证了它的向下兼容性和开箱即用的特点。不过既然新的ES规范已经添加了class和ArrayBuffer，解决了原本的问题，我想尝试利用最新的ES特性对CryptoJS进行实验性的重写。

既然是实验性的，生产应用和兼容性等就不多作考虑，对使用场景的定义就一条：满足ES的最新标准（2018），比如模块采用ES Module而非CommonJs，但成员变量定义还是提案就先不用。这样当前的话，源码在Babel、loader hooker的帮助下已经能运行测试了，未来随着ES规范的普及可使用的场景会更多。此外，既然是重写，所以要保证所有使用接口不变。

# ES6 类与继承

CryptoJS在核心架构中利用并扩展了JS的原型链，自己实现了一套原型继承体系，具有了extend、override、mixin等功能，使其比较符合通用的面向对象变成的习惯，可以说与ES6中的类异曲同工，我们重写的第一步就是直接应用ES6的类与继承改造它，去除冗余和变扭之处，使其更符合规范。再这之前，先看一下ES6 class 与继承中我们接下去会用到的一些关键点：

**constructor**

我们知道，ES6中以class为关键字的类定义只是一个语法糖，它本质上还是一个构造函数，因此可以通过实例的constructor属性，获取该实例的类。这有什么用呢，我们可以在实例方法中通过 `new this.constructor()` 的方式创建一个和该实例属于同类的新实例。

**this**

与JS传统的原型链继承不同，ES6中类的继承是先调用父类的构造函数，添加父类实例对象的属性和方法到this上面，然后再调用子类的构造函数修改this，这就使得我们在子类中定义的方法可以正确的添加和覆盖。

值得注意的是，实例方法中的this指向的是实例，而静态方法中的this则指向类，因此类定义中可以通过`new this()` 的方式定义静态的工厂方法

**super**

在类的定义中，加了括号的super() 指父类的构造函数，不加括号的super类似this，在静态方法中指父类，在实例方法中指父类的原型对象，因此在子类重写实例方法中，可以通过`super.overridedMethod.call(this)`的方式先调用一下父类的该方法

**prototype 与  \_\_proto\_\_**

类本质上是构造函数，因此类有prototype属性指向类的原型对象，而类同时作为一个对象，它也有\_\_proto\_\_属性指向父类。类原型对象的\_\_proto\_\_指向父类的原型对象，实现原型链。实例的\_\_proto\_\_指向类的原型对象。通过这些属性可以获取类或实例的继承关系。

# CryptoJS 类风格改造

CryptoJS 目前比较常用的npm版是托管在GitHub上的：[brix/crypto-js](https://github.com/brix/crypto-js)，本文也以此为参考基准。其核心的对象定义在 core.js文件中。

它通过名为Base的对象实现所谓的原型继承：

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

具体来说，通过给调用“父”对象的extend方法传入需要重写的成员变量和方法来生成新的“子”对象实现继承，通过这些对象的create方法来返回真正的实例，而init方法则在实例创建时被调用，起到构造函数的作用。存在的缺点是不能像习惯的那样使用new来新建实例，而且每个对象会在$super中递归的保存继承链中所有”父“对象的实例，信息冗余。

这些功能，通过类的extend关键字和构造函数就可以很好实现，无需额外的代码且避免以上问题，不过为了保存使用接口不变，我们还是保留create作为一个静态方法，以便通过ClassName.create()的方式创建实例，同时使用rest运算符和解构赋值，将传递给create()的参数赋给真正的构造函数：

```
export class Base {
  static create(...args) {
    return new this(...args);
  }
}
```

Base中提供的一个基本功能是mixin，在CryptoJS的原型继承体系中，类似Base这样的“类”对象和实例对象的界限并不明晰，mixin可通过“类”对象来调用：

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

但从逻辑和实际的使用情况来看，这应该是一个实例方法，其实就是实现的Object.assign的功能：

```
mixIn(properties) {
  return Object.assign(this, properties);
}
```

最后是实例拷贝的功能，按照它的原型继承思路是比较变扭的：

```
clone: function () {
  return this.init.prototype.extend(this);
}
```

我们按比较直白的方式来，通过new this.constructor()使其适用于任意对象的实例：

```
clone() {
  const clone = new this.constructor();
  Object.assign(clone, this);
  return clone;
}
```

完成了Base类后，所需要的各种核心类都可以通过类继承的方式来获取这些基本方法。并且使用了ES6类的体系后，很多操作可以采用更为规范通用的方式，比如原先Wordarray定义中构造新实例的两个方法为：

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

- 子类的\_\_proto\_\_指向父类
- 原型链中，子类实例的原型对象在父类原型对象的下一级

本项目采用jest：

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



