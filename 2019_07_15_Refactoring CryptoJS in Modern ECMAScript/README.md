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

Note that `this` in instance methods refers to the instance, while `this` in static methods refers to the class. So we could implement factory mode by calling  `new this()` in the static methods of a class.

**super**

In the class definition, `super()` with parentheses refers to super class's constructor, while `super` without parentheses in static methods, just like `this` , refers to super class. So, when overriding methods, we could call the overridden method of the super class first by expression `super.overridedMethod.call(this)` .

**prototype & \_\_proto\_\_**

A class is a constructor in essence, so it has a `prototype` field referring to its prototype object. And a class is also an object, so it has a `__proto__` field referring to it's super class too. Since the prototype of a class has a `__proto__` field referring to it's super class's prototype object, this composes a prototype chain.

An instance's  `__proto__` field refers to the prototype object of it's class.

With these features, we could get the inheritance relations of both instances and classes.

---

The commonly used version of CryptoJS nowadays is hosted on the GitHub: [brix/crypto-js](https://github.com/brix/crypto-js)，which [CryptoES](https://github.com/entronad/crypto-es) will mainly based on. It's "base objects" are defined in the core.js file.

The inheritance is implemented in the object named `Base` :

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

In fact, the inheritance is implemented by calling the `extend` method of the "super object" with arguments of overriding fields and methods, and the `extend` method creates a new "sub object". The real instance will be returned by the `create` method of this "sub object", of which the `init` method will be called as a constructor. The disadvantage of this approach is that instances are not created with the keyword `new` as regular. And an instance will recursively keeps all the "ancestor objects" of its inheritance chain in the `$super` field, which is certainly redundant.

These goals could easily achieved with the keyword `extend` and class constructor of ECMAScript Class, with out any extra codes or problems above. But for the consistence of the APIs, the static method `create` is kept, so that you could create instances with `ClassName.create()` . The arguments of the `create` method will be passed to the real constructor by the rest operator and the argument destruction:

```
export class Base {
  static create(...args) {
    return new this(...args);
  }
}
```

the basic purpose of `Base` is to provide the `mixin` method. In CryptoJS, the boundary between "class objects" like `Base` and real instance objects is fuzzy, and the `mixin` method could be called by "class objects":

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

But in both logic and fact, the `mixin` method should be an instance method, which just acts like `Object.assign()` :

```
mixIn(properties) {
  return Object.assign(this, properties);
}
```

Finally, the copying of instances. With prototypal inheritance, the CryptoJS implementation is quite odd:

```
clone() {
  const clone = new this.constructor();
  Object.assign(clone, this);
  return clone;
}
```

 Since `new this.constructor()` could be used in any instances with out indicating it's the class name, we could do it in a more straight way:

```
clone() {
  const clone = new this.constructor();
  Object.assign(clone, this);
  return clone;
}
```

Then, by inheriting the `Base` class, all other core classes will have these methods. And with ECMAScript Class, the usage of methods would be more standardized. For example, in CryptoJS, some instance creating methods of `Wordarray ` are:

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

But now they are standard constructor or static methods:

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

---

After this refactoring to class, we should add some extra unit test for the inheritance relations:

- `__Proto__` of the sub class must refer to the super class.
- prototype objects of sub class and super class are of right order in the prototype chain.

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

# WordArray and Bitwise Operations

Bitwise operations are the foundation of cryptography algorithms.

What ever the data type is, bitwise operation treats them as a contiguous sequence. For sack of performance, bitwise operation would better acting on a section of contiguous memory. Some languages provide ways to operate the contiguous memory, such as pointer in C++ and ArrayBuffer in ECMAScript 6.

JavaScript was designed as a script language for browser at first, so there was no memory operation before. But there is still a way to get the abstraction of the bitwise sequence, which is Binary Bitwise Operators.

According to the specification, in the operation with binary bitwise operators, all operands, whatever their origin types are, will be convert to 32 bits unsighed int by `ToInt32()` . Then they are treated as 32 bits length sequences, and the result is a 32 bits unsighed int. So, by splicing these 32 bits unsighed ints, we could simulate the operation on a contiguous memory.

Based on this, CryptoJS implemented a class called WordArray, as the abstraction of the bitwise sequence. WordArray is the most important basic class of CryptoJS, and all it's algorithms handle with WordArray objects in underlying implementation. Thus understanding WordArray is the precondition to understand it's algorithms.

---

The definition of WordArray is in the core.js file:

> *Note that all codes below are of [entronad/crypto-es](http://link.zhihu.com/?target=https%3A//github.com/entronad/crypto-es) .*

```
export class WordArray extends Base {

  constructor(words = [], sigBytes = words.length * 4) {
    super();

    this.words = words;
    this.sigBytes = sigBytes;
  }
  
  ...
}
```

WordArray directly inherits from `Base` . It has two fields: `word` and `sigBytes` . `words` is an array of 32 bits unsighed ints, and By splicing elements of this array in order we can get the bitwise sequence needed. In JavaScript conversion between the 32 bits unsighed int and the bit is through Binary Complement. But we don't have to get involved with that because the value of this int is meaningless. Generally the bitwise sequence is measured by bytes, or presented as hex numbers, so we just need to know that 32 bits equal to 4 bytes, or 8 hex numbers.

Encoding algorithms' subjects are strings. So the bitwise sequences are all of whole bytes, or times of 8 bits. But they are not certainly times of 32 bits. So only by the length of `words` we can't get the actual length of bitwise sequence, for there may be some empty tailing bytes. So we need the field `sigBytes` indicates the actual significant bytes length.

We could create a WordArray by directly passing these two fields:

```
const wordArray = CryptoES.lib.WordArray.create([0x00010203, 0x04050607], 6);
```

For convenience of pruning the `words` with the `sigBytes` , there is a `clamp` method in WordArray:

```
clamp() {
  // Shortcuts
  const { words, sigBytes } = this;

  // Clamp
  words[sigBytes >>> 2] &= 0xffffffff << (32 - (sigBytes % 4) * 8);
  words.length = Math.ceil(sigBytes / 4);
}
```

It will remove the "insignificant bytes" of the `words` . In the `words` array, the starting elements full of significant bytes will be kept, and the tailing elements with no significant bytes will be ignored through `words.length = Math.ceil(sigBytes / 4)` .

The middle element with both significant and insignificant bytes is a bit difficult to deal with. Firstly we should calculate the length to remove:  `(32 - (sigBytes % 4) * 8)` , and left move  `0xffffffff` bitwise by this length to get a 32 bits mask, then locate this middle element by  `sigBytes >>> 2` (just the same as int divided by 4) , finally `and` it with the mask to set the insignificant bytes to 0.

Locating elements by `>>>` and making `and` with masks is widely used in CryptoJS.

Just like `clamp` , the troublesome part of `concat` is also to handle the middle element:

```
concat(wordArray) {
  // Shortcuts
  const thisWords = this.words;
  const thatWords = wordArray.words;
  const thisSigBytes = this.sigBytes;
  const thatSigBytes = wordArray.sigBytes;

  // Clamp excess bits
  this.clamp();

  // Concat
  if (thisSigBytes % 4) {
    // Copy one byte at a time
    for (let i = 0; i < thatSigBytes; i += 1) {
      const thatByte = (thatWords[i >>> 2] >>> (24 - (i % 4) * 8)) & 0xff;
      thisWords[(thisSigBytes + i) >>> 2] |= thatByte << (24 - ((thisSigBytes + i) % 4) * 8);
    }
  } else {
    // Copy one word at a time
    for (let i = 0; i < thatSigBytes; i += 4) {
      thisWords[(thisSigBytes + i) >>> 2] = thatWords[i >>> 2];
    }
  }
  this.sigBytes += thatSigBytes;

  // Chainable
  return this;
}
```

Inside CryptoJs, WordArray is both input and output of most functions, but the external users concerns mostly about the string result. So WordArray provides overriding `toString` method:

```
toString(encoder = Hex) {
  return encoder.stringify(this);
}
```

Because the `words` array is of reference type, we should create a new copy of it by `slice` in the `clone` method:

```
clone() {
  const clone = super.clone.call(this);
  clone._data = this._data.clone();

  return clone;
}
```

Except the constructor, the static method `random` provides a random WordArray of certain length. Because `Math.random()` of JavaScript is unsafe and returns a 64 bits float, we will do some extra processing:

```
static random(nBytes) {
  const words = [];

  const r = (m_w) => {
    let _m_w = m_w;
    let _m_z = 0x3ade68b1;
    const mask = 0xffffffff;

    return () => {
      _m_z = (0x9069 * (_m_z & 0xFFFF) + (_m_z >> 0x10)) & mask;
      _m_w = (0x4650 * (_m_w & 0xFFFF) + (_m_w >> 0x10)) & mask;
      let result = ((_m_z << 0x10) + _m_w) & mask;
      result /= 0x100000000;
      result += 0.5;
      return result * (Math.random() > 0.5 ? 1 : -1);
    };
  };

  for (let i = 0, rcache; i < nBytes; i += 4) {
    const _r = r((rcache || Math.random()) * 0x100000000);

    rcache = _r() * 0x3ade67b7;
    words.push((_r() * 0x100000000) | 0);
  }

  return new WordArray(words, nBytes);
}
```

# ArrayBuffer and TypedArray

Since the ArrayBuffer is included to the ECMAScript, it has been used in more and more scenes, such as WebSocket, file objects, and canvas outputs. While handling with these forms of objects, we also need to encrypt, decrypt or hash them in some cases.

So CryptoJS extended the WordArray creator to allow ArrayBuffer and TypedArray as input. this extension is in a individual lib-typedArrays.js file, and dose a lot of checks and reconstructs the WordArray creator to ensure the compatibility. We integrate this to the origin WordArray constructor and simplified the these checks:

```
constructor(words = [], sigBytes = words.length * 4) {
  super();

  let typedArray = words;
  // Convert buffers to uint8
  if (typedArray instanceof ArrayBuffer) {
    typedArray = new Uint8Array(typedArray);
  }

  // Convert other array views to uint8
  if (
    typedArray instanceof Int8Array
    || typedArray instanceof Uint8ClampedArray
    || typedArray instanceof Int16Array
    || typedArray instanceof Uint16Array
    || typedArray instanceof Int32Array
    || typedArray instanceof Uint32Array
    || typedArray instanceof Float32Array
    || typedArray instanceof Float64Array
  ) {
    typedArray = new Uint8Array(typedArray.buffer, typedArray.byteOffset, typedArray.byteLength);
  }

  // Handle Uint8Array
  if (typedArray instanceof Uint8Array) {
    // Shortcut
    const typedArrayByteLength = typedArray.byteLength;

    // Extract bytes
    const _words = [];
    for (let i = 0; i < typedArrayByteLength; i += 1) {
      _words[i >>> 2] |= typedArray[i] << (24 - (i % 4) * 8);
    }

    // Initialize this word array
    this.words = _words;
    this.sigBytes = typedArrayByteLength;
  } else {
    // Else call normal init
    this.words = words;
    this.sigBytes = sigBytes;
  }
}
```

Then WordArray creator could recive an ArrayBuffer or TypedArray so that CryptoES algorisms could apply to them:

```
const words = CryptoES.lib.WordArray.create(new ArrayBuffer(8));
const rst = CryptoES.AES.encrypt(words, 'Secret Passphrase')
```

Note that ArrayBuffer could not directly passed to algorisms, you should change them to WordArray first.

With this, encrypting files would be easier:

```
const fileInput = document.getElementById('fileInput');
const file = fileInput.files[0];
const reader = new FileReader();
reader.readAsArrayBuffer(file);
reader.onload = function () {
  const arrayBuffer = reader.result;
  const words = CryptoES.lib.WordArray.create(arrayBuffer);
  const rst = CryptoES.AES.encrypt(words, 'Secret Passphrase')
  ...
};
```

