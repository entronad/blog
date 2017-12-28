![0](0.jpg)

# 【源码拾遗】从vue-router看前端路由的两种实现

> 本文由浅入深观摩vue-router源码是如何通过hash与History interface两种方式实现前端路由，介绍了相关原理，并对比了两种方式的优缺点与注意事项。最后分析了如何实现可以直接从文件系统加载而不借助后端服务器的Vue单页应用。

随着前端应用的业务功能越来越复杂、用户对于使用体验的要求越来越高，单页应用（SPA）成为前端应用的主流形式。大型单页应用最显著特点之一就是采用前端路由系统，通过改变URL，在不重新请求页面的情况下，更新页面视图。

“更新视图但不重新请求页面”是前端路由原理的核心之一，目前在浏览器环境中这一功能的实现主要有两种方式：

- 利用URL中的hash（“#”）
- 利用History interface在 HTML5中新增的方法

vue-router是Vue.js框架的路由插件，下面我们从它的源码入手，边看代码边看原理，由浅入深观摩vue-router是如何通过这两种方式实现前端路由的。

# 模式参数#

在vue-router中是通过mode这一参数控制路由的实现模式的：

```
const router = new VueRouter({
  mode: 'history',
  routes: [...]
})
```

创建VueRouter的实例对象时，mode以构造函数参数的形式传入。带着问题阅读源码，我们就可以从VueRouter类的定义入手。一般插件对外暴露的类都是定义在源码src根目录下的index.js文件中，打开该文件，可以看到VueRouter类的定义，摘录与mode参数有关的部分如下：

```
export default class VueRouter {
  
  mode: string; // 传入的字符串参数，指示history类别
  history: HashHistory | HTML5History | AbstractHistory; // 实际起作用的对象属性，必须是以上三个类的枚举
  fallback: boolean; // 如浏览器不支持，'history'模式需回滚为'hash'模式
  
  constructor (options: RouterOptions = {}) {
    
    let mode = options.mode || 'hash' // 默认为'hash'模式
    this.fallback = mode === 'history' && !supportsPushState // 通过supportsPushState判断浏览器是否支持'history'模式
    if (this.fallback) {
      mode = 'hash'
    }
    if (!inBrowser) {
      mode = 'abstract' // 不在浏览器环境下运行需强制为'abstract'模式
    }
    this.mode = mode

    // 根据mode确定history实际的类并实例化
    switch (mode) {
      case 'history':
        this.history = new HTML5History(this, options.base)
        break
      case 'hash':
        this.history = new HashHistory(this, options.base, this.fallback)
        break
      case 'abstract':
        this.history = new AbstractHistory(this, options.base)
        break
      default:
        if (process.env.NODE_ENV !== 'production') {
          assert(false, `invalid mode: ${mode}`)
        }
    }
  }

  init (app: any /* Vue component instance */) {
    
    const history = this.history

    // 根据history的类别执行相应的初始化操作和监听
    if (history instanceof HTML5History) {
      history.transitionTo(history.getCurrentLocation())
    } else if (history instanceof HashHistory) {
      const setupHashListener = () => {
        history.setupListeners()
      }
      history.transitionTo(
        history.getCurrentLocation(),
        setupHashListener,
        setupHashListener
      )
    }

    history.listen(route => {
      this.apps.forEach((app) => {
        app._route = route
      })
    })
  }

  // VueRouter类暴露的以下方法实际是调用具体history对象的方法
  push (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    this.history.push(location, onComplete, onAbort)
  }

  replace (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    this.history.replace(location, onComplete, onAbort)
  }
}
```

可以看出：

1. 作为参数传入的字符串属性mode只是一个标记，用来指示实际起作用的对象属性history的实现类，两者对应关系如下：

   | mode       | history         |
   | ---------- | --------------- |
   | 'history'  | HTML5History    |
   | 'hash'     | HashHistory     |
   | 'abstract' | AbstractHistory |

