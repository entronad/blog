![0](0.png)

# 面向未来的API —— GitHub GraphQL API 使用介绍

> 本文根据GitHub开发者文档，整理翻译了GitHub GraphQL API的使用方法，你可以了解到GraphQL的基本概念、GitHub GraphQL API的使用，两个实际的使用案例，以及使用Explorer查询GitHub GraphQL API

今年5月22日，GitHub[发文](https://github.com/blog/2359-introducing-github-marketplace-and-more-tools-to-customize-your-workflow)宣布，去年推出的GitHub GraphQL API已经正式可用（production-ready），并推荐集成商在GitHub App中使用最新版本的GraphQL API v4。

相信大家对[GraphQL](http://facebook.github.io/graphql/)早已不陌生，这一Facebook推出的接口查询语言，立志在简洁性和扩展性方面超越REST，并且已经被应用在很多复杂的业务场景中。GitHub这样描述他们为何对GraphQL青睐有加：

> 我们为API v4选择GraphQL，是因为它为我们的集成商提供了显著的灵活性。相比于REST API v3，它最强大的优势在于，你能够精确的定义所需要的数据，并且毫无冗余。通过GraphQL，你只需要一次请求就能取到通过多个REST请求才能获得的数据。

在GitHub的[开发者文档](https://developer.github.com/v4/)中有较为完整的GraphQL API v4介绍，本文整理并翻译了其中的部分内容，并按以下章节组织，希望能给入门GraphQL或有兴趣从事GitHub App开发的同学以参考：

- 概念解释
- 使用方法
- 示范案例
- 文档指引

# 概念解释

 GitHub GraphQL API v4在架构上和概念上与GitHub REST API v3有很大不同，在GraphQL API v4的文档中，你会遇到很多新的概念。

**Schema**

schema定义了GraphQL API的类型系统。它完整描述了客户端可以访问的所有数据（对象、成员变量、关系、任何类型）。客户端的请求将根据schema进行校验和执行。客户端可以通过“自省”（introspection）获取关于schema的信息。schema存放于GraphQL API服务器。

**Field**

field是你可以从对象中获取的数据单元。正如GraphQL官方文档所说：“GraphQL查询语言本质上就是从对象中选择field”。

关于field，官方标准中还说：

> 所有的GraphQL操作必须指明到最底层的field，并且返回值为标量，以确保响应结果的结构明白无误

*标量（scalar）：基本数据类型*

也就是说，如果你尝试返回一个不是标量的field，schema校验将会抛出错误。你必须添加嵌套的内部field直至所有的field都返回标量。

**Argument**

argument是附加在特定field后面的一组键值对。某些field会要求包含argument。mutation要求输入一个object作为argument。

**Implementation**

GraphQL schema可以使用implement定义对象继承于哪个接口。

下面是一个人为的schema示例，定义了接口 `X` 和对象 `Y` ：

```
interface X {
  some_field: String!
  other_field: String!
}

type Y implements X {
  some_field: String!
  other_field: String!
  new_field: String!
}
```

这表示对象`Y`除了添加了自己的field外，也要求有接口`X`的field/argument/return type.（`!`代表该field是必须的）

**Connection**

connection让你能在同一个请求中查询关联的对象。通过connection，你只需要一个GraphQL请求就可以完成REST API中多个请求才能做的事。

为帮助理解，可以想象这样一张图：很多点通过线连接。这些点就是node，这些线就是edge。connection定义node之间的关系。

**Edge**

edge表示node之间的connection。当你查询一个connection时，你通过edge到达node。每个`edges`field都有一个`node`field和一个`cursor`field。cursor是用来分页的。

**Node**

node是对象的一个泛型。你可以直接查询一个node，也可以通过connection获取相关node。如果你指明的`node`不是返回标量，你必须在其中包含内部field直至所有的field都返回标量。

# 基本使用

**发现GraphQL API**

GraphQL是可自省的，也就是说你可以通过查询一个GraphQL知道它自己的schema细节。

- 查询`__schema`以列出所有该schema中定义的类型，并获取每一个的细节：

  ```
  query {
    __schema {
      types {
        name
        kind
        description
        fields {
          name
        }
      }
    }
  }
  ```

- 查询`__type`以获取任意类型的细节：

  ```
  query {
    __type(name: "Repository") {
      name
      kind
      description
      fields {
        name
      }
    }
  }
  ```

> 提示：自省查询可能是你在GraphQL中唯一的`GET`请求。不管是query还是mutation，如果你要传递请求体，GraphQL请求方式都应该是`POST`

**GraphQL 授权**

要与GraphQL服务器通讯，你需要一个对应权限的OAuth token。

通过命令行创建个人access token的步骤详见[这里](https://help.github.com/articles/creating-a-personal-access-token-for-the-command-line/)。你访问所需的权限具体由你请求哪些类型的数据决定。比如，选择User权限以获取用户数据。如果你需要获取版本库信息，选择合适的Repository权限。

当某项资源需要特定权限时，API会通知你的。

**GraphQL 端点**

REST API v3有多个端点，GraphQL API v4则只有一个端点：

```
https://api.github.com/graphql
```

不管你进行什么操作，端点都是保持固定的。

**与 GraphQL 通讯**

在REST中，HTTP动词决定执行何种操作。在GraphQL中，你需要提供一个JSON编码的请求体以告知你要执行query还是mutation，所以HTTP动词为POST。自省查询是一个例外，它只是一个对端点的简单的GET请求。

**关于 query 和 mutation 操作**

在GitHub GraphQL API中有两种操作：query和mutation。将GraphQL类比为REST，query操作类似`GET`请求，mutation操作类似`POST`/`PATCH`/`DELETE`。mutation mame决定执行哪种改动。

query和mutation具有类似的形式，但有一些重要的不同。

**关于 query**

GraphQL query只会返回你指定的data。为建立一个query，你需要指定“fields within fields"（或称嵌套内部field）直至你只返回标量。

query的结构类似：

```
query {
  JSON objects to return
}
```

**关于 mutation**

为建立一个mutation，你必须指定三样东西：

1. mutation name：你想要执行的修改类型
2. input object：你想要传递给服务器的数据，由input field组成。把它作为argument传递给mutation name
3. payload object：你想要服务器返回给你的数据，由return field组成。把它作为mutation name的body传入

mutation的结构类似：

```
mutation {
  mutationName(input: {MutationNameInput!}) {
    MutationNamePayload
}
```

此示例中input object为`MutationNameInput`，payload object为`MutationNamePayload`.

**使用 variables**

variables使得query更动态更强大，同时他能简化mutation input object的传值。

以下是一个单值variables的示例：

```
query($number_of_repos:Int!) {
  viewer {
    name
     repositories(last: $number_of_repos) {
       nodes {
         name
       }
     }
   }
}
variables {
   "number_of_repos": 3
}
```

使用variables分为三步：

1. 在操作外通过一个`variables`对象定义变量：

   ```
   variables {
      "number_of_repos": 3
   }
   ```

   对象必须是有效的JSON。此示例中只有一个简单的`Int`变量类型，但实际中你可能会定义更复杂的变量类型，比如input object。你也可以定义多个变量。

2. 将变量作为argument传入操作：

   ```
   query($number_of_repos:Int!){
   ```

   argument是一个键值对，键是`$`开头的变量名（比如`$number_of_repos`），值是类型（比如`Int`）。如果类型是必须的，添加`!`。如果你定义了多个变量，将它们以多参数的形式包括进来。

3. 在操作中使用变量：

   ```
   repositories(last: $number_of_repos) {
   ```

   在此示例中，我们使用变量来代替获取版本库的数量。在第2步中我们指定了类型，因为GraphQL强制使用强类型。

这一过程使得请求参数变得动态。现在我们可以简单的在`variables`对象中改变值而保持请求的其它部分不变。

用变量作为argument使得你可以动态的更新`variables`中的值但却不用改变请求。

# 示范案例

**query 示例**

让我们来完成一个更复杂的query。

以下query查找`octocat/Hellow-World`版本库，找到最近关闭的20个issue，并返回每个issue的题目、URL、前5个标签：

```
query {
  repository(owner:"octocat", name:"Hello-World") {
    issues(last:20, states:CLOSED) {
      edges {
        node {
          title
          url
          labels(first:5) {
            edges {
              node {
                name
              }
            }
          }
        }
      }
    }
  }
}
```

让我们一行一行的来看各个部分：

- `query {`

  因为我们想要从服务器读取而不是修改数据，所以根操作为`query`。（如果不指定一个操作，默认为`query`）

- `repository(owner:"octocat", name:"Hello-World") {`

  为开始我们的query，我们希望找到`repository`对象。schema校验指示该对象需要`owner`和`name`参数

- `issues(last:20, states:CLOSED) {`

  为计算该版本库的所有issue，我们请求`issue`对象。（我们可以请求某个`repository`中某个单独的`issue`，但这要求我们知道我所需返回issue的序号，并作为argument提供。）

  `issue`对象的一些细节：

  - 根据文档，该对象类型为`IssueConnection`。
  - schema校验指示该对象需要一个结果的`last`或`first`数值作为argument，所以我们提供`20`。
  - 文档还告诉我们该对象接受一个`states` argument，它是一个`IssueState`的枚举类型，接受`OPEN`或`CLOSED`值。为了只查找关闭的issue，我们给`states`键一个`CLOSED`值。

- `edges {`

  我们知道`issues`是一个connection，因为它的类型为`IssueConnection`。为获取单个issue的数据，我们需要通过`edges`取得node。

- `node {`

  我们从edge的末端获取node。`IssueConnection`的文档指示`IssueConnection`类型末端的node是一个`issue`对象。

- 既然我们知道了我们要获取一个`Issue`对象，我们可以查找文档并指定我们想要返回的field：

  ```
  title
  url
  labels(first:5) {
    edges {
      node {
        name
      }
    }
  }
  ```

  我们指定`Issue`对象的`title`，`url`，`labels` field。

  `labels` field类型为`LabelConnection`。和`issue`对象一样，由于`labels`是一个connection，我们必须遍历它的edge以到达连接的node：`label`对象。在node上，我们可以指定我们想要返回的`label`对象field，在此例中为`name`。

你可能注意到了在这个Octocat的公开版本库`Hellow-World`中运行这个query不会返回很多label。试着在你自己的有label的版本库中运行它，你就会看到差别了。

**mutation 示例**

mutation往往需要你先通过执行query获取请求信息。本示例有两个操作：

1. 通过query获取issue ID
2. 通过mutation给issue添加一个emoji表情

```
query FindIssueID {
  repository(owner:"octocat", name:"Hello-World") {
    issue(number:349) {
      id
    }
  }
}

mutation AddReactionToIssue {
  addReaction(input:{subjectId:"MDU6SXNzdWUyMzEzOTE1NTE=",content:HOORAY}) {
    reaction {
      content
    }
    subject {
      id
    }
  }
}
```

> 不可能在执行一个query的同时执行一个mutation，反之亦然。

让我们看一遍这个示例。目标看起来很简单：给一个issue添加一个emoji表情。

那么我们怎么知道首先需要一个query的？我们目前还不知道。

因为我们想要改动服务器上的数据（给issue添加一个emoji），我们首先从schema中查找有用的mutation。文档显示有`addReaction`这一mutation，描述为： `Adds a reaction to a subject.` ，很好！

该mutation的文档列出了三个input field：

- `clientMutationId` (`String`)
- `subjectId` (`ID!`)
- `content` (`ReactionContent!`)

`!`表明`subjectId`和`content`是必需的。`content`是必需的很好理解：我们要添加表情，肯定要指明使用哪个emoji。

但是为什么`subjectId`也是必需的？因为`subjectId`是标明要给哪个版本库中的哪个issue添加表情的唯一方式。

这就是为什么在示例中首先要有一个query：为的是获取`ID`。

让我们一行一行的来看这个query：

- `query FindIssueID {`

  这里我们执行一个query，我们将它命名为 `FindIssueID`。给query命名是非必需的。

- `repository(owner:"octocat", name:"Hello-World") {`

  我们通过查询 `repository` 对象并传入 `owner` 和 `name` 两个argument来指定版本库。

- `issue(number:349) {`

  我们通过查询 `issue` 对象并传入 `number`argument来指定所要添加表情的issue。

- `id`

  这就是我们从 `https://github.com/octocat/Hello-World/issues/349`获取并作为`subjectId的`传递的`id`。

当我们执行这个query，就能得到 `id`: `MDU6SXNzdWUyMzEzOTE1NTE=`

> 注意：从query中返回的`id`就是我们将要在mutation中作为`subjectID`传递的值。文档和schema自省都不会指明这个关系；你需要明白这些名称背后的概念才能弄清楚。

知道了ID，我们就可以进行mutation了：

- `mutation AddReactionToIssue {`

  这里我们执行一个mutation，并将它命名为 `AddReactionToIssue`。和query一样，为mutation起名也是非必需的。

- `addReaction(input:{subjectId:"MDU6SXNzdWUyMzEzOTE1NTE=",content:HOORAY}) {`

  让我们来看这一行

  - `addReaction` 是mutation的名称。
  - `input` 是所需的argument键. 对于mutation来说这个键总是 `input` 。
  - `{subjectId:"MDU6SXNzdWUyMzEzOTE1NTE=",content:HOORAY}` 是所需的argument值。这个值总是由input field（此例中为`subjectId` 和 `content` ）组成的input object（所以要加大括号）。

  我们怎么知道content中用哪个值？ `addReaction` 文档告诉我们`content`field类型为`ReactionContent`，它是一个枚举类型，因为GitHub的issue只支持部分emoji表情。文档列出了允许的值（注意部分值与对应的emoji名称不同）：

  - `THUMBS_UP` 
  - `THUMBS_DOWN` 
  - `LAUGH` 
  - `HOORAY` 
  - `CONFUSED` 
  - `HEART` 

- 请求中剩下的部分由payload对象组成。在这里我们指明当执行mutation之后我们希望服务器返回的数据。这些行是从`addReaction`文档中获取的，可返回三个field：

  - `clientMutationId` (`String`)
  - `reaction` (`Reaction!`)
  - `subject` (`Reactable!`)

  在此示例中，我们返回两个必须的field（`reaction`和`subject`），他们都有必需的内部field（分别是`content`和`id`）。

当我们执行mutation，返回结果如下：

```
{
  "data": {
    "addReaction": {
      "reaction": {
        "content": "HOORAY"
      },
      "subject": {
        "id": "MDU6SXNzdWUyMTc5NTQ0OTc="
      }
    }
  }
}
```

最后还有一点：当你在input object中传递mutation field时，语法可能会很呆板。将field放到variable中可以改善这一点。以下是如何用variable重写原来的mutation：

```
mutation($myVar:AddReactionInput!) {
  addReaction(input:$myVar) {
    reaction {
      content
    }
    subject {
      id
    }
  }
}
variables {
  "myVar": {
    "subjectId":"MDU6SXNzdWUyMTc5NTQ0OTc=",
    "content":"HOORAY"
  }
}
```

> 你可能注意到了在之前的例子中的`content`（被直接在mutation中使用）没有在`HOORAY`外面加引号，但在variable中使用时则有引号。原因如下：
>
> - 当在mutation中直接使用`content`时，schema希望的值类型是`ReactionContent`，它是枚举类型而不是字符串。如果你加了引号在枚举类型外面的话，schema会抛出错误，因为引号代表了字符串。
> - 当在variable中直接使用`content`时，variables内容必须是有效的JSON，所以引号是必须的。当在执行过程中variable被传入mutation时，schema校验可以正确地解释为 `ReactionContent` 类型。

# 文档指引

在GitHub开发者网站中，有网页版的[接口文档](https://developer.github.com/v4/reference/)以供开发者查询，但我不推荐大家通过这种方式查询接口。

传统的REST API作为服务器资源或数据库数据的映射，其结构可理解为一种列表的形式，故通过文档目录可方便的进行查询。GraphQL API从名字就可以看出其内部采用的是一种类似“图”的数据结构，以反映纷繁复杂的节点之间的关系。故很难通过从头至尾阅读文档的方式全面的了解接口。

GitHub为开发者提供了一个名为[Explorer](https://developer.github.com/v4/explorer/)的工具：

![1](1.jpg)

通过开发者自己的GitHub账号授权，你可以使用这个工具方便的测试GitHub GraphQL API的各种请求。同时右侧的边栏可以让你方便的搜索或链接到所需的API文档。通过这种实验和文档相结合的方式，你就可以按照GraphQL的思维方式查看各个对象的连接关系和查询要求了。