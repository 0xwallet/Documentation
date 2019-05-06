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

对于所有Let's Encrypt renew 请求，该过程将涉及以下内容：

确保已配置并启动并运行Nginx的生产版本。只要您的站点已准备好部署，该容器将通过Docker Compose启动，并将保持正常运行。
配置将执行Docker运行命令中的 cron 任务，该命令每周或每两周执行一次 Certbot renew。

下面的 docker-compose 文件的作用就是运行一个基本版本的 nginx 来获取整数:

```yaml
version: '3.1'

services:

  letsencrypt-nginx-container:
    container_name: 'letsencrypt-nginx-container'
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
      - ./letsencrypt-site:/usr/share/nginx/html
    networks:
      - docker-network

networks:
  docker-network:
    driver: bridge
```

它执行了以下操作:

1. 将最新版本的 Nginx 从 Docker 注册点拉下来
2. 将容器内的 80 端口对应到 host 的 80 端口, 这意味着向你的域名发来的请求会转发到 docker 容器内的 nginx 上.
3. 将 nginx 的配置文件映射到我们下一步将要创建的配置文件的位置. 当容器启动时, 它会载入我们自定义的配置文件.
4. 将 `/docker/letsencrypt-docker-nginx/src/letsencrypt/letsencrypt-site` 关联到容器内 Nginx 的默认位置. 在这个实例里, 其实不是很必要, 因为这个站点的唯一作用就是响应挑战请求, 但是放置一个默认 HTML 文件是一个好习惯.
5. 创建默认的 docker 网络.

然后是配置一下 nginx: `/docker/letsencrypt-docker-nginx/src/letsencrypt/nginx.conf`
```
server {
    listen 80;
    listen [::]:80;
    server_name ohhaithere.com www.ohhaithere.com;

    location ~ /.well-known/acme-challenge {
        allow all;
        root /usr/share/nginx/html;
    }

    root /usr/share/nginx/html;
    index index.html;
}
```

Nginx 配置文件做了以下工作:

1. 监听 80 端口, 为 URL ohhaithere.com 和 www.ohhaithere.com
2. 给 Certbot agent 访问 `./well-known/acme-challenge` 的权限
3. 设定默认的 root 和文件

然后创建 html 文件: `/docker/letsencrypt-docker-nginx/src/letsencrypt/letsencrypt-site/index.html`

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <title>Let's Encrypt First Time Cert Issue Site</title>
</head>
<body>
    <h1>Oh, hai there!</h1>
    <p>
        This is the temporary site that will only be used for the very first time SSL certificates are issued by Let's Encrypt's
        certbot.
    </p>
</body>
</html>
```

在运行 certbot 之前, 确保这个临时的 nginx 启动了:

```bash
cd /docker/letsencrypt-docker-nginx/src/letsencrypt
sudo docker-compose up -d
```

接下来运行以下命令来申请最新的证书:

```bash
sudo docker run -it --rm \
-v /docker-volumes/etc/letsencrypt:/etc/letsencrypt \
-v /docker-volumes/var/lib/letsencrypt:/var/lib/letsencrypt \
-v /docker/letsencrypt-docker-nginx/src/letsencrypt/letsencrypt-site:/data/letsencrypt \
-v "/docker-volumes/var/log/letsencrypt:/var/log/letsencrypt" \
certbot/certbot \
certonly --webroot \
--register-unsafely-without-email --agree-tos \
--webroot-path=/data/letsencrypt \
--staging \
-d ohhaithere.com -d www.ohhaithere.com
```

## Certbot docker image 的使用方法

在 DaoCloud 上配置应用的 yaml 文件, 如下:
```yaml
certbot:
  image: index.docker.io/certbot/certbot:v0.33.1
  command: certonly --webroot --register-unsafely-without-email --agree-tos --webroot-path=/data/letsencrypt --staging -d owaf.io -d www.owaf.io -n
  privileged: false
  restart: always
  ports:
    - '80'
    - '443'
  volumes:
    - /docker-volumes/etc/letsencrypt:/etc/letsencrypt
    - /docker-volumes/var/lib/letsencrypt:/var/lib/letsencrypt
    - /docker/letsencrypt-docker-nginx/src/letsencrypt/letsencrypt-site:/data/letsencrypt
    - /docker-volumes/var/log/letsencrypt:/var/log/letsencrypt