2. 在初始化对应的history之前，会对mode做一些校验：若浏览器不支持HTML5History方式（通过supportsPushState变量判断），则mode强制设为'hash'；若不是在浏览器环境下运行，则mode强制设为'abstract'

3. VueRouter类中的onReady(), push()等方法只是一个代理，实际是调用的具体history对象的对应方法，在init()方法中初始化时，也是根据history对象具体的类别执行不同操作

在浏览器环境下的两种方式，分别就是在HTML5History，HashHistory两个类中实现的。他们都定义在src/history文件夹下，继承自同目录下base.js文件中定义的History类。History中定义的是公用和基础的方法，直接看会一头雾水，我们先从HTML5History，HashHistory两个类中看着亲切的push(), replace()方法的说起。

# HashHistory

看源码前先回顾一下原理：

hash（“#”）符号的本来作用是加在URL中指示网页中的位置：

> http://www.example.com/index.html#print

\#符号本身以及它后面的字符称之为hash，可通过window.location.hash属性读取。它具有如下特点：

- hash虽然出现在URL中，但不会被包括在HTTP请求中。它是用来指导浏览器动作的，对服务器端完全无用，因此，改变hash不会重新加载页面

- 可以为hash的改变添加监听事件：

  ```
  window.addEventListener("hashchange", funcRef, false)
  ```

- 每一次改变hash（window.location.hash），都会在浏览器的访问历史中增加一个记录

利用hash的以上特点，就可以来实现前端路由“更新视图但不重新请求页面”的功能了。

**HashHistory.push()**

我们来看HashHistory中的push()方法：

```
push (location: RawLocation, onComplete?: Function, onAbort?: Function) {
  this.transitionTo(location, route => {
    pushHash(route.fullPath)
    onComplete && onComplete(route)
  }, onAbort)
}

function pushHash (path) {
  window.location.hash = path
}
```

transitionTo()方法是父类中定义的是用来处理路由变化中的基础逻辑的，push()方法最主要的是对window的hash进行了直接赋值：

````
window.location.hash = route.fullPath
````

hash的改变会自动添加到浏览器的访问历史记录中。

那么视图的更新是怎么实现的呢，我们来看父类History中transitionTo()方法的这么一段：

```
transitionTo (location: RawLocation, onComplete?: Function, onAbort?: Function) {
  const route = this.router.match(location, this.current)
  this.confirmTransition(route, () => {
    this.updateRoute(route)
    ...
  })
}

updateRoute (route: Route) {
  
  this.cb && this.cb(route)
  
}

listen (cb: Function) {
  this.cb = cb
}
```

可以看到，当路由变化时，调用了History中的this.cb方法，而this.cb方法是通过History.listen(cb)进行设置的。回到VueRouter类定义中，找到了在init()方法中对其进行了设置：

```
init (app: any /* Vue component instance */) {
    
  this.apps.push(app)

  history.listen(route => {
    this.apps.forEach((app) => {
      app._route = route
    })
  })
}
```

根据注释，app为Vue组件实例，但我们知道Vue作为渐进式的前端框架，本身的组件定义中应该是没有有关路由内置属性_route，如果组件中要有这个属性，应该是在插件加载的地方，即VueRouter的install()方法中混合入Vue对象的，查看install.js源码，有如下一段：

```
export function install (Vue) {
  
  Vue.mixin({
    beforeCreate () {
      if (isDef(this.$options.router)) {
        this._router = this.$options.router
        this._router.init(this)
        Vue.util.defineReactive(this, '_route', this._router.history.current)
      }
      registerInstance(this, this)
    },
  })
}
```

通过Vue.mixin()方法，全局注册一个混合，影响注册之后所有创建的每个 Vue 实例，该混合在beforeCreate钩子中通过Vue.util.defineReactive()定义了响应式的\_route属性。所谓响应式属性，即当\_route值改变时，会自动调用Vue实例的render()方法，更新视图。

总结一下，从设置路由改变到视图更新的流程如下：

