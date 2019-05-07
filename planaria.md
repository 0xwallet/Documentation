# Planaria

planaria 是 unwriter 编写的一个与 bsv 区块链进行交互的工具. https://github.com/interplanaria 上提供了 planaria 的 dockerfile, 使用 docker 可以轻松部署 planaria 到我们的主机上.

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
3.