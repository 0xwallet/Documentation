- [Planaria 介绍](./planaria.md)
- [Planaria 编写教程之 解读 genesis](./genesis.md)
- [前端文档](./coreui.md)
- [bitcoinsv-elixir文档](./bitcoinsv-elixir.md)
- [部署(Depoly)文档](./depoly.md)
- [phoenix框架文档](./phoenix.md)
- [METANET 时代的 Web 应用架构](./metanet.md)
- [Trouble Shooting](./trouble.md)

# 比特币转账手续费(矿工费)计算方式

比特币网络中是按照交易的体积来计算手续费的, 但是费率没有一个绝对的标准, 对于 bsv 来说, 一般是在
1 satoshi/byte 及以上. 由于 bsv 网络不存在像 btc, eth 那样的拥堵状况, 所以手续费不会突然飚高. 当
然用户也可以选择支付更高一点的手续费, 吸引矿工更快地来吧这笔交易打包进区块.

即使用户支付了最低的手续费, 只要他能够把这笔交易广播给足够多的矿工, 这笔交易就是安全的. 因为矿工是
遵循先见原则的.

由于采用交易体积来计算手续费, 而在签名完成之前, 我们是不知道交易体积究竟有多大的, 所以在构建交易的
时候, 我们需要先预测交易的体积会是多少. 这涉及到 inputs 和 outputs 的个数.

以下是一笔普通的 p2pkh 交易, 总体积是 226 bytes, 按照 1satoshi/tytes 的费率来算, 应付 226 satoshis.
而实际上 https://blockchair.com/bitcoin-sv/transaction/1f2de5350418ccca6234ccadf692035dbf057cf9d832d0e8f54fa2edbb6d00f6,
这笔交易支付了 5000 satoshis, 尽管还不到 1 美分, 但也是应付的十倍还要多.

```json
{
  "txid":"1f2de5350418ccca6234ccadf692035dbf057cf9d832d0e8f54fa2edbb6d00f6","hash":"1f2de5350418ccca6234ccadf692035dbf057cf9d832d0e8f54fa2edbb6d00f6","size":226,"version":1,"locktime":0,

  "vin":  [
    {"txid":"5ae75c2c7f19b42840f299ffdf167caec060defea0502ba8e1cf5bc03bbe8244","vout":0,"scriptSig":{"asm":"3045022100fcde460d1a3500f492bf1f17a248cdabf0713bc7ecb0d68b9f43b2ceb49a3d50022072d4bdb3d56d5863f18256d2af97a67db801cc087d47022ede8f48f595676981[ALL|FORKID] 02c214179fda82335a83b982383b093344c08d502ee472b349886be869e5080052","hex":"483045022100fcde460d1a3500f492bf1f17a248cdabf0713bc7ecb0d68b9f43b2ceb49a3d50022072d4bdb3d56d5863f18256d2af97a67db801cc087d47022ede8f48f595676981412102c214179fda82335a83b982383b093344c08d502ee472b349886be869e5080052"},"sequence":4294967295}
  ],

  "vout": [
    {"value":0.12588296,"n":0,"scriptPubKey":{"asm":"OP_DUP OP_HASH160 926f915bd7285586ae795ba40461d3d4ae537608 OP_EQUALVERIFY OP_CHECKSIG","hex":"76a914926f915bd7285586ae795ba40461d3d4ae53760888ac","reqSigs":1,"type":"pubkeyhash","addresses":["bitcoincash:qzfxly2m6u59tp4w09d6gprp6022u5mkpqkwa7997y"]}},

    {"value":0.66106704,"n":1,"scriptPubKey":{"asm":"OP_DUP OP_HASH160 45395b25d6339bbb8f3da68e278f21d54ba935bd OP_EQUALVERIFY OP_CHECKSIG","hex":"76a91445395b25d6339bbb8f3da68e278f21d54ba935bd88ac","reqSigs":1,"type":"pubkeyhash","addresses":["bitcoincash:qpznjke96ceehwu08kngufu0y825h2f4h5hzq38r4r"]}}
    ]
}
```

由于 btc 的高手续费, 有些钱包对于手续费的计算很不精细, 尽管对于偶尔转账, 或者是大额转账来说, 这些手续费不值一提, 但是对于需要大量
发送交易, 特别是大量使用脚本功能的用户来说, 10 倍的手续费差异是不可接受的.

计算手续费时, 要考虑的方面有:

1. inputs 数量很多, 但都是零钱, 就要考虑 input 里的的金额能否支付得起手续费.
2. 如果有多个输出有打入同一个地址的, 除非特殊要求, 应该合并在一起.
3. 有很多 utxo 可供选择的时候, 优先选择数额最大的, 这样产生的零钱就不会太小.
4. 但是对于链式交易(从 a 转到 b, 再从 b 转到 c)的长度是有限制的, 如果总是选择最大额的 utxo 来花费, 可能导致很大一部分钱在交易被确认
前, 处于不可用状态.

我们可以参考一下几个 bsv 常用开源钱包的手续费计算代码:

**bitsv(python)**:

```py
## 估算这笔交易的手续费, 参数有(input 个数, output 个数, 费率, 公钥是否被压缩过, opreturn 的大小)
def estimate_tx_fee(n_in, n_out, satoshis, compressed, op_return_size=0):

    # 费率未知, 返回 0
    if not satoshis:
        return 0

    # 估算交易体积
    estimated_size = (
        4 +  # version
        n_in * (148 if compressed else 180) # inputs 总体积
        + len(int_to_varint(n_in)) # input count 的长度
        + n_out * 34  # excluding op_return outputs, dealt with separately
        + len(int_to_varint(n_out)) # output count 的长度
        + op_return_size  # grand total size of op_return outputs(s) and related field(s)
        + 4  # time lock
    )

    estimated_fee = estimated_size * satoshis # 体积乘以费率得到估计的手续费

    logging.debug('Estimated fee: {} satoshis for {} bytes'.format(estimated_fee, estimated_size))

    return estimated_fee
```

代入之前那笔交易的数值 1 个 input, 2 个 outputs, 公钥是压缩过的, 无 opreturn, 我们得到的体积是 226, 和实际体积完全
相等. 可见, 是费率设置过高了.

# elixir 开发准备工作

# 需要安装以下程序:

# postgresql (9.0版本以上)

```bash
apt-get install postgresql-10

# 新建用户名/密码
sudo -u postgres psql
postgres=# create user myuser with encrypted password 'mypass';
```

# asdf
```bash
git clone https://github.com/asdf-vm/asdf.git ~/.asdf --branch v0.7.1

echo -e '\n. $HOME/.asdf/asdf.sh' >> ~/.bashrc
echo -e '\n. $HOME/.asdf/completions/asdf.bash' >> ~/.bashrc
```

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

# Depoly

[部署指南](./depoly.md)