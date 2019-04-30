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

## other

进入iex: `docker run -it --rm elixir`

进入 bash: `docker run -it --rm elixir bash`

elixir 预编译包的 checksum: https://elixir-lang.org/elixir.csv 对于第三方提供的软件包, 都应该检查其 checksum, 确保安全.

# SSH-COPY-ID

`ssh-copy-id` 是 ssh 的配套工具之一, 它提供了 passwordless 的登陆体验.

ssh 使用的是 publickey-authentication 机制, ssh-copy-id 可以让 publickey 的配置更加简单.

首先使用 `ssh-keygen` 来生成一个 keypair. 一般会位于 `~/.ssh` 目录下.

使用 `ssh-copy-id -i ~/.ssh/mykey user@host` 我们生成的公钥会被发送到服务器上作为认证的公钥, 这一
步骤可能需要提供密码, 或者其它的秘钥来登录.

最后, 使用 `ssh -i ~/.ssh/mykey user@host` 就可以无密码登录远程服务器了.
