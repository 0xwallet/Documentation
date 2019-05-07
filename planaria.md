# Planaria

planaria 是 unwriter 编写的一个与 bsv 区块链进行交互的工具.

因为它是由 BSV 驱动的，所以它具有我们在传统的集中式云服务中从未见过的各种独特特征：

- 透明：因为所有数据都是以透明方式从 BSV 派生的，所以您可以确保生成的机器按设计运行。此外，任何人都可以通过 Pull 和运行节点来验证结果，只需几个简单的命令。
- 便携：所有planaria机器都是 docker 化的，可以作为单个便携式Node.js模块发布，这意味着其他任何人都可以获取已发布的代码并运行与您完全相同的状态机。
- 可共享：一个组织垄断整个后端没有任何价值，因为所有内容都源自透明存储在比特币上的公共数据和公共算法。后端可以并且将比原始创建者存在更久。
- 可定制：您可以自定义 Planaria 的数量, 没有限制。您甚至可以将它与外部事件生成器和API混合，以构建由比特币驱动的新型后端。
- 用户友好：不像大多数难懂技术让你通过繁杂的环节来实现“去中心化”，Planaria让你与比特币交互，就像它只是一个带有HTTP API的常规CRUD数据库一样。

它还附带了一个名为 Planaria Computer 的命令行界面工具，能让你通过几个命令来启动和生成这些后端，这使得使用起来更加容易。

## 使用

可以通过 `pc` planaria computer, 一个命令行的远程工具, 来连接到远程的 planaria 节点.

1. 安装 pc:  `npm install -g planaria`
2. 生成新用户(秘钥对): `pc new user`
3. 连接到远程 planaria 节点

## 部署

运行 planaria 节点需要至少 2GB 内存和 200GB 硬盘.

首先, 需要运行 BitcoinSV 节点.

https://github.com/interplanaria 上提供了 planaria 的 dockerfile, 使用 docker 可以轻松部署 planaria 到我们的主机上.

planaria 的 dockerfile 内容如下:

```dockerfile
FROM mongo:latest
RUN apt-get update && \
    apt-get install -y wget unzip htop && \
    apt-get install -y sudo && \
    apt-get install -y curl && \
    apt-get install -y netcat && \
    curl -sL https://deb.nodesource.com/setup_10.x | bash - && \
    apt-get install -y nodejs \
    libtool pkg-config build-essential autoconf automake uuid-dev
RUN wget -q https://github.com/zeromq/libzmq/releases/download/v4.2.2/zeromq-4.2.2.tar.gz
RUN tar -xzvf zeromq-4.2.2.tar.gz
WORKDIR /zeromq-4.2.2
RUN ./configure
RUN make install & ldconfig

COPY . /planaria
RUN cd /planaria && rm -rf node_modules && npm install

VOLUME /data/db
VOLUME /planaria/.state

WORKDIR /planaria

EXPOSE 28332
EXPOSE 28339
EXPOSE 28337

ENTRYPOINT ["/planaria/entrypoint.sh"]
```

可以看出其中包含了 MongoDB, ZeroMQ 等工具. 并且开放了 28332, 28339, 28337 端口.

部署流程如下:

1. 在 DaoCloud 上创建新项目, 命名为 planaria, github 仓库地址填写 https://github.com/interplanaria/planaria.git, 手动触发构建任务.
2. 构建成功后, 切换到"应用"界面, 选择planaria 右边的 "部署最新版本"
3. 部署成功后, 需要设置启动命令.

查看 planaria 内置的 entrypoint 脚本:

```sh
#!/bin/bash

set -e
if [[ "$1" == "start" ]]; then

  echo "# Starting Mongodb......."
  echo "arg0: $2"
  echo "arg1: $3"
  echo "arg2: $4"
  echo "arg3: $5"
  mongod --bind_ip_all --wiredTigerCacheSizeGB=$2 &
  until nc -z localhost 27017
    do
      sleep 1
    done

  echo "# Inheriting package.json...."
  node /planaria/merge /planaria
  echo "# rm -rf node_modules..."
  rm -rf ./node_modules
  echo "# npm install..."
  npm install

  echo "# Starting Planaria......."
  node --max-old-space-size=4096 /planaria/index $3 $4 $5
fi

exec "$@"
```

可以看出它需要读取 5 个参数.