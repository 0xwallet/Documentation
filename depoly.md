# 部署流程

1. 在本地安装 Docker

docker 下载地址: https://hub.docker.com/editions/community/docker-ce-desktop-mac

2. 构建 image

3. 将 image push 到云端

4. 在云端启动项目

5. 配置项目的各种环境变量, 证书等.

# Daocloud 部署

daocloud 支持自动流水线部署.

1. 新建项目, 选择使用 git 地址.

2. 按 http://guide.daocloud.io/dcs/git-9870399.html 所示, 复制 depoly key 到项目的 github 上.

# Docker elixir

elixir 官方的 docker image: https://hub.docker.com/_/elixir/


## Dockerfile
在我们的 elixir 项目的根目录创建 `Dockerfile`:

```dockerfile
# ./Dockerfile

# Extend from the official Elixir image
FROM elixir:latest

# Create app directory and copy the Elixir projects into it
RUN mkdir /app
COPY . /app
WORKDIR /app

# Install hex package manager
# By using --force, we don’t need to type “Y” to confirm the installation
RUN mix local.hex --force

# Compile the project
RUN mix do compile
```

## Docker-compose

我们使用 docker-compose 文件来管理集群里的各个服务:

```yaml
# Version of docker-compose
version: '3'

# Containers we are going to run
services:
  # Our Phoenix container
  phoenix:
    # The build parameters for this container.
    build:
      # Here we define that it should build from the current directory
      context: .
    environment:
      # Variables to connect to our Postgres server
      PGUSER: postgres
      PGPASSWORD: postgres
      PGDATABASE: database_name
      PGPORT: 5432
      # Hostname of our Postgres container
      PGHOST: db
    ports:
      # Mapping the port to make the Phoenix app accessible outside of the container
      - "4000:4000"
    depends_on:
      # The db container needs to be started before we start this container
      - db
  db:
    # We use the predefined Postgres image
    image: postgres:9.6
    environment:
      # Set user/password for Postgres
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      # Set a path where Postgres should store the data
      PGDATA: /var/lib/postgresql/data/pgdata
    restart: always
    volumes:
      - pgdata:/var/lib/postgresql/data
# Define the volumes
volumes:
  pgdata:
```

运行 `docker-compose build` 即可编译所有的 images.

运行 `docker images` 查看 images:
```bash
REPOSITORY           TAG                 IMAGE ID            CREATED             SIZE
helloworld_phoenix   latest              be0fed013a6b        2 minutes ago       917MB
elixir               latest              9d6cef9afe13        13 days ago         889MB
postgres             9.6                 d3ac03f9698d        4 weeks ago         268MB
```

运行 `docker-compose up` 即可启动.

但之前, 还需要一个 `entrypoint.sh` 来作为phoenix 应用的启动脚本, 执行创建数据库, 数据库 migration 等操作:
```bash
# entrypoint.sh

#!/bin/bash
# Docker entrypoint script.

# Wait until Postgres is ready
while ! pg_isready -q -h $PGHOST -p $PGPORT -U $PGUSER
do
  echo "$(date) - waiting for database to start"
  sleep 2
done

# Create, migrate, and seed database if it doesn't exist.
if [[ -z `psql -Atqc "\\list $PGDATABASE"` ]]; then
  echo "Database $PGDATABASE does not exist. Creating..."
  createdb -E UTF8 $PGDATABASE -l en_US.UTF-8 -T template0
  mix ecto.migrate
  mix run priv/repo/seeds.exs
  echo "Database $PGDATABASE created."
fi

exec mix phx.server
```

所以, 我们项目的 dockerfile 应该是这样:
```dockerfile
# Use an official Elixir runtime as a parent image
FROM elixir:latest

RUN apt-get update && \
  apt-get install -y postgresql-client

# Create app directory and copy the Elixir projects into it
RUN mkdir /app
COPY . /app
WORKDIR /app

# Install hex package manager
RUN mix local.hex --force

# Compile the project
RUN mix do compile

CMD ["/app/entrypoint.sh"]
```

