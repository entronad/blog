> 一种在 React Native 中封装的响应式 Echarts 组件，使用与示例请参见：[react-native-echarts-demo](https://github.com/entronad/react-native-echarts-demo)

近年来，随着移动端对数据可视化的要求越来越高，类似 [MPAndroidChart](https://github.com/PhilJay/MPAndroidChart) 这样的传统移动端图表库已经不能满足产品经理日益变态的需求。前端领域数据可视化的发展相对繁荣一些，很多时候设计也会要求移动端和PC前端的图表视觉效果统一，通过 WebView 在移动端使用 [Echarts](http://echarts.baidu.com/) 这样功能强大的前端数据可视化库，是解决问题的好办法。

在 React Native 开发中，由于使用的是与前端相同的JavaScript语言，所以与Echarts的衔接相对顺畅些，不过一些必要的组件封装工作还是能够大大提高开发效率的。Echarts官方推荐过一个第三方的封装库：[react-native-echarts](https://github.com/somonus/react-native-echarts)（注：它对应的nmp package名字为[native-echarts](https://www.npmjs.com/package/native-echarts) ），目前有400+ star 和100+的周下载量，可见还是被广泛使用的。但是我们经过，发现react-native-echarts存在以下一些问题：

- 该库已半年多未更新，Echarts版本停留在3.0，Android端打包需手动添加assets的问题也一直未处理
- 库的接口灵活度较低，比如智能通过width、height设置大小，无法使用Echarts扩展包，无法进行事件注册、WebView通信等

由于用WebView封装Echarts涉及到本地html，不是纯JavaScript语言层面的功能，有没有native代码，所以做成nmp package并不是一个很好的选择，写成项目里的内部组件，自己进行配置反而是既方便又灵活的方案。

因此我们决定不使用第三方的Echarts封装库，自己写一个通用组件WebChart，该组件具有以下特点：

- 按照响应式进行设计，只需在option中配置好数据源，数据变化后图表就会自动刷新，更符合React的风格
- 利用WebView的postMessage和onMessage接口，可实现图表与其它React Native组件的事件通信
- 通过组件的exScript参数，可为WebView添加任意脚本，使用灵活
- 由于是自己写的组件，echarts版本、扩展包，svg/canvas、数据增量加载都可以自己设定

# Demo 与使用方法

这里是一个[demo](https://github.com/entronad/react-native-echarts-demo)，如果你需要使用的话按以下步骤：

1. 将根目录下的WebChart文件夹拷到你项目中合适的地方
2. 将/android\app\src\main\assets\web文件夹拷到你项目的对应位置，没有assets文件夹需手动创建。

只需以上两步就可以在项目中使用WebChart组件了。

如果需要进一步定制的话，Echarts在以上两个文件夹中的index.html中的\<script /\>标签内，目前是放的是4.0完整版，无扩展包，可到[官网](http://echarts.baidu.com/download.html)下载所需的版本和扩展包替换；svg/canvas、数据增量加载等可在WebChart/index.js中直接进行修改。

具体使用可参加demo，style的设置就和普通的React Native组件一样，可使用flex，也可设为定值。说明一下WebChart的三个参数：

- option(object)：赋给setOption的参数对象，发生变化后WebChart内部会自动调用setOption，实现响应式刷新；由于内部是通过JSON与WebView通信，考虑到性能，JSON解析时未进行函数的处理，所以避免使用函数式的formatter和类形式的LinearGradient，和demo一样使用模板式和普通对象的吧
- exScript(string)：任何你想在WebView 加载时执行的代码，一般会是事件注册之类的，推荐使用模板字面量
- onMessage(function)：WebView内部触发postMessage之后的回调，postMessage需先在exScript中进行设置，用于图表与其它React Native组件的通信

当然这是根据我们的业务需要设计的参数，你完全可以随便怎么设定。

# Echarts与React Native组件的通信

在React Native的WebView组件中，提供了onMessage和postMessage来进行html与组件的双向通信，具体使用可参加文档。

利用webView.postMessage，WebChart实现了通知Echarts执行setOption；在exScript中，可利用window.postMessage实现Echarts的事件向React Native组件的通信。

一般我们会约定一个通信协议的data为这样格式的对象：

```
{
  type: 'someType',
  payload: {
  	value: 111,
  },
}
```

由于onMessage和postMessage只能进行字符串的传递，在exScript需进行JSON序列化，类似这样：

```
exScript={`
  chart.on('click', (params) => {
    if(params.componentType === 'series') {
      window.postMessage(JSON.stringify({
        type: 'select',
        payload: {
      	  index: params.dataIndex,
        },
      }));
    }
  });
`}
```



以上就是我们封装的响应式WebChart组件及使用，完整代码请参见[demo](https://github.com/entronad/react-native-echarts-demo) 。

在使用中，还有以下几个坑未解决，目前只能绕过，欢迎知道的同学指正：

- 在IOS中，Echarts好像渲染不出透明的效果，用rgba设置的颜色不能正常
- React Native的WebView好像style.height属性无效，因此不得不在外面套了个View
- 按现在这个方式，index.html在Android上会有两份，因为平台判断是运行时进行的，哪怕分开设置index.anroid.js和index.ios.js打包时也会都打包进去，而Android中又必须手动添加assets
- index.html中必须内联引入Echarts的代码，外部引用单独的js文件好像不行