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
