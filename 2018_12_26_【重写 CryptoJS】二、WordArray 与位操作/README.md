我们常见的各种编码、散列、加密算法，其基础都是位操作。

不管是对哪种数据类型，位操作对象的本质都是一段连续的比特序列。从性能的角度讲，位操作最好是能直接操作连续的内存位。很多语言提供了直接操作连续内存位的操作，比如C++中的数组与指针，ECMAScript 6 中的ArrayBuffer。JavaScript最初是作为浏览器的脚本语言设计的，并没有直接操作内存的特性，但我们还是有办法获取到比特序列的抽象的，那就是通过二进制位操作符（Binary Bitwise Operators）。

根据标准，在含有位操作符的运算中，不管是什么类型的操作数，都通过ToInt32()转换为32位有符号整数，然后将其当做32位的比特序列进行位运算，运算结果返回也为32位有符号整数。因此，通过拼接32位有符号整数，就可以实现“对一段连续的比特序列进行位操作”的功能了。

正是基于这样的原理，CryptoJs实现了名为WordArray的类，作为“一段连续比特序列”的抽象进行各种位操作。WordArray是CryptoJs中最核心的一个类，所有主要算法的实际操作对象都是WordArray对象。

# WordArray 解析

>  *注：以下所有代码为[entronad/crypto-es](http://link.zhihu.com/?target=https%3A//github.com/entronad/crypto-es)中的重写代码*

WordArray的定义位于core.js中：

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

它直接继承自Base，有两个成员变量，一个是words，为32位有符号整数构成的数组，通过按顺序拼接数组中的数，就组成了比特序列。JavaScript中32位有符号整数是通过补码转换为二进制的，不过在这里我们不需要关注这点，因为这个整数的值对于我们是没有意义的，实际使用中，比特序列更多的是用字节作单位，或用16进制数表示，因此我们只需要知道32位等价于4个字节，等价于8个16进制数。

编码算法的对象是字符，因此实际比特序列长度都是整字节的，即8的倍数，但不一定是32的倍数，因此仅通过words数组是不能反映比特序列实际长度的，最后可能有多余位，因此WordArray有第二个成员变量sigBytes，表示实际有用的字节数（significant bytes）。

可通过直接传入这两个字段构建实例：

```
const wordArray = CryptoES.lib.WordArray.create([0x00010203, 0x04050607], 6);
```

