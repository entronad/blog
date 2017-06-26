http://blog.csdn.net/xllily_11/article/details/51820909

https://zhuanlan.zhihu.com/p/24574970



h5 history API 介绍，重点是`history.pushState` 和 `history.replaceState`

history 必须要后端配合





> 

随着前端应用的业务功能需求越来越复杂、用户对于应用内使用体验的要求越来越高，单页应用（SPA）越来越成为前端应用的主流形式。大型单页应用最显著特点之一就是采用前端路由系统，通过改变URL，在不重新请求刷新页面的情况下，更新页面视图。

“更新视图但不重新请求页面”的实现是前端路由系统原理的核心之一，目前在浏览器环境中这一功能的实现主要有两种方式：

- 利用URL中的hash（“#”）符号
- 利用history API在 HTML5中新增的方法

下面我们从Vue.js框架的路由插件vue-router源码入手，边看代码边看原理，由浅入深观摩vue-router是如何通过以上两种方式实现路由的。

# index#

根据官方文档的介绍，在vue-router中是通过mode这一参数控制路由的实现方式的：

```
const router = new VueRouter({
  mode: 'history',
  routes: [...]
})
```

创建VueRouter的实例对象时，mode以构造函数参数的形式传入，那么带着问题阅读源码，我们就可以从VueRouter类的定义入手。一般插件对外暴露的类都是定义在源码src根目录下的index.js文件中，打开该文件，里面果然定义了VueRouter类，摘录与mode参数有关的部分如下：

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

1. 作为参数传入的字符串属性mode只是一个标记，用来指示实际VueRouter中起作用的对象属性history，两者关系如下：

   | mode       | history         |
   | ---------- | --------------- |
   | 'history'  | HTML5History    |
   | 'hash'     | HashHistory     |
   | 'abstract' | AbstractHistory |

2. 在mode实际确定history之前，会做一个判断：若浏览器不支持HTML5History方式（通过supportsPushState变量判断），则mode强制设为'hash'；若不是在浏览器环境下运行，则mode强制设为'abstract'

3. VueRouter类中的onReady(), push()等方法，实际是调用的具体history对象的对应方法，在init()方法中初始化时，也是根据history对象具体的类别执行不同操作

故我们可以看出，在浏览器环境下的两种方式，分别就是在HTML5History，HashHistory两个类中实现的。他们都定义在src/history文件夹下，继承自同目录下base.js文件中定义的History类。History中定义的是公用和基础的方法，直接看会一头雾水，我们先从HTML5History，HashHistory两个类对常用的push(), replace()方法的实现看起，用到哪个方法看哪个。

# HashHistory

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

transitionTo()方法是父类中定义的是用来处理路由变化中的基础逻辑的方法，push()方法最主要的是对window的hash进行了直接赋值：`window.location.hash = route.fullPath`，hash的改变会自动添加到浏览器的访问历史记录中。

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

根据注释，app即为Vue组件实例，但我们知道Vue作为渐进式的前端框架，本身的组件定义中应该是没有有关路由内置属性_route，如果出现，应该是在插件加载的地方，即VueRouter的install()方法中混合入Vue对象的，查看install.js源码，有如下一段：

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

通过Vue.mixin()方法，全局注册一个混合，影响注册之后所有创建的每个 Vue 实例，该混合在beforeCreate钩子中通过Vue.util.defineReactive()定义了响应式的\_route属性，当\_route值改变时，调用Vue实例的render()方法，更新视图。

从设置路由改变到视图更新的流程如下：

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

可以看出，它与push()的实现结构上基本相似，不同点在于它不是直接对window.location.hash进行复制，而是调用window.location.replace方法将路由进行替换。

**监听路由**

以上讨论的VueRouter.push()和VueRouter.replace()是可以在vue组件的逻辑代码中直接调用的，除此之外在浏览器中，用户还可以直接在浏览器地址栏中输入改变路由，因此VueRouter还需要能监听浏览器地址栏中路由的变化，并具有与代码调用方法相同的响应行为。在HashHistory中这一功能通过setupListeners实现：

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

