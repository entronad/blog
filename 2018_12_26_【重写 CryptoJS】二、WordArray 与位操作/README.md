>  源码地址：[entronad/crypto-es](https://github.com/entronad/crypto-es)
>
>
>
> [【重写 CryptoJS】一、ECMAScript 类与继承](https://zhuanlan.zhihu.com/p/52165088)

我们常见的各种编码、散列、加密算法，其基础都是位操作。

不管是对哪种数据类型，位操作对象的本质都是一段连续的比特序列。从性能的角度讲，位操作最好是能直接操作连续的内存位。很多语言提供了直接操作连续内存位的操作，比如 C++ 中的数组与指针，ECMAScript 6 中的 ArrayBuffer 。 JavaScript 最初是作为浏览器的脚本语言设计的，并没有直接操作内存的特性，但还是有办法获取到比特序列的抽象，那就是通过二进制位操作符（ Binary Bitwise Operators ）。

根据标准，在含有位操作符的运算中，不管是什么类型的操作数，都通过 ToInt32() 转换为 32 位有符号整数，然后将其当做 32 位的比特序列进行位运算，运算结果返回也为 32 位有符号整数。因此，通过拼接 32 位有符号整数，就可以实现“对一段连续的比特序列进行位操作”的功能了。

正是基于这样的原理， CryptoJs 实现了名为 WordArray 的类，作为“一段连续比特序列”的抽象进行各种位操作。 WordArray 是 CryptoJs 中最核心的一个类，所有主要算法的实际操作对象都是 WordArray 对象。理解 WordArray 是理解 CryptoJs 各算法的基础。

WordArray 的定义位于 core.js 中：

>  *注：以下所有代码为 [entronad/crypto-es](http://link.zhihu.com/?target=https%3A//github.com/entronad/crypto-es) 中的重写代码*

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

它直接继承自 Base ，有 words 和 sigBytes 两个成员变量。words 为 32 位有符号整数构成的数组，通过按顺序拼接数组中的数，就组成了比特序列。 JavaScript 中 32 位有符号整数是通过补码转换为二进制的，不过在这里我们不需要关注这点，因为这个整数的值是没有意义的，实际使用中，比特序列更多的是用字节作单位，或用 16 进制数表示，因此我们只需要知道 32 位等价于 4 个字节，等价 于 8个 16 进制数。

编码算法的对象是字符，因此实际比特序列长度都是整字节的，即 8 的倍数，但不一定是 32 的倍数，因此仅通过 words 数组是不能反映比特序列实际长度的，最后可能有多余位，因此 WordArray 有第二个成员变量 sigBytes ，表示实际有效字节数（ significant bytes ）。

可通过直接传入这两个字段构建实例：

```
const wordArray = CryptoES.lib.WordArray.create([0x00010203, 0x04050607], 6);
```

为方便 sigBytes 对 words 数组的控制， WordArray 中定义了一个特别的方法 clamp ：

```
clamp() {
  // Shortcuts
  const { words, sigBytes } = this;

  // Clamp
  words[sigBytes >>> 2] &= 0xffffffff << (32 - (sigBytes % 4) * 8);
  words.length = Math.ceil(sigBytes / 4);
}
```

 clamp 意指压缩，作用是移除 words 中不是有效的字节（ insignificant bytes ）。前段全是有效字节的 word 直接保留，末段完全没有有效字节的 word 通过 `words.length = Math.ceil(sigBytes / 4)` 移除。

比较麻烦的是中间不全是有效字节的一个分界 word 。首先算出要去除的位数： `(32 - (sigBytes % 4) * 8)` ，对 `0xffffffff` 左移该位数获得一个 32 位的掩码，然后通过 `sigBytes >>> 2` （相当于整除 4 ）找到分界 word 下标，通过与掩码取与将无效字节置 0 。

这种右移定位下标和掩码与或计算在 CryptoJS 中非常普遍。

与 clamp 类似，拼接两个 WordArray 的 concat 方法主要麻烦之处也在处理分界 word ：

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

在 CryptoJs 内部 WordArray 是各算法主要的操作对象和结果，不过外部使用者想要的结果还是指定编码方式的字符串，因此 WordArray 有重写的 toString 方法：

```
toString(encoder = Hex) {
  return encoder.stringify(this);
}
```

由于 words 数组是引用类型，因此 clone 方法需要重写一下，通过 slice 复制一份拷贝：

```
clone() {
  const clone = super.clone.call(this);
  clone._data = this._data.clone();

  return clone;
}
```

除了构造函数，还有一个静态函数生成指定字节长度的随机 WordArray 。由于 Math.random() 提供的不是安全的随机数，且类型为 64 位浮点数，所以生成过程中进行了一些处理：

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



> 题图： Royal 打字机

