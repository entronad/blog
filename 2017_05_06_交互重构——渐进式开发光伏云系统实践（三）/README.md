![0](0.jpg)

## 交互逻辑的简化

在偏中后台系统的交互设计中，多层目录导航是一种常见的内容选择方式，通过左侧或顶部导航栏多层目录的深入，确定主要内容区域的路由跳转。一般的管理平台中，主要内容区域所针对的对象是唯一确定的，比如代码托管平台中的某个项目，商城应用管理平台中的某个App。

传统的光伏云系统的导航有两种做法：一种是根目录选择功能模块，目录的末枝为选择电站，比如“运行监测->发电量监测->XX电站”；另一种是根目录选择电站，然后一步步选择功能，比如“XX电站->运行监测->发电量监测”。这两者的共同点为选择功能的逻辑与选择对象的逻辑在一个体系中，带来的问题是切换较为麻烦，需要重新爬目录，比如第二个例子中，我想对比看另一个电站的发电量，就需要重新从根目录选起，视图肯定要重新渲染。

关于这一问题个人的看法是：传统的前端开发技术，大多是围绕着DOM操作，从根节点开始层次分明的DOM树的概念潜移默化影响着开发者，故视图的逻辑设计也自然以此为蓝本。

然而近年来，随着三大MVVM框架的普及，数据驱动视图的概念逐渐开始深入人心。数据双向绑定、单向数据流、申明式编程、前端状态管理等技术也使得前端开发的思维方式发生了翻天覆地的变化。关于这些技术推荐阅读文章[《GUI 应用程序架构的十年变迁：MVC、MVP、MVVM、Unidirectional、Clean》](https://zhuanlan.zhihu.com/p/26799645)

较为理想的交互最好是：当我点击另一个电站时，我当前的视图结构不要发生变化，还是我选择的那些功能，只是数据切换为新选电站的；当我选择其他功能模块时，系统记得我当前所选的电站，新功能的视图依然渲染的是该电站的数据。

通过前端的路由管理与状态管理技术可以很自然的实现这样的交互体验。将电站对象的选择与功能的选择分离到左侧和顶部独立的导航栏中，两者共同决定主要内容区域渲染的组件和数据。同时功能的选择也不采用多层目录的选择方式，而是将不同的功能模块以组件的形式放置在主要内容区域中，尽可能实现用户可定制化选择当前视图中有哪些功能组件。结构如下：

![1](1.png)



**状态管理**

本系统前端视图层框架选用Vue.js，对应的路由管理为vue-router，状态管理为vuex。通过顶部导航栏切换路由实现主要内容区域大框架的切换。通过store中的user模块，存储主视图上所需要的状态。

用户信息、token等存储在localStorage中，以便自动登录，应用初始化的时候状态从localStorage中获取用户信息，并通过Ajax异步获取电站列表。状态中保存着当前的电站。由于用户电站数量可能较多，不需要全部显示在列表中，所以还有个indexesOnDesk。user模块如下：

```
const state = {
  id: null,
  loginName: null,
  plants: [],
  indexesOnDesk: [],
  currentIndex: 0,
  overviewIsShown: true
}

// getters
const getters = {
  currentPlant: state => state.plants[state.currentIndex]
}

// actions
// 所有 action 都为 async ,返回值为成功与否的状态码
const actions = {

  async updateUserInfo({commit}) {
    const authStr = window.localStorage.getItem('auth')
    if (authStr) {
      const auth = JSON.parse(authStr)
      if (auth.id && auth.loginName) {
        commit('SET_USER_INFO', {
          id: auth.id,
          loginName: auth.loginName
        })
        return 200
      }
    }
    return 404
  },

  async updatePlants({state, commit}) {
    const response = await api.getPlantList({userId: state.id})
    // 401 就api就近处处理
    if (response.status == 200) {
      // 提交电站列表，并重新初始化indexesOnDesk和currentIndex
      commit('SET_PLANTS', response.data.plants)
      return response.status
    }
    if (response.status == 401) {
      window.localStorage.removeItem("auth")
      this.$router.replace({name: 'login'})
      return response.status
    }
    return response.status
  }
}

// mutations
const mutations = {

  [types.SET_USER_INFO](state, {id, loginName}) {
    // id, loginName 需一致修改
    state.id = id
    state.loginName = loginName
  },

  [types.SET_PLANTS](state, plants) {
    // plants, indexesOnDesk, currentIndex 上一级的set后以下级都要重置
    state.plants = plants
    state.indexesOnDesk = plants.reduce(
      (indexesOnDesk, value, index) => {
        indexesOnDesk.push(index)
        return indexesOnDesk
      },
      []
    )
    if (!state.indexesOnDesk.includes(state.currentIndex)) {
      state.currentIndex = state.indexesOnDesk[0]
    }
  },

  [types.SET_INDEXES_ON_DESK](state, indexesOnDesk) {
    // indexesOnDesk, currentIndex 默认能选中传进来的已是合法值
    state.indexesOnDesk = indexesOnDesk
    if (!state.indexesOnDesk.includes(state.currentIndex)) {
      state.currentIndex = state.indexesOnDesk[0]
    }
  },

  [types.SET_CURRENT_INDEX](state, currentIndex) {
    // indexesOnDesk, currentIndex 默认能选中传进来的已是合法值
    state.currentIndex = currentIndex
  },
  
  [types.SET_OVERVIEW_IS_SHOWN](state, overviewIsShown) {
    state.overviewIsShown = overviewIsShown
  },

  [types.CLEAR_ALL](state) {
    state = {
      id: null,
      loginName: null,
      plants: [],
      indexesOnDesk: [],
      currentIndex: 0,
      overviewIsShown: true
    }
  }
}

export default {
  state,
  getters,
  actions,
  mutations
}
```

**异步action**

值得一提的是，action并不要求一定是异步的，但为我习惯所有Action都封装为async异步函数，返回Promise，含有Ajax请求的action返回的Promise中填入http返回码，不含Ajax请求的action也根据分发是否成功填入对应的http返回码，这样便于外面处理的统一。

## 模块组件的自然排列

上文提到，本系统的各子功能以组件的方式放置于主要内容区域，那么这些模块的排布就需要一个统一而合理的方式：

- 为了让光伏云系统的使用体验上更接近“操作台”而不是“网页”，我固定视图的纵向宽度，增加的模块组件横向向右增长，视图可在横向上滑动。
- 组件的大小也进行统一的规范，设计small和large两种规格，约为1/2的关系，一个large组件可占满纵向一列，两个small视图占满一列。这样大小两种模块组件，既能保证视觉上的统一整齐，又可满足不同的业务需求。

![2](2.png)

**Flex布局实现**

当要将模块组件在容器中动态增减排列，我自然想到的是Flex布局。下图可以很好的概括Flex布局中的基本概念（图片来自阮一峰博客）：

![3](3.png)

Flex容器主要的属性如下：

> - flex-direction：决定主轴的方向（即项目的排列方向）
> - flex-wrap：如果一条轴线排不下，如何换行
> - flex-flow：flex-direction属性和flex-wrap属性的简写形式
> - justify-content：项目在主轴上的对齐方式
> - align-items：项目在交叉轴上如何对齐
> - align-content：多根轴线的对齐方式。如果项目只有一根轴线，该属性不起作用
>

Flex项目的主要属性如下：

> - order：项目的排列顺序。数值越小，排列越靠前，默认为0
> - flex-grow：项目的放大比例，默认为0，即如果存在剩余空间，也不放大
> - flex-shrink：项目的缩小比例，默认为1，即如果空间不足，该项目将缩小
> - flex-basis：在分配多余空间之前，项目占据的主轴空间（main size）。浏览器根据这个属性，计算主轴是否有多余空间。它的默认值为auto，即项目的本来大小
> - flex：flex-grow, flex-shrink 和 flex-basis的简写
> - align-self：允许单个项目有与其他项目不一样的对齐方式

经过一些摸索与试验，主要内容区域的Flex布局如下实现：

Flex容器的style：

```
.container {
  display: flex;
  flex-direction: column; /* 容器内项目排列为纵向，自上而下 */
  flex-wrap: wrap; /* 当一行排不下时向右换行，通过这两个属性使得项目按从左到右、自上而下，满格换行的自然排布 */
  justify-content: flex-start; /* 项目在主轴上起点对齐 */
  align-items: flex-start; /* 项目交叉轴上起点对齐 */
  align-content: flex-start; /* 多根主轴起点对齐，即每列都向左靠 */
  overflow-x: auto; /* 容器通过滑动条向右扩展 */
  overflow-y: auto; /* 因为规定了换行，所以当y方向项目放不下时会换行，但是一列至少要放一个项目，当一列的高度内连一个项目也放不下时，这样设置可以启用滑动条 */
  width: 100%;
  height: 100%;
  padding: 5px;
}
```

Flex项目的style（以small版为例）：

```
.item-small {
  flex-basis: 47%; /* 垂直方向上的高度自动撑到一半略少一点，以便给margin留下空间 */
  min-height: 240px; /* 最小高度不得低于项目中内容区的高度，在Flex容器高度过小时不再压缩Flex项目而是启用滑动条 */
  margin: 0.5%; /* margin设置计算见下文 */
  width: 600px; /* 内容的宽度为定值 */
  display: flex; /* 项目本身依然设置为Flex布局，方便定义内容区居中显示 */
  align-items: center; /* 定义内容区垂直居中 */
  justify-content: center; /* 定义内容区水平居中 */
}
```

**边距比例的混合设置**

通过给项目的flex-basis、margin值设置百分比，可以实现在不同分辨率下项目都能很好的充满容器（容器一般都要求充满屏幕，故高度的绝对值也是不定的），这也正是flexible的意义之一，但如果仅仅这样设置，视觉上会产生两点别扭之处：

1. 我们希望项目本体随着分辨率的变化可以自适应的变大变小，但边距最好固定一点，比如macbook pro分辨率达到2560X1600，但普通工作站分辨率只有1369X768，项目高度扩大了一倍，如果边距也随着扩大一倍看起是不太舒服的，最好在不同分辨率下都是定值，比如10px
2. 我们希望项目-项目之间的边距与项目-容器边缘的边距一样，在Flex布局中没有margin覆盖，仅设置margin值项目-项目之间的边距会是项目-容器之间边距的两倍

针对以上问题，想到的最简单的方法就是容器的padding和项目的margin设为相等的定值，比如都为5px，flex-basis设为一个较小的值确保低分辨率也能装下，然后设置flex-grow为1，让项目拉伸填满剩余空间，但如果设置了flex-grow，若最后一列只有一个项目，该项目则会拉满整列，这不是我们想要的。

只通过简单的属性设置，能同时完美实现比例值的弹性灵活与绝对值的固定精确，的确是鱼与熊掌不可兼得，思考多种解决方案后，我考虑了如下的解决方案（以一列排两个的small版为例，large版类似）：

1. 容器padding设为绝对值5px；项目margin设为百分比0.5%

   对于光伏云系统实际的主内容区，这两个值比较接近，在1369X768到1920X1080主流分辨率下近似能够实现项目-项目之间的边距与项目-容器边缘的边距差不多。

2. 项目flex-basis设为百分比47%，不设flex-grow，min-height设为绝对值240px

   47%即为项目的实际大小，根据分辨率或浏览器窗口大小自动适应，一列如果只有一项也不会拉伸变形，设置一个绝对值的min-height是可以防止项目所得太小挤压内部的内容。

对于将百分比换算为实际值，容器padding和内部项目的margin基数不一样，即使通过方程组去求解，考虑到取值位数等因素，也是很难做到”严丝合缝“的，故通过这种绝对值和百分比混合的方式，既做到尽可能的布满视图、平均边距，各数值又留有一定的裕量提高针对分辨率变动的鲁棒性。实际排布效果如下：

![4](4.png)

**抽取组件**

在实际开发中，我将刚刚所设计的Flex布局项抽象为vue组件，并引入iview库中的`<Card>`标签以利实现圆角、阴影等特性：

```
<template>
  <Card class="flex-card" :class="{ 'flex-card-large': large }">
    <div class="content" :class="{ 'content-large': large }">
      <slot></slot>
    </div>
  </Card>
</template>

<script>
export default {
  props: {
    large: {
      type: Boolean,
      default: false
    }
  }
}
</script>

<style scoped>
.flex-card {
  flex-basis: 47%;
  min-height: 240px;
  margin: 0.5%;
  width: 600px;
  display: flex;
  align-items: center;
  justify-content: center;
}
.flex-card-large {
  flex-basis: 98%;
  min-height: 480px;
}
.content {
  height: 220px;
  width: 580px;
  margin: -6px;
}
.content-large {
  height: 460px;
}
</style>
```

该组件命名为FlexCard，内容插槽可放入各类业务模块，通过组件参数large的真值，即可设置该组件是small还是large类型