```

`--staging` 是测试获取证书的流程. `-n` 表示跳过交互操作.

上面一步执行完后, 运行启动命令: `--staging certificates` 来查看证书.

如果一切正常, 将看到:
```bash
2019-05-03 20:54:51:Found the following certs:
2019-05-03 20:54:51:  Certificate Name: owaf.io
2019-05-03 20:54:51:    Domains: owaf.io www.owaf.io
2019-05-03 20:54:51:    Expiry Date: 2019-08-01 10:48:42+00:00 (INVALID: TEST_CERT)
2019-05-03 20:54:51:    Certificate Path: /etc/letsencrypt/live/owaf.io/fullchain.pem
2019-05-03 20:54:51:    Private Key Path: /etc/letsencrypt/live/owaf.io/privkey.pem
2019-05-03 20:54:51:- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
2019-05-03 20:54:53:  Certificate Name: owaf.io
2019-05-03 20:54:53:    Domains: owaf.io www.owaf.io
2019-05-03 20:54:53:    Expiry Date: 2019-08-01 10:48:42+00:00 (INVALID: TEST_CERT)
2019-05-03 20:54:53:    Certificate Path: /etc/letsencrypt/live/owaf.io/fullchain.pem
2019-05-03 20:54:53:    Private Key Path: /etc/letsencrypt/live/owaf.io/privkey.pem
```

在主机上执行`sudo rm -rf /docker-volumes/` , 删除测试证书.

接下来获取正式证书:

```yaml
certbot:
  image: index.docker.io/certbot/certbot:v0.33.1
  command: certonly --webroot --email bitcoinsv@yahoo.com --no-eff-email --agree-tos --webroot-path=/data/letsencrypt -d owaf.io -d www.owaf.io --keep-until-expiring
  privileged: false
  restart: always
  ports:
    - '80'
    - '443'
  volumes:
    - /docker-volumes/etc/letsencrypt:/etc/letsencrypt
    - /docker-volumes/var/lib/letsencrypt:/var/lib/letsencrypt
    - /docker/letsencrypt-docker-nginx/src/letsencrypt/letsencrypt-site:/data/letsencrypt
    - /docker-volumes/var/log/letsencrypt:/var/log/letsencrypt
```

现在我们已经

接下来, 可以用 Haproxy 实现负载均衡以及 https:
 https://blog.georgejose.com/moving-my-http-website-to-https-using-letsencrypt-haproxy-and-docker-deb56ff6be9b


## HAProxy

HAProxy, 全称是 High Availability Proxy. 是一个非常流行的负载均衡和反向代理应用. 在我们的配置中, 我们将会用它作为一个代理层, 将所有通过 HTTPS 指向我们的服务器的请求转到对应的静态文件. 我们也会将其设置成把所有 HTTP 流量转移到 HTTPS.

首先在服务器上创建以下两个文件, 分别用来存放我们的网站, 和 haproxy:

```
|-- personal-website
|-- haproxy
```

如果你已经有了一个网站应用了, 可以将其启动在 8080(或者其它) 端口. 并且先获取一下 SSL 证书, 获取证书的方法见上文.

下面是 haproxy 文件的结构:

```
# HAProxy folder structure
HAProxy
|-- Dockerfile
|-- haproxy.cfg
|-- private/
  |-- domain.com.pem # place the ssl certificate you obtained here
```

将 Dockerfile 更新成下面这样:

```dockerfile
# Dockerfile

# Use the offical haproxy base image
FROM haproxy:1.7

# Copy our haproxy configuration into the docker container
COPY haproxy.cfg /usr/local/etc/haproxy/haproxy.cfg

# Copy our ssl certificate into the docker container
COPY private/domain.com.pem /private/domain.com.pem

# HAProxy requires a user & group named haproxy in order to run
RUN groupadd haproxy && useradd -g haproxy haproxy

# HAProxy also requires /var/lib/haproxy/run/haproxy/ to be created before it's run
RUN mkdir -p /var/lib/haproxy/run/haproxy/
```


dockercloud-haproxy 对 HAProxy 进行了包装，增加了在 docker 环境下自动配置的功能

部署 dockercloud-haproxy
按照如下 compose 配置

lb 服务监听机器的 80 端口，并把 docker.sock 映射到容器中，HAProxy 容器会检查 links 的服务的变化并更新负载均衡配置。

在 web 中使用 VIRTUAL_HOST 配置服务的访问地址。

```
version: '2'
services:
  web:
    image: daocloud.io/daocloud/dao-2048
    environment:
    - VIRTUAL_HOST=*owaf.io
  lb:
    image: daocloud.io/daocloud/dockercloud-haproxy
    links:
    - web
    volumes:
    - /var/run/docker.sock:/var/run/docker.sock
    ports:
    - 80:80