在项目中, 我们使用环境变量来传递敏感配置:
```ex
defmodule YourApp.Repo do
  use Ecto.Repo, otp_app: :your_app

  def init(_, config) do
    config = config
      |> Keyword.put(:username, System.get_env("PGUSER"))
      |> Keyword.put(:password, System.get_env("PGPASSWORD"))
      |> Keyword.put(:database, System.get_env("PGDATABASE"))
      |> Keyword.put(:hostname, System.get_env("PGHOST"))
      |> Keyword.put(:port, System.get_env("PGPORT") |> String.to_integer)
    {:ok, config}
  end
end
```

完成了. 可以运行 `docker-compose build` 和 `docker-compose up` 来启动容器了.

## 如何在开发时配置环境变量

在本地开发的时候, 我们也可以沿用环境变量传递参数的模式, 以保证代码的一致. 这时候就需要用到
direnv 这个管理环境变量的工具.

在 https://github.com/direnv/direnv 下载安装 direnv. 安装好后, 如果你用的是 bash, 就在 `~/.bashrc` 文件中
加入这一行: `eval "$(direnv hook bash)"` . 运行 `source ~/.bashrc` 载入配置.

在本项目的根目录下, 新建文件 `.envrc`, 在里面写入所需的环境变量, 例如:

```bash
export FOO=123
```

然后运行 `direnv allow .` 即可载入本项目专属的环境变量.

## HTTPS

CERTBOT 的 docker image https://hub.docker.com/r/certbot/certbot/dockerfile

Certbot 的更新很频繁. 如果你将 Certbot 安装在你的服务器上, 每次更新的时候都需要卸载并安装新的版本, 所以, 使用 docker 容器是更好的选择. 当 Certbot 更新时, 新的 image 会自动从 Docker 注册点被 pull 下来.

**使用了 docker 化地 Certbot, 获取 Let's Encrypt 证书只需要以下两步:**

1. 获取证书, 只需要简单地执行一个 Docker run 脚本. 这个脚本看起来很像是你在服务器上安装 Certbot 时会用到的. Docker 会在一个容器里面执行这个脚本, 当脚本完成时, 容器会被关闭.

2. 设置一个 cron 任务, 定期地执行证书更新脚本.

在我们可以执行 Certbot 命令来获取新的证书之前, 我们需要运行一个非常基础的 Nginx 实例, 让域名可以被通过 HTTP 访问到.

Let's Encrypt 在颁发证书之前, 会发起一个 ACME 挑战请求:

- 我们执行一个 Certbot agnet 的命令
- Certbot 告知 Let's Encrypt 我们需要一个 SSL/TLS 证书
- Let's Encrypt 向 Certbot agent 发送一个独特的 token
- Certbot agnet 将这个 token 放在我们的域名的 endpoint 上, 就像 `http://ohhaithere.com/.well-known/acme-challenge/{token}`
- 如果这个 token 是正确的, 那么挑战成功, Let's Encrypt 就知道这个域名是我们控制的.

这个基础的 Nignix 实例只会在第一次请求证书的时候被运行. 这个基本的实例甚至不需要有一个默认页面. 只需要给 Certbot agent 写的权限, 这样就可以将 token 放在 endpoint 上.

我们不能配置一个单独的 Nginx 实例, 因为在没有证书之前, Nginx 只能配置 HTTP. 一旦我们有了证书, 就可以配置 SSL/TLS. 如果60 或者 90 天后, 需要 renew 证书, 之后的 challenge 将会发送到我们生产版本的 Nginx 上, 所以我们不需要再次运行基本的 Nginx 实例.

回顾一下，Let's encrypt的证书的第一个请求将涉及以下内容：

