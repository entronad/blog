> Repository: [entronad/crypto-es](https://github.com/entronad/crypto-es)
>
> npm: [crypto-es](https://www.npmjs.com/package/crypto-es)

Needs of encryption, decryption and hash are quite common in both front and back end projects. When dealing with sensitive data, functions with the name like MD5, Base64 or AES may be quite often seen in your codes.

Of all JavaScript cryptography libraries, [CryptoJS](https://code.google.com/archive/p/crypto-js/) is mostly widely used for its simple APIs and rich variety of functions. It derives in early years, and the main codebase is still on the Google Code. Although it has been migrated to npm, the last update was 3 years ago. For historical reasons, some of it's features seem out of date for modern usages:

- It creates an unique object oriented system based on prototypal inheritance to simulate class inheritance.
- The files use Immediately Invoked Function Expression (IIFE) to export modules.

Before ECMAScript 6, these features avoided certain defects of JavaScript while ensuring browser compatibility and out-of-the-box usage. But since the new ECMAScript standard has specified the Class and Module system to solve those defects, I would try to experimentally refactor CryptoJS in modern ECMAScript.

The project is called  [CryptoES](https://github.com/entronad/crypto-es) . As an  experiment, the compatibility would not be the first concern, and the only rule is to meet the latest ECMAScript standard. For example, I would only use ECMAScript Module for the module system, with out CommonJS support. With the help of Babel and loader hook, this project could satisfy any production application. And as the ECMAScript standard gets more widespread, there would be more directly usages of it.

Besides, as a refactoring project, it's APIs is all the same as CryptoJS.

# Refactoring to the ECMAScript Class

CryptoJS extends the  JavaScript prototypal inheritance and implements an unique "base objects inheritance" system. It has functions such as `extend`, `override`, `mixin`, etc. which makes it like general class-based OOP languages. This is defferent in approach but the same in purpose of the ECMAScript Class. Our first step is to replace this unique inheritance system with classes, which would make the code concise. But before that, let's have a review of some key concepts of the ECMAScript Class:

**constructor**

We know that in ECMAScript, class definition is a syntactic sugar of traditional JavaScript constructor. So we could get access to an instance's class through it's `constructor` field. What could we do with this knowledge? In an instance's instance methods, we could create a new instance which has the same class as the former one by expression `new this.constructor()` .

**this**

Unlike in the traditional JavaScript prototypal inheritance, in ECMAScript Class inheritance, while creating an instance, the constructor of it's super class will firstly called, and then the fields and methods of the super class will be added to the instance, finally the constructor of the instance will be called. This makes sure that the fields and methods defined in the sub class will override the super class's correctly.

Note that 'this' in the instance methods refers to the instance object, while 'this' in the static methods refers to the class object. So we could impliment factory mode by calling  `new this()` in the solid methods of classes.

**super**

In the class definition,' super()' with parentheses refers to super class's constructor, while 'super' without parentheses in static methods refers to super class, just the same as 'this'. So, When overriding methods of the sub class, we could call the overrided method of the super class first by `super.overridedMethod.call(this)` .

**prototype &  \_\_proto\_\_**

A class is a constructor in essence, so a class has 'prototype' field refering to its prototype object.

A class is also an object, so it has '\_\_proto\_\_' field refering to it's super class, while the prototype of a class also has '\_\_proto\_\_' field refering to the prototype object of a it's super class. This is a kind of prototype chain.

A instance object's  '\_\_proto\_\_' field refers to the prototype object of it's class.

With these properties, we could get the inheritance relations of instance and classes.

---

The commonly used version of CryptoJS is hosted on the GitHub: [brix/crypto-js](https://github.com/brix/crypto-js)，which  [CryptoES](https://github.com/entronad/crypto-es) will mainly based on. It's base objects are defined in the core.js file.

CryptoJS implimented prototypal inheritance with the object named Base:

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

In fact, the inheritance is implemented by calling the extend method of "super object" with arguments of overriding fields and methods to create a new "sub object". The real instances will be returned by the create method of these objects, while the init method will be called as a constructor. The disadvantage of these approach is that instances is not created with the keyword 'new' as regular. And instances will recursively keep all "ancestor objects" of its inheritance chain in the $super field, which is certainly redundant.