```

启动后自动完成配置，

可以在机器上执行命令 curl -H 'Host: hello.example' xxx.xxx.xxx.xxx 来测试配置，如返回 2048 的HTML内容则代表配置成功。

注意这里使用 *owaf.io 来表示 VIRTUAL_HOST, 可以匹配到 `owaf.io` 和 `www.owaf.io`.

在应用页面增加容器的数量，HAProxy 会自动把流量负载均衡到这些容器上。

负载均衡的架构如下所示：
```

                                                           |---- container_a1
                                    |----- service_a ----- |---- container_a2
                                    |   (virtual host a)   |---- container_a3
internet --- dockercloud/haproxy--- |
                                    |                      |---- container_b1
                                    |----- service_b ----- |---- container_b2
                                        (virtual host b)   |---- container_b3

```
这里只介绍了简单的使用方式

更高级的用法请查看镜像页面 HAProxy

环境变量配置:

- `CACERTFILE`
    the path of a ca-cert file. This allows you to mount your ca-cert file directly from a volume instead of from envvar. If set, CA_CERT envvar will be ignored. Possible value: `/cacerts/cert0.pem`

## HAProxy 配置

HAProxy 的配置流程包含主要 3 种参数来源:

- 来自命令行的参数, 优先级最高
- "global" 全局参数
- "defaults", "listen", "frontend", "backend"

配置文件的格式是 keyword 启头, 后面跟着几个用空格分隔的参数.

配置文件支持 3 种类型的 quote:

- 反斜杠
- 单引号弱引用
- 双引号强引用

反斜杠可以对以下字符反义:

- `\ ` 空格
- `\#` 井号
- `\\` 反斜杠
- `\'` 单引号
- `\"` 双引号

双引号可以对以下字符反义:

- " " 空格
- "'" 单引号
- "#" 井号

## HAProxy config example

`/etc/haproxy/haproxy.cfg`

```
frontend fe-owaf
    bind *:80

    # This is our new config that listens on port 443 for SSL connections
    bind *:443 ssl crt /etc/letsencrypt/live/owaf.io/fullchain.pem

    default_backend be-owaf

# Normal (default) Backend
# for web app servers
backend be-owaf
    # Config omitted here
    server owaf 161.117.83.227:80
```

## Certbot 获得的证书如何被 HAProxy 使用

HAProxy 需要一个单个文件的 SSL 证书, 并且是固定的格式. 所以, 我们需要将 certbot 的 live 文件夹下的证书文件, 转换成 HAProxy 可以使用的格式.

步骤如下:

```bash
sudo mkdir -p /etc/ssl/demo.scalinglaravel.com

sudo cat /etc/letsencrypt/live/demo.scalinglaravel.com/fullchain.pem \
    /etc/letsencrypt/live/demo.scalinglaravel.com/privkey.pem \
    | sudo tee /etc/ssl/demo.scalinglaravel.com/demo.scalinglaravel.com.pem
```

我们把 fullchain 和 privkey 两个文件结合在了一起, 顺序很关键.

## 如何使用环境变量写入配置文件

使用 DaoCloud 的 Stack 功能部署 HAProxy, 并且传递配置文件.

```yaml
version: "2"

services:
  haproxy:
    image: library/haproxy:1.8.20
    ports:
      - 80:80
      - 443:443
    volumes:
      - /docker-volumes/etc/letsencrypt:/etc/letsencrypt
    environment:
      IFS: ""
      HAPROXY_CONFIG: |
        frontend fe-owaf
            bind *:80

            # This is our new config that listens on port 443 for SSL connections
            bind *:443 ssl crt /etc/letsencrypt/live/owaf.io/owaf.io.pem

            default_backend be-owaf

        # Normal (default) Backend
        # for web app servers
        backend be-owaf
            # Config omitted here
            server owaf 161.117.83.227:80
    command:
      [sh, -c, "cat /etc/letsencrypt/live/owaf.io/fullchain.pem /etc/letsencrypt/live/owaf.io/privkey.pem | tee /etc/letsencrypt/live/owaf.io/owaf.io.pem && mkdir -p /usr/local/etc/haproxy/ && touch /usr/local/etc/haproxy/haproxy.cfg && echo $$HAPROXY_CONFIG > /usr/local/etc/haproxy/haproxy.cfg && cat /usr/local/etc/haproxy/haproxy.cfg && haproxy -f /usr/local/etc/haproxy/haproxy.cfg"]
```

