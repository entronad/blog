# Fetch + GraphQL 的实践

> 本文针对在 React Native 中通过 fetch 访问 GitHub GraphQL API 这一场景，构建了应用的模型层网络请求模块，探索了 HTTP 与 GraphQL 分离、内置异常处理等理念的实践。
>
> 欢迎关注我的专栏： [熵与单子的代码本](https://zhuanlan.zhihu.com/c_90568250) 。

最近终于有时间，开始做酝酿已久的个人项目： Gitview （ [项目地址](https://github.com/entronad/gitview) ）。

Gitview 的定位是：“ GitHub 数据可视化客户端 ”；主要功能包括 GitHub 日常使用、版本库数据可视化对比、源码阅读这三块，这是平时我使用 GitHub 时，很希望在手机上能实现的需求。准备使用 React Native 来做，由于这是头一次尝试 React Native 技术栈，本着小步迭代的原则，先从基本的数据可视化功能做起。

搭建完项目后，首先着手的就是模型层的网络请求模块了。在 React Native 中网络请求通过 fetch 发起，为了使用方便，需要进行一些封装。由于 GraphQL 查询方式的特殊性，封装工作对比传统的 RESTful 接口请求有所不同，以下详细讲述在此过程中尝试的一些实践。

# GitHub GraphQL API

GitHub 最新版本的 [API v4](https://developer.github.com/v4/) 是基于 GraphQL 的，为此我前期做了一些了解学习， GitHub GraphQL API 的具体使用方式可以参考这篇文章： [《面向未来的API —— GitHub GraphQL API 使用介绍》](https://zhuanlan.zhihu.com/p/28077095) 。

之所以选用基于 GraphQL 的 API v4 ，而不是选用基于 REST 的 API v3 ，除了拥抱新技术的原则之外，很实际的原因就是：它确实更适合灵活的大数据查询。

比如我想绘制 facebook/react 近三个月的 Star 历史曲线，需要的数据是每个 Star 的创建时间构成的数组。API v3 中对应的接口是 https://api.github.com/repos/facebook/react/stargazers ，该接口每页只能返回 30 个 Stargazer 的对象，且每个 Stargazer 对象中都会包含大段完整的 User 信息。

而在 GitHub GraphQL API 中，通过在请求体中设置 query document ，每页能返回 100 个 Stargazer 对象，且 Stargazer 对象中仅包含创建时间，不仅请求数量减少到 1/3 ，每个 Response 的大小更是减小到 1/10 。

# 构建的理念

**HTTP与GraphQL分离**

GraphQL 作为一个接口查询语言，与 HTTP 协议是没有必然联系的。 GitHub GraphQL API 采用HTTP 的POST方法，以请求体中的 query 字段作为 query document 的载体，实现了GraphQL查询。

对于 GitHub GraphQL API ，所有 HTTP 请求的方法、端点、请求头、请求体格式都是一样的。这就使得我们在封装时无需处理各种变化的参数，仅需一个固定的请求构造器函数即可。

而不同的 GraphQL 查询内容、请求参数则是通过不同的 query document 体现，因此单独分离出一个对象，用以管理 GraphQL 查询部分，即 query document 的构造。

**借鉴实用功能**

相比 fetch ，目前前端应用中使用较多的还是基于 XHR 的网络请求。基于 XHR 有很多成熟的封装库，比如 axios ，它具有很多实用的功能，具体可以参考这篇文章： [《【源码拾遗】axios —— 极简封装的艺术》](https://zhuanlan.zhihu.com/p/28396592) ，其中有两点很值得借鉴：

- 可在 axios 对象中统一进行设置请求头、添加前后拦截器等操作；
- 对 request body 和 response body 中的 JSON 自动进行序列化和解析，对于使用者来说传入和返回的是 JavaScript 对象；

**异常处理**

封装好的网络请求函数，返回结果是一个 Promise 对象，如果 Promise 的结果为reject，调用者捕获不当将导致程序中断运行。 fetch 与 axios 有一个明显的不同，就是当返回码不是成功（ 2XX ）时，在 axios 中 Promise 将返回reject，即抛出异常，而在 fetch 中 Promise 结果依然是 resolve。

对于无线端，网络相对不稳定，网络异常时有发生，再加上 access token 过期、请求参数无效等，返回结果不成功也很常见。且不同于浏览器环境，如果这些失败以异常抛出，捕获不当很容易引起应用崩溃。

因此在封装网络模块时，我比较赞同 fetch 的理念，不将返回错误作为异常抛出。并且我希望网络异常等也在网络请求模块内部捕获，并给出类似返回错误的对象，方便调用者统一处理。

此外，在 GitHub GraphQL API 中，如果 GraphQL 查询内部发生了错误，返回码依然是 2XX ，只是返回体中 data 字段是 null ，且会多出一个 errors 字段，这种情况也需要甄别出来进行处理。

总之就是发生各种类型异常，都在网络模块内部进行处理，返回结构统一的对象，异常类型用一个返回码区别。这样方便调用者统一处理，且避免处理不当导致的应用奔溃。



最终网络请求模块暴露给调用者的，是针对不同需求抽象出的请求函数，输入为请求必需的参数，返回 Promise 对象，结果都是 resolve ，其 value 对象隐藏了网络请求细节和异常的处理，仅返回标识成功或异常类型的返回码、数据模型。

# Query Documents

接下去就是实际的项目构建了，先以搜索版本库、版本库基本信息、 Star 历史这三个需求为例。

GraphQL 查询请求的核心是被称为 query document 的内容，它描述了所要查询的数据和输入变量。我希望以类似 Redux 中 action creator 的方式，通过一个构造器函数，传入查询参数，获得所需的 query document 字符串；另外要注意的是，在 GraphQL 语法中，字段的间隔是通过换行符而不是逗号实现的。

这两个要求可以通过模板字面量语法（ \`\` ）很好的满足。模板字面量中可通过 ${} 插入任意的 JavaScript 表达式，且其输出的字符串中将保留源码中的所有空格和换行符。

我们把生成各 query document 的构造器函数放在一个对象中方便引用：

```
// api/documents.js

export default {
  querySearchRepos: ({ query, after }) => `
    query { 
      search(query: "${query}", type: REPOSITORY, first: 100${after ? `, after: "${after}"` : ''}) {
        repositoryCount
        pageInfo {
          hasNextPage
          endCursor
        }
        edges {
          node {
            ... on Repository {
              id
              name
              description
              viewerHasStarred
              stargazers {
                totalCount
              }
            }
          }
        }
      }
    }
  `,
  queryRepoOverview: ({ owner, name }) => `
    query { 
      repository(owner:"${owner}", name: "${name}") {
        description
        stargazers {
          totalCount
        }
        watchers {
          totalCount
        }
        forks {
          totalCount
        }
      }
    }
  `,
  queryStargazers: ({ owner, name, before }) => `
    query { 
      repository(owner:"${owner}", name: "${name}") {
        stargazers(last: 100${before ? `, after: "${before}"` : ''}) {
          totalCount
          pageInfo {
            startCursor
            hasPreviousPage
          }
          edges {
            starredAt
          }
        }
      }
    }
  `,
}
```

可以看到，对于各个具体的请求， query document 的主体已预先写好，构造器函数通过模板字面量将参数插入为查询参数，且可灵活的根据请求参数是否为空决定是否有某个查询参数。

# 请求构造器

由于所有的 GraphQL 查询中， HTTP 请求的方法、端点、请求头、请求体格式都是一样的，故无需像 axios 那样创建功能强大的请求构造器对象，仅需一个函数创建 fetch 请求，并返回相应的 Promise 即可。采用 async / await 语法，可以直观清晰的列出异步操作的先后顺序，结构大致如下：

```
import documents from './documents';
const GRAPHQL_URL = 'https://api.github.com/graphql';

const createCall = async (document, accessToken) => {
  const payload = JSON.stringify({
    query: document,
  });
  let response; 
  response = await fetch(
    GRAPHQL_URL,
    {
      method: 'POST',
      headers: {
        Authorization: `token ${accessToken}`,
      },
      body: payload,
    }
  );
  body = await response.json();
  return {
    status: response.status,
    ok: response.ok,
    body: body.data,
  };
}
```

其中对于请求体和返回体，添加了 JSON 序列化与解析的处理，如果有所有请求都添加的拦截器，可通过 await 语句在 fetch 前后添加。

整个网络请求模块暴露的是对应具体请求的函数：

```
export const querySearchRepos = ({ query, after }, accessToken) => 
  createCall(documents.querySearchRepos({ query, after }), accessToken);

export const queryRepoOverview = ({ owner, name }, accessToken) => 
  createCall(documents.queryRepoOverview({ owner, name }), accessToken);

export const queryStargazers = ({ owner, name, before }, accessToken) => 
  createCall(documents.queryStargazers({ owner, name, before }), accessToken);
```

# 异常处理

**异常类型**

对于通过 HTTP 协议请求 GraphQL 查询，异常处理是一件比较麻烦的事，因为你需要将 GraphQL 查询错误、 HTTP 请求错误、网络请求失败等问题一一甄别，分别处理。 GitHub GraphQL API 请求中，各种异常具体如下：

- GraphQL 查询错误

  特征：返回码为200，返回体 data 字段为 null ，返回体对象中有一个 errors 字段

  例子： query document 语法错误错误


- HTTP 错误

  特征：返回码为 4XX 或 5XX ，返回体有 message 和 documentation_url 两个字段

  例子：请求体 JSON 解析错误； access token 失效等

- 网络异常

  特征： fetch 方法抛出异常

  例子：手机无网络信号

- JSON 解析异常

  特征： Response.json() 方法抛出异常

  例子：由于 fetch 方法一旦开始接收到返回信息就会立刻 resolve ， Response.json() 的解析过程就会开始，如果此时网络突然中断，未能接受到全部的返回体，则 Response.json() 的解析就会失败抛出异常

**返回对象**

Fetch 返回的原生 Response 对象中，大部分信息是关于网络处理的。而对于网络请求模块的调用者来说，他希望面对的是一个抽象的模型层，并不关心网络处理的细节信息。因此 Promise 的 value 我并不是直接返回原生 Response 对象，而是按照相同的结构重新构造对象，仅保留有用的信息，这种重新构造的步骤也方便了异常的处理：

正常：

```
{ 
  status: 2XX,
  ok: true,
  body: {
    ...
  }
}
```

异常：

```
{ 
  status: 4XX,
  ok: false,
  body: {
    message: 'Not Found'
  }
}
```

**处理策略**

为了统一通过 status 字段分辨异常类型，仿照 HTTP 的返回码自定义两个新的 status 值：

```
const MY_STATUS = {
  NET_ERROR: 499,
  GRAPHQL_ERROR: 599,
};
```

分别对应网络异常、 GraphQL 查询错误。这两个虽然不是 HTTP 协议标准，但与 HTTP 协议中 4XX 为客户端错误、 5XX 为服务端错误的语义保持一致。

各类型异常处理策略如下：

- 如果 HTTP 错误， status 为对应错误码， message 为 HTTP 返回体的 message ；
- 如果网络异常， status 为 499 ， message 为异常对象的 message ；
- 如果 GraphQL 查询错误， status 为 599 ， message 为 HTTP 返回体中 errors[0] 的 message ；
- 如果解析 JSON 错误，返回 499 ， message 为异常对象的 message ；

这样请求构造器函数改造为：

```
// api/index.js

const createCall = async (document, accessToken) => {
  const payload = JSON.stringify({
    query: document,
  });
  let response; 
  try {
    response = await fetch(
      GRAPHQL_URL,
      {
        method: 'POST',
        headers: {
          Authorization: `token ${accessToken}`,
        },
        body: payload,
      }
    );
  } catch (e) {
    return {
      status: MY_STATUS.NET_ERROR,
      ok: false,
      body: {
        message: e.message,
      },
    };
  }
  let body;
  try {
    body = await response.json();
  } catch (e) {
    return {
      status: MY_STATUS.NET_ERROR,
      ok: response.ok,
      body: {
        message: e.message,
      },
    };
  }
  if (!response.ok) {
    return {
      status: response.status,
      ok: response.ok,
      body: {
        message: body.message,
      }
    };
  }
  return body.data
    ? {
      status: response.status,
      ok: response.ok,
      body: body.data,
    }
    : {
      status: MY_STATUS.GRAPHQL_ERROR,
      ok: false,
      body: {
        message: body.errors[0].message
      }
    };
}
```

**测试结果**

这样模拟各种可能的情况，并用 logcat 输出网络函数的返回对象如下：

```
// logcat out

// 正常
{ status: 200,
  ok: true,
  body: 
   { repository: 
      { description: 'A progressive, incrementally-adoptable JavaScript framework for building UI on the web.',
        stargazers: { totalCount: 73396 },
        watchers: { totalCount: 4034 },
        forks: { totalCount: 10497 } } } }

// 断网
{ status: 499,
  ok: false,
  body: { message: 'Network request failed' } }

// 写错端点
{ status: 404, ok: false, body: { message: 'Not Found' } }

// document写错
{ status: 599,
  ok: false,
  body: { message: 'Parse error on "quer" (IDENTIFIER) at [2, 5]' } }
```

符合我们的策略。

# 获取 Access Token

在 GitHub GraphQL API 中没有登录与授权的相关接口，因此登录获取 access token 依然要使用用 API v3 。对应的接口为 https://api.github.com/authorizations/clients/:client_id ，采用 Basic Auth 的方式，将用户名与密码进行 Base64 加密后作为请求头的 Authorization 字段发送，以获取 access token 。

在浏览器环境中， Base64 加密可使用 window.btoa() 方法，但 React Native 环境下没有该方法，因此引入 buffer 这个 npm 库，使用其 new Buffer(str, 'base64') 方法进行 Base64 加密。由于只有一个固定的函数，就直接写出：

```
// api/index.js

export const login = async ({
  username,
  password,
  footprint
}) => {
  const payload = JSON.stringify({
    client_secret: client.secret,
    scopes: [
    'user',
    'repo',
    'delete_repo',
    'notifications',
    'gist',
    'admin:repo_hook',
    'admin:org_hook',
    'admin:org',
    'admin:public_key',
    'admin:gpg_key',
    ],
    note: `${username} on Gitview`,
    footprint,
  });
  let response;
  try {
    response = await fetch(
      LOGIN_URL,
      {
        method: 'PUT',
        headers: {
          Authorization: getBasicAuth({
            username,
            password,
          }),
        },
        body: payload,
      }
    );
  } catch (e) {
    return {
      status: MY_STATUS.NET_ERROR,
      ok: false,
      body: {
        message: e.message,
      },
    };
  }
  let body;
  try {
    body = await response.json();
  } catch (e) {
    return {
      status: MY_STATUS.NET_ERROR,
      ok: response.ok,
      body: {
        message: e.message,
      },
    };
  }
  if (!response.ok) {
    return {
      status: response.status,
      ok: response.ok,
      body: {
        message: body.message,
      }
    };
  }
  return {
    status: response.status,
    ok: response.ok,
    body,
  };
}
```



以上就是 Gitview 中模型层网络请求模块的大致结构，它是针对在 React Native 中通过 fetch 访问 GitHub GraphQL API 这一场景进行的封装实践。由于 GraphQL 与 REST 在理念上的差异，要想最大化 GraphQL 的效果，设计出合理的请求函数划分，还需要仔细研究研究。

项目源码请见： [Gitview](https://github.com/entronad/gitview) 