```
$router.push() --> HashHistory.push() --> History.transitionTo() --> History.updateRoute() --> {app._route = route} --> vm.render()
```

**HashHistory.replace()**

replace()方法与push()方法不同之处在于，它并不是将新路由添加到浏览器访问历史的栈顶，而是替换掉当前的路由：

```
replace (location: RawLocation, onComplete?: Function, onAbort?: Function) {
  this.transitionTo(location, route => {
    replaceHash(route.fullPath)
    onComplete && onComplete(route)
  }, onAbort)
}
  
function replaceHash (path) {
  const i = window.location.href.indexOf('#')
  window.location.replace(
    window.location.href.slice(0, i >= 0 ? i : 0) + '#' + path
  )
}
```

可以看出，它与push()的实现结构上基本相似，不同点在于它不是直接对window.location.hash进行赋值，而是调用window.location.replace方法将路由进行替换。

**监听地址栏**

以上讨论的VueRouter.push()和VueRouter.replace()是可以在vue组件的逻辑代码中直接调用的，除此之外在浏览器中，用户还可以直接在浏览器地址栏中输入改变路由，因此VueRouter还需要能监听浏览器地址栏中路由的变化，并具有与通过代码调用相同的响应行为。在HashHistory中这一功能通过setupListeners实现：

```
setupListeners () {
  window.addEventListener('hashchange', () => {
    if (!ensureSlash()) {
      return
    }
    this.transitionTo(getHash(), route => {
      replaceHash(route.fullPath)
    })
  })
}
```

该方法设置监听了浏览器事件hashchange，调用的函数为replaceHash，即在浏览器地址栏中直接输入路由相当于代码调用了replace()方法

# HTML5History

History interface是浏览器历史记录栈提供的接口，通过back(), forward(), go()等方法，我们可以读取浏览器历史记录栈的信息，进行各种跳转操作。

从HTML5开始，History interface提供了两个新的方法：pushState(), replaceState()使得我们可以对浏览器历史记录栈进行修改：

```
window.history.pushState(stateObject, title, URL)
window.history.replaceState(stateObject, title, URL)
```

- stateObject: 当浏览器跳转到新的状态时，将触发popState事件，该事件将携带这个stateObject参数的副本
- title: 所添加记录的标题
- URL: 所添加记录的URL

这两个方法有个共同的特点：当调用他们修改浏览器历史记录栈后，虽然当前URL改变了，但浏览器不会立即发送请求该URL（the browser won't attempt to load this URL after a call to pushState()），这就为单页应用前端路由“更新视图但不重新请求页面”提供了基础。

我们来看vue-router中的源码：

```
push (location: RawLocation, onComplete?: Function, onAbort?: Function) {
  const { current: fromRoute } = this
  this.transitionTo(location, route => {
    pushState(cleanPath(this.base + route.fullPath))
    handleScroll(this.router, route, fromRoute, false)
    onComplete && onComplete(route)
  }, onAbort)
}

replace (location: RawLocation, onComplete?: Function, onAbort?: Function) {
  const { current: fromRoute } = this
  this.transitionTo(location, route => {
    replaceState(cleanPath(this.base + route.fullPath))
    handleScroll(this.router, route, fromRoute, false)
    onComplete && onComplete(route)
  }, onAbort)
}

// src/util/push-state.js
export function pushState (url?: string, replace?: boolean) {
  saveScrollPosition()
  // try...catch the pushState call to get around Safari
  // DOM Exception 18 where it limits to 100 pushState calls
  const history = window.history
  try {
    if (replace) {
      history.replaceState({ key: _key }, '', url)
    } else {
      _key = genKey()
      history.pushState({ key: _key }, '', url)
    }
  } catch (e) {
    window.location[replace ? 'replace' : 'assign'](url)
  }
}

export function replaceState (url?: string) {
  pushState(url, true)
}
```

代码结构以及更新视图的逻辑与hash模式基本类似，只不过将对window.location.hash直接进行赋值window.location.replace()改为了调用history.pushState()和history.replaceState()方法。

