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