如果文件不存在, 可以使用 `mkdir -p ~/unexist/yes/ && touch ~/unexist/yes/lol.txt && echo "hello" > ~/unexist/yes/lol.txt` 命令, 先创建文件.

## 自动更新证书

更新的流程如下:

1. 获得新的证书
2. 使用新证书制作 HAProxy 可以使用的格式的证书
3. 默认情况下, LetsEncrypt 创建了一个 CRON 任务在 `/etc/cron.d/certbot` 里. 这个任务一天跑两次(默认情况下, LetsEncrypt 只会在离过期 30 天以内更新证书.)

我们要做的是写一个每月运行一次的bash脚本, 并且每次强制更新证书.

我们将 CRON 任务修改成:

```
0 0 1 * * root bash /opt/update-certs.sh
```

这个任务会在每月的第一天的 0 点 0 分执行.

任务里执行的 bash 脚本(/opt/update-certs.sh) 是这样的:

```bash
#!/usr/bin/env bash

# Renew the certificate
certbot renew --force-renewal --tls-sni-01-port=8888

# Concatenate new cert files, with less output (avoiding the use tee and its output to stdout)
bash -c "cat /etc/letsencrypt/live/demo.scalinglaravel.com/fullchain.pem /etc/letsencrypt/live/demo.scalinglaravel.com/privkey.pem > /etc/ssl/demo.scalinglaravel.com/demo.scalinglaravel.com.pem"

# Reload  HAProxy
service haproxy reload
```

这和我们之前步骤唯一的区别是使用了 `--force-renewal` 来让 LetsEncrypt 每月更新一次证书. 这种方式比一天跑两次更容易理解, 也更不容易出错.



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

- shell 中写入文件时换行符 `\n` 丢失

将环境变量 IFS 设置为 "" (空的), 即可关闭自动剪裁.

## 集群与主机

集群，是 DaoCloud 平台上资源的结合。集群为用户提供了统一管理计算资源的一种方式。

集群是一个逻辑概念，您可以创建自有集群，并向集群中添加属于自己的主机。集群用来区分不同目的的资源和应用交付目的地，比如供团队内部测试和交付的部门测试集群、位于公司私有云或公有云之上的大规模应用预发布平台。

使用容器化软件交付，在完成镜像构建后，我们可以非常方便的把一个或者一组镜像部署到不同的集群之上，用于不同的交付目的。

在选择部署应用到集群时, 会随机选择集群内的一台主机部署.

自有主机上的容器端口，没有任何限制，用户可以暴露任意多端口，但需要注意的是，在应用多实例的情况下，用户需要自己完成负载均衡的配置，并确保同一主机之上无端口冲突。自有主机并不提供预置的数据库服务，当需要使用 MySQL 等数据服务时，可以选择 MySQL 进行进行部署，使用 Volume 保存数据库文件，并通过环境变量等方式与应用容器建立连接.

我们还可以修改启动命令，和调整容器实例个数。当容器实例多于一个时，则会在主机之上创建多个容器实例，直到主机资源耗尽为止。另外再次提醒，自有集群之上多实例应用，需要用户自己完成负载均衡设置。


## 部署复杂的多节点微服务应用

在集群中, 我们需要自己实现负载均衡, 也就是流量转发.

详见 [DaoCloud 的官方文档](http://guide-static.daocloud.io/dcs/%E9%83%A8%E7%BD%B2%E5%A4%8D%E6%9D%82%E7%9A%84%E5%A4%9A%E8%8A%82%E7%82%B9%E5%BE%AE%E6%9C%8D%E5%8A%A1%E5%BA%94%E7%94%A8-9153682.html).

注意, DaoCloud 不支持 docker-compose v3 的语法,

Docker Compose YML是由若干Service组成，每个Service都必须包括Image。其他的字段都是可选的，功能和docker run命令保持一致。

### DaoCloud 不支持的功能
Docker Compose运行在本地，而DaoCloud运行在云端。有少量的参数是暂时不支持的。

因此DaoCloud暂时不支持和构建相关的参数。也不支持和本地配置文件有关的参数。包括

- build
- env_file
- dockerfile
- extends

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

# asdf

asdf 是一个管理各种应用程序版本的工具, 它有很多强大的插件和功能:

- 从 master branch 安装最新的 elixir:

`asdf install elixir ref:master`