- 配置仅在HTTP上运行的Nginx的基本版本，并为以下端点提供Certbot代理写入访问权限：`http://ohhaithere.com/.well-known/acme-challenge/ {token}`
- 通过Docker Compose调整Nginx的基本容器
- 执行一个Docker run命令，该命令将启动Certbot代理程序。 Certbot代理将执行质询请求，如果成功，请将SSL证书放在服务器上的Let's Encrypt文件夹中。
- Certbot代理程序完成后，容器将自动停止
- 发出一个Docker Compose down命令，它将停止和关闭你的基本版本的Nginx容器

## Useful Commands

Here is a small list of useful commands when dealing with Docker:

`docker ps` lists all containers that are currently running
`docker container ls --all` lists all containers that are available
`docker logs <container>` shows the log of the given container
`docker start/stop <container>` starts or stops a container
`docker rm <container>` removes a container
`docker images` lists all available images
`docker rmi <image>` removes an image
`docker-compose down --volumes` destroys the created volumes

# multi-stage build

使用 multi-stage build 可以有效地减小最终产生的 images 的体积. 在 Dockerfile 里, 每条指令都会产生一个新的 layer, 而你需要在进入下一个 layer 之前清理掉所有垃圾. 想编写一个非常高效的 Dockerfile, 你需要很多的 shell 操作技巧.

开发环境和生产环境的 Dockerfile 很不一样, 开发环境里会包含编译所需要的所有东西, 而生产环境只需要运行你的 app. 然而, 维护两个 Dockerfile 并不是理想做法.

## 命令

进入iex: `docker run -it --rm elixir`

进入 bash: `docker run -it --rm elixir bash`

elixir 预编译包的 checksum: https://elixir-lang.org/elixir.csv 对于第三方提供的软件包, 都应该检查其 checksum, 确保安全.

## Trouble Shooting

- elixir 报错无法启动 apps.

可能是由于在 dockerfile 中设置了非 root 用户, 导致权限不够, 无法写入某些文件. 在 Dockerfile 里设置 `User root` 即可.

- 连接不上数据库.

可能是数据库的 hostname, password 或者 port 配置错误.

- Daocloud 流水线在发布环节失败

删除应用, 重新部署.

## POSTGRES docker

配置端口映射:

    5432:5432

配置环境变量:

      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      PGDATA: /var/lib/postgresql/data/pgdata

配置 VOLUMES:

      pgdata:/var/lib/postgresql/data

检查是否启动成功:

        `pg_isready -h <IP ADDRESS>`


## 如何传递配置信息到 docker 中的 phoenix 项目里

一般我们可以通过环境变量来传递参数, 在 DaoCloud 上设置环境变量:

```
- PGUSER:
- PGPASSWORD:
- PGDATABASE:
- PGHOST:
- PGPORT:
```

然后在我们的 phoenix 应用启动时读取环境变量:

```ex
defmodule Demo.Repo do
  use Ecto.Repo, otp_app: :demo

  def init(_, config) do
    config = config
      |> Keyword.put(:username, System.get_env("PGUSER"))
      |> Keyword.put(:password, System.get_env("PGPASSWORD"))
      |> Keyword.put(:database, System.get_env("PGDATABASE"))
      |> Keyword.put(:hostname, System.get_env("PGHOST"))
      |> Keyword.put(:port, System.get_env("PGPORT") |> String.to_integer)
    {:ok, config}
  end
end
```


# SSH-COPY-ID

`ssh-copy-id` 是 ssh 的配套工具之一, 它提供了 passwordless 的登陆体验.

ssh 使用的是 publickey-authentication 机制, ssh-copy-id 可以让 publickey 的配置更加简单.

首先使用 `ssh-keygen` 来生成一个 keypair. 一般会位于 `~/.ssh` 目录下.

使用 `ssh-copy-id -i ~/.ssh/mykey user@host` 我们生成的公钥会被发送到服务器上作为认证的公钥, 这一
步骤可能需要提供密码, 或者其它的秘钥来登录.

最后, 使用 `ssh -i ~/.ssh/mykey user@host` 就可以无密码登录远程服务器了.
