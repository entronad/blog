> 

最近终于有时间，开始做酝酿已久的个人项目： Gitview （[项目地址](https://github.com/entronad/gitview)）。

Gitview 的定位是：“ GitHub 数据可视化客户端 ”；主要功能是 GitHub 基本使用、项目数据可视化对比、源码阅读这三块，是平时我使用 GitHub 时，很希望在手机上能实现的需求。项目准备使用 React Native 来做，由于这是头一次尝试 React Native 技术栈，本着小步迭代的原则，我准备先从基本的数据可视化功能做起。

搭建完项目后，首先着手的就是模型层网络请求部分了。在 React Native 中网络请求通过 Fetch 发起，为了使用方便，需求进行一些封装工作，由于 GraphQL 查询方式的特殊性，封装工作对比于传统的 RESTful 接口请求的封装有所不同，以下详细讲述我在封装过程中尝试的一些实践。

# GitHub GraphQL API

GitHub最新版本的 [API v4](https://developer.github.com/v4/) 是基于 GraphQL 的，为此前期做了一些了解准备， GitHub GraphQL API 的具体使用方式可以参考这篇文章：[《面向未来的API —— GitHub GraphQL API 使用介绍》](https://zhuanlan.zhihu.com/p/28077095)。

之所以选用基于 GraphQL 的 API v4 ，而不是选用 基于 REST 的 API v3，除了拥抱新技术的原则之外，很实际的原因就是它确实更适合灵活的大数据查询：

比如我想绘制一个 facebook/react 的项目的 star 历史曲线，我们需要的数据是每个star的创建时间构成的数组。v3 版本中对应的API 是https://api.github.com/repos/facebook/react/stargazers，该API每页只能返回30个star的信息，且每个star信息中都会包含大段完整的user信息。而在v4版本的GraphQL中，通过在请求体中设置document，每页能返回100个star的信息，且star信息中仅包含创建时间，不仅请求数量减少到1/3，每个response的大小更是减小到原来的1/10。

此外，API v3的请求限制为固定的每小时5000个，以上面为例最多查询到15万个star，而API v4采用按照查询节点计分的方式设置请求限制，以上面为例每个请求只会消耗0.01分



# 封装的理念





# 特点理念

相对固定的http请求（方法、头、端点、请求响应体都固定）

网络引起的异常很“正常”，不要中断程序运行

引入axios的先进理念：自动json处理，拦截器（在createCall函数中前后加await）

# 创建document



# 权限登录

​	window.btoa(str)//转码结果 "amF2YXNjcmlwdA=="window.atob("amF2YXNjcmlwdA==")，rn中无此方法

# 异常处理