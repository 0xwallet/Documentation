# 介绍

什么是阿波罗客户端, 它能做什么?

阿波罗客户端是使用 GraphQL 来构造客户端应用的最佳方式. 这个客户端被设计成能够帮助你很快地构造一个从 GraphQL 获取数据的 UI, 并且可以用于任何的 JavaScript 前端. 这个客户端是:

- 可移植的: 你可以把它放到一个现有的 app 里面.
- 通用的: 阿波罗适用于任何编译配置, 任何 GraphQL 服务器 和 任何 GraphQL schema.
- 可观测和易懂的: 清晰地知道应用中正在发生什么.
- 为交互应用而生: 应用的用户做出修改然后立刻看到反应.
- 小巧灵活: 你不会得到你不需要的东西.
- 社区驱动: 阿波罗是社区驱动的, 并且适用于多种场景.

# 为什么选择 Apollo 客户端来管理数据?

数据管理不应该是一件难事! 如果你在思考如何简化对React应用中远程和本地数据的管理, 那么你就选对地方了. 通过我们的实际实例应用 Pupstagram, 你会学到 Apollo 的只能缓存和声明式的数据获取方法, 能够帮助你更快的而且写更少的代码. 让我们开始吧! 🚀

# 声明式数据获取

通过 Apollo 的声明式方式来获取数据, 所有的数据搜索逻辑, 载入过程和错误状态, 以及对 UI 的更新全都包含在一个单独的 Query 组件中. 这使得你的 Query 组件和表现层的组件结合起来非常容易! 让我们看看 React Apollo 中它是什么样子的:

```js
const Feed = () => (
  <Query query={GET_DOGS}>
    {({ loading, error, data }) => {
      if (error) return <Error />
      if (loading || !data) return <Fetching />;

      return <DogList dogs={data.dogs} />
    }}
  </Query>
)
```

这里我们使用了一个 Query 组件来从 GraphQL 服务器获取到一些 dogs, 并且显示爱一个列表里. Query 组件使用 React render prop API (包含一个函数作为子组件) 来绑定一个 query 到我们的组件, 并且基于 query 的返回值来渲染. 一旦我们的数据返回了, 我们的 `<DogList/>` 组件就会根据所需的数据来更新.

Apollo Client 会管理好请求的开始到结束, 包括载入进度和错误状态. 你不需要设置中间件或者是编写模板, 你也不需要担心转换和缓存回复. 你要做的只是描述你的组件所需要的数据, 然后让 Apollo Client 来做重活. 💪

当你切换到 Apollo Client 之后, 你会发现你可以删除很多关于数据管理的不必要的代码. 具体的数据取决于你的应用, 但有一些团队报告说是几千行的代码. 当你发现使用 Apollo 可以编写更少的代码时, 这不意味着你需要压缩新功能! 高级功能可以通过 Query 组件的 props 很容易地得到, 例如 优化 UI, 重试, 以及分页等.


# 零配置缓存

Apollo Client 和其它的数据管理解决方案最关键的不同是它的通用缓存. 启动 Apollo Client, 你就获得了一个不需要额外配置的智能缓存. 在 Pupstagram 示例应用的主页, 点击其中一个 dog 来查看详细页面. 然后返回主页. 你会注意到主页的图片立刻就载入好了, 多亏了 Apollo 缓存.

```js
import ApolloClient from 'apollo-boost';

// the Apollo cache is set up automatically
const client = new ApolloClient();
```

缓存不是简单的工作, 但我们已经花了两年时间专注于解决它. 由于你可以从多个路径访问同样的数据, 通用化是保证你的数据在多个组件间的一致性的关键. 让我们来看看实际例子:

```js
const GET_ALL_DOGS = gql`
  query {
    dogs {
      id
      breed
      displayImage
    }
  }
`;

const UPDATE_DISPLAY_IMAGE = gql`
  mutation UpdateDisplayImage($id: String!, $displayImage: String!) {
    updateDisplayImage(id: $id, displayImage: $displayImage) {
      id
      displayImage
    }
  }
`;
```

查询语句 `GET_ALL_DOGS` 获取到了一系列的 dogs 和它们的显示图片. 修改语句 `UPDATE_DISPLAY_IMAGE` 更新了一个 dog 的显示图片. 如果我们更新一个特定的 dog 的显示图片, 我们就需要dogs列表能够表现出这个更新. Apollo Client 将 GraphQL 结果的每个 object 分割成一个带有 __typename 和 id 属性的对象, 放在 Apollo 缓存里. 这就保证了从 mutation 返回的值, 会根据其带有的 id 自动地更新任何查询语句中带有相同 id 的对象. 这也保证了两个返回相同数据的查询语句总是同步的.

通常很复杂的流程, 通过 Apollo cache 变得十分简单. 让我们回到之前的 `GET_ALL_DOGS` 查询语句里面. 对于特定的 dog 的详细页面, 我们想要传送什么信息过去? 由于我们已经获取到了每个 dog 的信息, 所以我们不想重新从服务器获取同样信息. 通过缓存重定向, Apollo cache 会帮我们连接两个 query 里面重复的部分, 这样就不需要反复获取信息.

这里是我们的查询一个 dog 的 query:

```js
const GET_DOG = gql`
  query {
    dog(id: "abc") {
      id
      breed
      displayImage
    }
  }
`;
```

这里是我们的 cache redirect, 我们可以很容易的指定一个 map 在 apollo-boost client 的 `cacheRedirects` 属性里. cache redirect 会返回一个 key, 然后 query 就可以使用它来从缓存里找到数据.

```js
import ApolloClient from 'apollo-boost';

const client = new ApolloClient({
  cacheRedirects: {
    Query: {
      dog: (_, { id }, { getCacheKey }) => getCacheKey({ id, __typename: 'Dog' })
    }
  }
})
```

# 结合本地和远程的数据

数以千计的开发者告诉我们, Apollo Client 在管理远程数据方面非常优秀, 满足了大约 80% 的数据需求. 但是剩下的 20% 本地数据怎么办? 例如全局 flags 和设备 API 的返回值. 这就是为什么我们有了 apollo-link-state, 它能让你将 Apollo cache 最为唯一的真实数据来源.

在 Apollo Client 中管理你全部的数据, 让你能够在你所有的数据接口中利用 GraphQL 的优点. 同时让你能够从 Apollo DevTools 的 GraphiQL 中观察你本地和远程的 schemas.

```js
const GET_DOG = gql`
  query GetDogByBreed($breed: String!) {
    dog(breed: $breed) {
      images {
        url
        id
        isLiked @client
      }
    }
  }
`;
```

通过 `apollo-link-state` 你可以轻松地添加只在本地有效的项, 到你的远程数据存储里, 然后从你的组件里面查询它们. 在这个例子里, 我们在查询服务端数据的同时查询 client-only 的项 isLiked. 你的组件是由本地和远程数据组成的, 现在你的查询语句也是一样!

# 充满活力的生态

Apollo Client 很容易上手, 同时在你需要进行扩展的时候也很容易. 如果你需要的自定义功能不包含在 apollo-boost 中, 例如针对特定应用的中间件或者持久化缓存, 你可以通过 Apollo Link 来将你自己的 client 连接到 Apollo cache 并且加入你的网络栈.

如果你的公司正在生产环境中使用 Apollo Client, 我们很乐意在我们的博客中添加你的案例! 请使用 Spectrum 联系我们, 这样我们就能了解你是如何使用 Apollo 的. 另外, 如果你已经有了一个关于此的博文或者大会演讲, 请发送 PR 给我们.

[开始](./apollo/getting_start.md)