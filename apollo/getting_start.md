# 在 React 项目中快速设置 Apollo Client

最简单的方式是使用 Apollo Boost, 其中包含了推荐的配置. Apollo Boost 包含了我们认为的构建 Apollo 应用所需的基本的库, 包括内存管理, 本地状态管理和错误处理. 同时也足够灵活, 能够处理类似身份认证之类的功能.

如果你是一个想要从零开始配置 Apollo Client 的高级用户, 请移步我们的 Apollo Boost 迁移手册. 对于大部分用户, Apollo Boost 应该能够满足你的需求, 所以, 在你真的需要更多的定制化再去看.

# 安装

首先, 安装一些包!

```
npm install apollo-boost react-apollo graphql
```

- apollo-boost: 包含了 Apollo Client 的所有需求
- react-apollo: 将 Apollo 和 React 的 View 层结合
- graphql: 处理 GraphQL 查询语句

> 我们还可以使用 [CodeSandbox](https://codesandbox.io/) 来进行开发.

# 创建一个客户端

很好, 现在我们所需的所有依赖已经安装好了, 让我们来创建 Apollo Client. 唯一要做的就是启动你的 GraphQL 服务器. 一般是在你的应用的 host 下的 `/graphql` 路径.

在 index.js 文件中 import ApolloClient 从 apollo-boost, 并在 client config 对象中添加 GraphQL server 的 uri 配置.

```js
import ApolloClient from "apollo-boost";

const client = new ApolloClient({
  uri: "https://48p1r2roz4.sse.codesandbox.io"
});
```

这样就行了! 现在你的 client 已经准备好开始获取数据了. 在我们将 Apollo Client 连接到 React 之前, 先试着使用普通的 JavaScript 发送一个 Query. 在 index.js 文件中, 试着调用 `client.query()`. 记住先 import `gql` 函数将你的 query 字符串解析成 query document.

```js
import { gql } from "apollo-boost";
// or you can use `import gql from 'graphql-tag';` instead

...

client
  .query({
    query: gql`
      {
        rates(currency: "USD") {
          currency
        }
      }
    `
  })
  .then(result => console.log(result));
```

打开控制台观察结果. 你会看到它包含了各种属性, 例如 rates, loading 以及 netwrokStatus. 当你不需要 React 或者其它的前端框架时, 我们的 view 层也可以让你更容易地绑定 query 和 UI, 并且反应式地更新你的组件数据.让我们学习如何将 Apollo Client 和 React 连接起来, 这样我们就可以开始用 react-apollo 构造查询组件了.

# 将客户端连接到 React

要将 Apollo Client 连接到 React, 你会需要使用 react-apollo 中的 ApolloProvider 组件. ApolloProvider 类似于 React 的 context provider. 它会包装你的 React 应用, 并且将客户端放在 context 中, 这让你可以从组件树的任意位置获取它.

在 `index.js` 中, 用 `ApolloProvider` 把 React 应用包裹起来. 我们建议把 `ApolloProvider` 放在比较高的位置, 在你需要获取 GraphQL 数据的上方. 例如, 它可以放在的 root route 组件的外面, 如果你正在使用 React Router 的话.

```js
import React from "react";
import { render } from "react-dom";

import { ApolloProvider } from "react-apollo";

const App = () => (
  <ApolloProvider client={client}>
    <div>
      <h2>My first Apollo app 🚀</h2>
    </div>
  </ApolloProvider>
);

render(<App />, document.getElementById("root"));
```

# 请求数据

一旦你的 ApolloProvider 放置好了, 你就可以使用 Query 组件来获取数据了! Query 是一个从 react-apollo 中导出的 React 组件, 它使用 render prop 模式来与你的 UI 共享 GraphQL 数据.

首先, 将你包装在 gql 函数里的 GraphQL query 传递给 Query 组件的 query prop. 然后, 你要提供一个函数给 Query 组件的子 prop 来确定有什么是要渲染的, 这个 Query 会包含对象的载入, 错误和数据等属性. Apollo Client 会为你跟踪错误和载入情况, 这将会被反映在 loading 和 error 属性上. 一旦你的查询结果反悔了, 它将会被附加到 data 属性上.

让我们在 `index.js` 文件中创建一个 `ExchangeRates` 组件, 来看看 Query 组件是如何运作的!

```js
import { Query } from "react-apollo";
import { gql } from "apollo-boost";

const ExchangeRates = () => (
  <Query
    query={gql`
      {
        rates(currency: "USD") {
          currency
          rate
        }
      }
    `}
  >
    {({ loading, error, data }) => {
      if (loading) return <p>Loading...</p>;
      if (error) return <p>Error :(</p>;

      return data.rates.map(({ currency, rate }) => (
        <div key={currency}>
          <p>{currency}: {rate}</p>
        </div>
      ));
    }}
  </Query>
);
```

恭喜, 你刚才构建了你的第一个 Query 组件! 如果你在你的 App 组件里面按照前面的例子渲染了 ExchangeRates 组件, 那么你会先看到一个载入提示然后是数据. Apollo Client 会自动缓存数据, 当数据从 server 发来的时候. 所以第二次运行相同 query 的时候你不会再看到 loading 提示.

# Apollo Boost

在我们的例子中, 我们使用 Apollo Boost 来快速配置 Apollo Client. 尽管 GraphQL 服务器是你开始时唯一需要做的配置, 这里还有很多其它功能的配置, 例如本地状态管理, 身份验证和错误处理.

## 它包括哪些?

Apollo Boost 包含了一些我们认为是很基础的包.这里有:

- apollo-client
- apollo-cache-inmemory
- apollo-link-http
- apollo-link-error
- apollo-link-state

Apollo Boost 最棒的地方在于你不需要自己进行全部配置! 只要指定一些选项, 它会自动帮你做好其它的.

## 配置选项

这里有一些配置选项, 你可以传送给 apollo-boost 导出的 ApolloClient. 它们都是可选的.

- `uri`: string
一个用来表示你的 GraphQL 服务器 endpoint 的字符串. 默认是 `/graphql`

- `fetchOptions`: Object
任何你想要传送给 fetch 的设置(证书, headers 等等). 这些配置是静态的, 所以它们不会因为每个 request 而变化.

- `request`: (operation: Operation) => Promise <void>
这个函数在每次请求的时候都会被调用. 它接受一个 GraphQL 操作并返回一个 promise. 要动态设置 `fetchOptions`, 你可以使用 `operation.setContext({ headers })` 将它们添加到操作的上下文中. 任何在这里设置的配置会传送给 `fetchOptions`. 对于身份验证很有帮助.

- `onError`: (errorObj: { graphQLErrors: GraphQLError[], networkError: Error, response?: ExecutionResult, operation: Operation }) => void
我们包括一个默认的错误处理器, 会将你的错误发送到 console. 如果你想用别的机制处理错误, 那么设定这个函数.

- `clientState`: { resolvers?: Object, defaults?: Object, typeDefs?: string | Array <string> }
一个用来表示你的 `apollo-link-state` 配置的对象. 对于本地状态管理很有用.