在HTML5History中添加对修改浏览器地址栏URL的监听是直接在构造函数中执行的：

```
constructor (router: Router, base: ?string) {
  
  window.addEventListener('popstate', e => {
    const current = this.current
    this.transitionTo(getLocation(this.base), route => {
      if (expectScroll) {
        handleScroll(router, route, current, true)
      }
    })
  })
}
```

当然了HTML5History用到了HTML5的新特特性，是需要特定浏览器版本的支持的，前文已经知道，浏览器是否支持是通过变量supportsPushState来检查的：

```
// src/util/push-state.js
export const supportsPushState = inBrowser && (function () {
  const ua = window.navigator.userAgent

  if (
    (ua.indexOf('Android 2.') !== -1 || ua.indexOf('Android 4.0') !== -1) &&
    ua.indexOf('Mobile Safari') !== -1 &&
    ua.indexOf('Chrome') === -1 &&
    ua.indexOf('Windows Phone') === -1
  ) {
    return false
  }

  return window.history && 'pushState' in window.history
})()
```

以上就是hash模式与history模式源码的导读，这两种模式都是通过浏览器接口实现的，除此之外vue-router还为非浏览器环境准备了一个abstract模式，其原理为用一个数组stack模拟出浏览器历史记录栈的功能。当然，以上只是一些核心逻辑，为保证系统的鲁棒性源码中还有大量的辅助逻辑，也很值得学习。此外在vue-router中还有路由匹配、router-view视图组件等重要部分，关于整体源码的阅读推荐滴滴前端的[这篇文章](https://zhuanlan.zhihu.com/p/24574970)

# 两种模式比较

在一般的需求场景中，hash模式与history模式是差不多的，但几乎所有的文章都推荐使用history模式，理由竟然是："\#" 符号太丑...0_0 "

> 如果不想要很丑的 hash，我们可以用路由的 history 模式    ——官方文档

当然，严谨的我们肯定不应该用颜值评价技术的好坏。根据MDN的介绍，调用history.pushState()相比于直接修改hash主要有以下优势：

- pushState设置的新URL可以是与当前URL同源的任意URL；而hash只可修改\#后面的部分，故只可设置与当前同文档的URL
- pushState设置的新URL可以与当前URL一模一样，这样也会把记录添加到栈中；而hash设置的新值必须与原来不一样才会触发记录添加到栈中
- pushState通过stateObject可以添加任意类型的数据到记录中；而hash只可添加短字符串
- pushState可额外设置title属性供后续使用

**history模式的一个问题**

我们知道对于单页应用来讲，理想的使用场景是仅在进入应用时加载index.html，后续在的网络操作通过Ajax完成，不会根据URL重新请求页面，但是难免遇到特殊情况，比如用户直接在地址栏中输入并回车，浏览器重启重新加载应用等。

hash模式仅改变hash部分的内容，而hash部分是不会包含在HTTP请求中的：

```
http://oursite.com/#/user/id   // 如重新请求只会发送http://oursite.com/
```

故在hash模式下遇到根据URL请求页面的情况不会有问题。

而history模式则会将URL修改得就和正常请求后端的URL一样

```
http://oursite.com/user/id
```

在此情况下重新向后端发送请求，如后端没有配置对应/user/id的路由处理，则会返回404错误。官方推荐的解决办法是在服务端增加一个覆盖所有情况的候选资源：如果 URL 匹配不到任何静态资源，则应该返回同一个 `index.html` 页面，这个页面就是你 app 依赖的页面。同时这么做以后，服务器就不再返回 404 错误页面，因为对于所有路径都会返回 `index.html` 文件。为了避免这种情况，在 Vue 应用里面覆盖所有的路由情况，然后在给出一个 404 页面。或者，如果是用 Node.js 作后台，可以使用服务端的路由来匹配 URL，当没有匹配到路由的时候返回 404，从而实现 fallback。

# 直接加载应用文件

> Tip: built files are meant to be served over an HTTP server.
> Opening index.html over file:// won't work.

Vue项目通过vue-cli的webpack打包完成后，命令行会有这么一段提示。通常情况，无论是开发还是线上，前端项目都是通过服务器访问，不存在 "Opening index.html over file://" ，但程序员都知道，需求和场景永远是千奇百怪的，只有你想不到的，没有产品经理想不到的。

本文写作的初衷就是遇到了这样一个问题：需要快速开发一个移动端的展示项目，决定采用WebView加载Vue单页应用的形式，但没有后端服务器提供，所以所有资源需从本地文件系统加载：

```
// AndroidAppWrapper
public class MainActivity extends AppCompatActivity {

    private WebView webView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        webView = new WebView(this);
        webView.getSettings().setJavaScriptEnabled(true);
        webView.loadUrl("file:///android_asset/index.html");
        setContentView(webView);
    }

    @Override
    public boolean onKeyDown(int keyCode, KeyEvent event) {
        if ((keyCode == KeyEvent.KEYCODE_BACK) && webView.canGoBack()) {
            webView.goBack();
            return true;
        }
        return false;
    }
}
```

此情此景看来是必须 "Opening index.html over file://" 了，为此，我首先要进行了一些设置

- 在项目config.js文件中将assetsPublicPath字段的值改为相对路径 './' 
- 调整生成的static文件夹中图片等静态资源的位置与代码中的引用地址一致

这是比较明显的需要改动之处，但改完后依旧无法顺利加载，经过反复排查发现，项目在开发时，router设置为了history模式（为了美观...0_0"），当改为hash模式后就可正常加载了。

为什么会出现这种情况呢？我分析原因可能如下：

当从文件系统中直接加载index.html时，URL为：

```
file:///android_asset/index.html
```

而首页视图需匹配的路径为path: '/' :

```
export default new Router({
  mode: 'history',
  routes: [
    {
      path: '/',
      name: 'index',
      component: IndexView
    }
  ]
})
```

我们先来看history模式，在HTML5History中：

```
ensureURL (push?: boolean) {
  if (getLocation(this.base) !== this.current.fullPath) {
    const current = cleanPath(this.base + this.current.fullPath)
    push ? pushState(current) : replaceState(current)
  }
}

export function getLocation (base: string): string {
  let path = window.location.pathname
  if (base && path.indexOf(base) === 0) {
    path = path.slice(base.length)
  }
  return (path || '/') + window.location.search + window.location.hash
}
```

逻辑只会确保存在URL，path是通过剪切的方式直接从window.location.pathname获取到的，它的结尾是index.html，因此匹配不到 '/' ，故 "Opening index.html over file:// won't work" 。

再看hash模式，在HashHistory中：

```
export class HashHistory extends History {
  constructor (router: Router, base: ?string, fallback: boolean) {
    ...
    ensureSlash()
  }

  // this is delayed until the app mounts
  // to avoid the hashchange listener being fired too early
  setupListeners () {
    window.addEventListener('hashchange', () => {
      if (!ensureSlash()) {
        return
      }
      ...
    })
  }

  getCurrentLocation () {
    return getHash()
  }
}

function ensureSlash (): boolean {
  const path = getHash()
  if (path.charAt(0) === '/') {
    return true
  }
  replaceHash('/' + path)
  return false
}

export function getHash (): string {
  const href = window.location.href
  const index = href.indexOf('#')
  return index === -1 ? '' : href.slice(index + 1)
}
```

我们看到在代码逻辑中，多次出现一个函数ensureSlash()，当#符号后紧跟着的是'/'，则返回true，否则强行插入这个'/'，故我们可以看到，即使是从文件系统打开index.html，URL依旧会变为以下形式：

```
file:///C:/Users/dist/index.html#/
```

getHash()方法返回的path为 '/' ，可与首页视图的路由匹配。

故要想从文件系统直接加载Vue单页应用而不借助后端服务器，除了打包后的一些路径设置外，还需确保vue-router使用的是hash模式。