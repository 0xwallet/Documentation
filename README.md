# elixir 开发准备工作

<a name="c9ba11ec"></a>
# 需要安装以下程序:

<a name="e8a50bf5"></a>
# postgresql (9.0版本以上)

```bash
apt-get install postgresql-10

# 新建用户名/密码
sudo -u postgres psql
postgres=# create user myuser with encrypted password 'mypass';
```


<br />
<a name="asdf"></a>
# asdf
```bash
git clone https://github.com/asdf-vm/asdf.git ~/.asdf --branch v0.7.1

echo -e '\n. $HOME/.asdf/asdf.sh' >> ~/.bashrc
echo -e '\n. $HOME/.asdf/completions/asdf.bash' >> ~/.bashrc
```
<a name="d41d8cd9"></a>
#
<a name="elixir"></a>
# elixir

erlang deps
```bash
apt-get -y install build-essential autoconf m4 libncurses5-dev libwxgtk3.0-dev libgl1-mesa-dev libglu1-mesa-dev libpng-dev libssh-dev unixodbc-dev xsltproc fop dirmngr gpg
```

erlang

```bash
asdf plugin-add erlang https://github.com/asdf-vm/asdf-erlang.git
```

elixir

```bash
asdf plugin-add elixir https://github.com/asdf-vm/asdf-elixir.git
```

```bash
asdf install erlang 21.3.3
asdf global erlang 21.3.3
asdf install elixir 1.8.1
asdf global elixir 1.8.1
```

phoenix

```bash
mix archive.install hex phx_new 1.4.3
```

node.js

```bash
asdf plugin-add nodejs https://github.com/asdf-vm/asdf-nodejs.git
bash ~/.asdf/plugins/nodejs/bin/import-release-team-keyring

asdf install nodejs 8.9.4
```

cnpm

```bash
npm install -g cnpm --registry=https://registry.npm.taobao.org
```

yarn(选装)

```
curl -sL https://deb.nodesource.com/setup_11.x | sudo -E bash -

curl -sL https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
     echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
     sudo apt-get update && sudo apt-get install yarn
```


<a name="67b31f12"></a>
# 以上程序安装完之后, 将项目的仓库 clone 到本地.(以 dashboard 为例)

在 config/dev.exs 文件中, 找到

```erlang
# Configure your database
config :demo, Demo.Repo,
  username: "postgres",  # 改为数据库设置的用户名
  password: "postgres",  # 改为数据库设置的密码
  database: "demo_dev",
  hostname: "localhost",
  pool_size: 10
```

命令行运行
```bash
cd dashboard

# 前端依赖安装
cd assets
cnpm install

cd .. # 回到主目录
# 数据库
mix ecto.create
mix ecto.migrate

# 后端依赖安装
mix deps.get

# 启动服务器
iex -S mix phx.server
```

