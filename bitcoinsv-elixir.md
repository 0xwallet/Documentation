# BitcoinSV-elixir

本项目基于 [comboy/elixir](https://github.com/comboy/bitcoin-elixir) 修改而来, 正在开发中, 请勿用于生产环境.

# 目录

- [模块结构介绍](#模块结构)
- [网络相关模块](#网络模块)
- [比特币交易构造](#交易构造)
- [比特币脚本](#比特币脚本)
- [如何验证交易签名](#签名验证)

## 使用说明

1. [如何安装](#如何安装)
2. [如何查看bitcoin地址余额](#如何查看bitcoin地址余额)
3. [如何构建交易](#如何构建交易)
4. [如何计算手续费](#如何计算手续费)
5. [如何签名交易](#如何签名交易)

## 测试相关

- [代码测试流程](#代码测试)
- [如何测试签名](#测试签名)

## 如何安装

本项目是一个 elixir lib, 在使用本项目前, 请确保已安装了以下应用:

- [erlang 21.0 以上](http://www.erlang.org/downloads)
- [elixir 1.7 以上](https://elixir-lang.org/install.html)

之后, 就可以 clone 本项目到本地, 运行 `iex -S mix` 即可进入shell 交互.

## 如何查看bitcoin地址余额

本项目尚未提供直接查询某个比特币地址的余额的 API. 但目前可以通过组合本项目里已提供的多个 API 来达到查询余额的目的.

后续开发过程中将实现专门的余额查询 API.

目前的查询步骤如下:

1. 启动节点, 开始同步区块数据.

        ```ex
        # 确保配置文件里有以下内容, 即使用 Postgres 作为区块数据存储引擎
        config :bitcoin, :node,
          modules: [
            storage_engine: Bitcoin.Node.Storage.Engine.Postgres
        ]
        ```

2. 运行 `iex -S mix` 进入交互 shell. 这时节点程序会自动和其它比特币节点进行连接, 并同步区块数据.

3. 当已同步了的区块达到了网络中的当前区块时, 就可以在数据库中搜索对应的 outputs(即想要查询的比特币地址, 这里设为 addr1 ).

        需要查询的是 `tx_outputs` table. 查询条件是 tx_output 的 pk_script 要等于 addr1 对应的 pk_script.

        这样, 就获得了所有转入 addr1 的交易输出.

4. 此时我们已经可以得到 addr1 一共被转入了的金额. 还需要减去已经从 addr1 中转出的总金额.

        利用刚才得到的 tx_outputs, 可以找到每个 tx_outputs 的 tx_hash 与 index.

        然后, 在 `tx_inputs` table 中, 查找符合以下条件的 input:

        - input.prevout_hash == output.tx_hash
        - input.prevout_index == output.index

        能够找到符合以上条件的 input, 就代表这笔 output 已经被花出去了. 由此, 我们就得到了 addr1 已花费的金额, 以及余额.

## 如何构建交易

构建(未签名的)交易首先需要以下几个部分的数据:

- UTXO
        (未花费的交易输出, 使用 [如何查看bitcoin地址余额](#如何查看bitcoin地址余额)) 中所提到的方法, 可以找到特定地址的 UTXO.
- 交易输出
        (也就是这笔交易想要转入的地址, 以及金额)

本项目中尚未提供直接构建交易的 API, 但目前可以通过组合本项目里已提供的多个 API 来达到构建交易的目的.

后续开发过程中将实现专门的构建交易 API.

目前的构建交易步骤如下:

1. 使用 [如何查看bitcoin地址余额](#如何查看bitcoin地址余额) 中提到的方法, 获得 UTXO 列表, 设为 my_utxos.

2. 新建一个 `%Bitcoin.Protocol.Messages.Tx{}` 结构体, 设为 my_tx, 简写为 `%Tx{}`. 该结构体有以下字段:

        - version: 默认值即可
        - inputs: 需要构造
        - outputs: 需要构造
        - lock_time: 默认值即可

3. 将 my_utxos 中的每个 utxo 转换为 `%Bitcoin.Protocol.Types.Outpoint{}` 结构体, 简写为 `%Outpoint{}`:

        ```ex
        %Outpoint{
          hash: utxo.hash,
          index: utxo.index
        }
        ```

4. 将上一步得到的所有 outpoint 逐一转换成 `%Bitcoin.Protocol.Types.TxInput{}` 结构体, 简写为 `%TxInput{}`.

        每个 input 里有如下字段:
        - previous_output: 上一步得到的 outpoint 结构体
        - signature_script: 用于解锁 utxo 的签名脚本, 暂时未空
        - sequence: 未启用特性, 设为默认值即可.

5. 将上一步的到的 input 列表放入 my_tx 的 inputs 字段.
6. 根据所需的交易输出, 构造多个 `%Bitcoin.Protocol.Types.TxOutput{}` 结构体. 该结构体有以下字段:

        - value: 转入金额
        - pk_script: 锁定脚本

        最为常见的转账是使用 [P2PKH 脚本](#P2PKH). 按照 BSV 的升级路线图, 更多的脚本功能会逐渐开放.

7. 现在, my_tx 就已构造完成了. 但想要作为一个合法的交易广播到网络中, 还需要以下步骤:

- [如何计算手续费](#如何计算手续费)
- [如何签名交易](#如何签名交易)

在本项目中将实现 TxMaker 服务, 预期行为如下:

![tx_maker](tx_maker.jpg)

这里需要注意, 当使用第三方 api 的时候, 例如 Bitindex, 它需要我们主动请求, 才会返回 utxo 数据. 假设这时用户进行充值, 而我们没有主动去获取最新的 utxo, 那么显示的用户余额就是滞后的, 可能导致用户无法转账. 所以, 还需要使用能够实时推送交易的 api, 例如 BitSocket, 来告知服务器用户的 utxo 已发生变化.

**例子:**

```ex
alias Bitcoin.Tx.TxMaker
alias Bitcoin.Key
alias Bitcoin.Script

priv = Binary.from_hex("c28a9f80738f770d527803a566cf6fc3edf6cea586c4fc4a5223a5ad797e1ac3")

pub = Key.privkey_to_pubkey(priv)

pkscript = TxMaker.address_to_pk_script(pub)

script_code = [:OP_DUP, :OP_HASH160, pubkeyhash, :OP_EQUALVERIFY, :OP_CHECKSIG]

script_bin = Script.to_binary(script_code)

version = 1

lock_time = 0

sequence = 0xffffffff

hash_type = 0x41

...未完
```


# 主要功能

本项目包含 bitcoinsv 全节点所需的所有功能.

# 模块结构

在 elixir 应用中, 代码通常是按照模块(module)来划分的, 每个模块中包含了某些特定的功能, 例如 "Base58Check" 模块, 就提供了一些关于 Base58 编码的函数.

当程序运行起来后, 模块会被加载到 BEAM VM 中, 就可以对各个模块中的函数进行调用. 最初的调用者通常是我们定义的 Application 进程, 这个进程会以监控树的形式, 逐级地生成子进程. 然后每个进程里面, 会执行各自负责的代码.

一般来说, 一个文件中我们会定义一个模块, 有时一个文件中会定义多个模块. 模块的定义方式是:

```elixir
defmodule Modulename do
    ...
end
```

模块名的格式一般和文件路径是对应的. 例如, "addr.ex" 文件位于路径 "lib/bitcoin/node/network/addr.ex" . 所以在 "addr.ex" 文件中, 我们将模块名定义为 "Bitcoin.Node.Network.Addr".

模块按其代码的功能来分, 大致有以下几类:

1. 工具函数模块:

这种模块中, 只定义了一系列的工具函数, 供其它的进程使用. 例如本项目中的 "Bitcoin.Base58Check" 模块, "Bitcoin.Crypto" 模块.

2. GenServer(通用服务者)模块:

在有的模块中, 会看到 "use GenServer" 这样的代码, 这就意味着, 这个模块是实现了一个通用的服务者. 一般可以通过该模块中的 "start" 函数来启动这个服务者. 在启动之后, 这个服务者进程会一直运行, 接收其它进程发来的消息, 并根据自身状态来回复消息.

在监控树下, GenServer模块是作为 "worker"(工作者) . 本项目中, "Bitcoin.Node.Storage" 模块, "Bitcoin.Node.Inventory" 模块, 都是 GenServer 模块.

3. Supervisor(监控者)模块:

有的模块中, 会看到 "use Supervisor" 这样的代码, 这意味着, 这个模块实现了一个监控者.

在我们的程序运行过程中, 可能会遇到一些意想不到的问题, 导致某个进程崩溃, 这个时候, 就需要对崩溃了的进程进行重启. 在 BEAM VM 里, 通常每个进程都有对应的监控者进程, 监控者进程负责启动, 重启, 关闭它的子进程.

监控者模块也可以被放在监控树下, 形成一个树状的监控结构, 我们以 "supervisor"(监控者) 来标注这些模块. 本项目中, "Bitcoin.Node.Network.Supervisor" 模块, "Bitcoin.Node.Supervisor" 模块, 都是 Supervisor 模块. 注意到, 监控者模块通常以 "Supervisor" 来命名.

4. Application(应用)模块

应用模块是最顶级的模块, 也是 elixir 程序启动之后第一个执行的模块. 在应用模块中, 会看到 "use Application" 这样的代码.

本质上, Application 是一种特殊的 Supervisor, 在应用模块中也需要定义它的子进程. 本项目中的应用模块是 "Bitcoin".

5. 数据结构定义模块

在 elixir 中, 使用 struct 来定义数据结构. 一般地, 结构的名称与其定义所处于的模块名相同. 在模块中看到 "defstruct", 就表明这是一个数据结构定义模块.

----

接下来, 我们将会依照启动的顺序, 为每个模块做详细的介绍:

# 网络模块

## bitcoin.ex 模块名 Bitcoin

**类型:** Application

该文件是启动节点应用时的入口文件, 定义了 Bitcoin Application. 为了方便测试, 我们不会在每次运行 `iex -S mix` (elixir 应用的命令行交互程序)的时候都启动节点. 所以, 要让节点正常启动, 需要在"config/dev.exs" 中配置:

```ex
config :bitcoin, :node,
    modules: [storage_engine: Bitcoin.Node.Storage.Engine.Postgres]
```

这里我们配置的是 storage_engine 的回调模块, 可以是 Postgres 数据库, 也可以是其它的数据库, 只要该模块里实现了节点所需的回调函数即可.

在有了这个配置之后, 在启动程序之后, Bitcoin Application 就会读到此信息, 然后启动节点:

```ex
    # bitcoin.ex

    # Start node only if :bitcoin,:node config section is present
    children = case Application.fetch_env(:bitcoin, :node) do
      :error ->
         []
      {:ok, _node_config} ->
         [ supervisor(Bitcoin.Node.Supervisor, []) ]
    end
```

## bitcoin/node/supervisor.ex 模块名 Bitcoin.Node.Supervisor

**类型:** Supervisor

该文件定义了 bitcoin 节点的最高监控树, Application 进程启动后, 首先启动的就是该监控树.

```ex
    # bitcoin/node/supervisor.ex

    children = [
      worker(Bitcoin.Node, []),
      supervisor(Bitcoin.Node.Network.Supervisor, [])
    ]
```

该监控树下有两个子进程, 一个是 Bitcoin.Node 的 worker(普通进程), 另一个是 Bitcoin.Node.Network.Superviosr(监控树).

当子进程出错崩溃的时候, 监控树会重启子进程.

## bitcoin/node.ex 模块名 Bitcoin.Node

**类型:** GenServer

**职责:**  代表一个在运行的 Bitcoin 节点.

**这个 GenServer 暴露的 API 有:**

- start_link/0

        启动节点进程.

- version_fields/0

        获得本节点的 version 消息.

- config/0

        获得本节点的配置信息.

- nonce/0

        获得本节点所使用的随机数.

- height/0

        本节点的初始区块高度.

- protocol_version/0

        所使用的 p2p 协议的版本.

**该节点启动后的行为是:**

1. 给自己发送 :initialize 消息
```ex
  def init(_) do
    self() |> send(:initialize)
    {:ok, %{}}
  end
```

2. 收到 :initialize 消息后, 从配置文件里读取配置, 并且新建存储区块数据的文件夹(如果区块数据保存在本地的话). 生成之后要使用的随机数 nonce.
```ex
  def handle_info(:initialize, state) do
    Logger.info "Node initialization"

    config = case Application.fetch_env(:bitcoin, :node) do
      :error -> @default_config
      {:ok, config} ->
        @default_config |> Map.merge(config |> Enum.into(%{}))
    end

    File.mkdir_p(config.data_directory)

    state = state|> Map.merge(%{
      nonce: Bitcoin.Util.nonce64(),
      config: config
    })

    {:noreply, state}
  end
```

## bitcoin/node/network/supervisor.ex 模块名 Bitcoin.Node.Network.Supervisor

**类型:** Supervisor

**职责:**  它负责启动和网络有关的进程.

```ex
  def init(_) do
    Logger.info "Starting Node subsystems"

    [
      @modules[:addr],
      @modules[:discovery],
      @modules[:connection_manager],
      # Storage module is an abstraction on top of the actual storage engine so it doesn't have to be dynamic
      Bitcoin.Node.Storage,
      @modules[:inventory]
    ]
    |> Enum.map(fn m -> worker(m, []) end)
    |> supervise(strategy: :one_for_one)
  end
```

它有5个子进程, 是从 @modules 这个模块属性里读取到的, 目前, 根据它会读取到的数据, 这个监控树所有的子进程是:

- Bitcoin.Node.Network.Addr: 负责管理网络中的其它节点的地址
- Bitcoin.Node.Network.Discovery: 负责搜索 DNS, 获得种子节点的地址
- Bitcoin.Node.Network.ConnectionManager: 负责管理与其它节点的连接
- Bitcoin.Node.Storage: 负责存储
- Bitcoin.Node.Inventory: 负责获取缺失的交易或区块信息, 在获取到之后, 广播给其它节点

以上进程全部都是 GenServer.

## bitcoin/node/network/addr.ex 模块名 Bitcoin.Node.Network.Addr

**类型:** GenServer

**职责:**  负责管理网络中的其它节点的地址.

**它暴露出来的 API 有:**

- start_link/0

        启动.

- add/1

        添加新的节点地址.

- get/0

        获取随机的一个节点的地址.

- count/0

        计算已知的节点的数量.

- clear/0

        删除所有的地址.

**它的行为机制是:**

1. 启动后, 过60秒, 给自己发送一个 :periodical_persistance 消息.
2. 收到 :periodical_persistance 消息后, 过60秒, 会再给自己发送一个 :periodical_persistance 消息. 并且删除超出限制的节点地址(目前上限1000个), 并保存剩余的地址到持久存储设备上.

## bitcoin/node/network/discovery.ex 模块名 Bitcoin.Node.Network.Discovery

**类型:** GenServer.

**职责:** 查找种子节点的地址.

**APIs:**

- start_link/0

        启动进程, 并且添加与父进程之间的 link.

> link 的作用是, 子进程崩溃的时候, 会传导到父进程.

- begin_discovery/0

        开始 DNS 搜索.

**行为模式:**

该进程在收到 :begin_discovery 消息之后, 会执行 DNS 搜索策略, 即 "Strategy.DNS.gather_peers/1" . 根据配置中已知的域名, 来搜索 DNS 服务器上的 A 记录, 获取到种子节点的 ip 列表. 然后将这些节点地址发送给负责管理节点地址的进程.

## bitcoin/node/network/connection_manager.ex 模块名 Bitcoin.Node.Network.ConnectionManager

**类型:** GenServer.

**职责:** 管理节点与其它 BSV 节点之间的连接.

**APIs:**

- start_link/0

        启动进程, 并且添加与父进程之间的 link.

- connect(ip, port)

        请求根据 ip 和端口号, 建立 TCP 连接.

- register_peer/0

        由负责与其它节点保持连接的 peer 进程, 向这个 ConnectionManager 进程发送注册请求.

- peers/0

        获取 ConnectionManager 进程所有的 peer 信息.

**行为模式:**

ConnectionManager GenServer 的内部状态有:

- config: 配置信息
- peers: 节点列表

在启动本 GenServer 时, 会使用 "Reagent.start(ReagentHandler, port: port)" 函数来新建 Socket, 然后启动一个 peer 进程, 并将 Socket 移交给 peer 进程.

这里需要重点介绍一下 Reagent. Reagent 是用于实现 Socket 连接池的. 很多情况下, 我们不想实现一个完整的 GenServer 来处理 TCP 连接, 而是仅仅需要一个函数来 handle 连接, 这时候就可以用 reanget 来轻松实现. 在本项目中, 我们的 Bitcoin 节点在连接到区块链网络里之后, 会收到其它节点发来的 TCP 连接请求, 这里使用 Regaent 来处理这些请求.

启动 Reagent 服务, 需要调用 Reagent.start(ReagentHandler, options) , 这里的 ReagentHandler 是我们自己实现的 Reagent 定义, options 里包括端口号等配置信息.

要自定义一个 ReagentHandler, 需要实现 start/0 和 handle/1 这两个函数回调. 采用 "use Reagent" 的时候, 可以省略 start/0 的实现. 在本项目中, 是这样定义 Reagent 的:

```elixir
  # Reagent connection handler
  defmodule ReagentHandler do
    use Reagent
    use Bitcoin.Common

    def handle(%Reagent.Connection{socket: socket}) do
      {:ok, pid} = @modules[:peer].start(socket)
      # Potential issue:
      # If the connection gets closed after Peer.start but before switching the controlling process
      # then probably Peer will never receive _:tcp_closed. Not sure if we need to care because
      # it should just timout then
      socket |> :gen_tcp.controlling_process(pid)
      socket |> :inet.setopts(active: true)
      :ok
    end
  end
```

我们的 Reagent 服务在接收到新的 socket 连接时, 会启动一个专门的 peer 进程, 并将 socket 移交个这个 peer 进程, 然后将这个 socket 设置为 active 状态(所有通过这个 socket 发送来的消息都会被转发给拥有这个 socket 的进程, 这里也就是 peer 进程.).

在 GenServer ConnectionManager 启动之后, 如果配置中没有预先定义好的节点地址列表, 本 GenServer 就会给自己发送一个 :periodical_connectivity_check 消息. 如果有预先定义好的节点, 本 GenServer 就会主动与这些节点建立连接.

在收到 :periodical_connectivity_check 消息时, 首先会给自己发送一个 :check_connectivity 消息, 并且在 10 秒钟之后, 给自己再次发送 :periodical_connectivity_check 消息.

在收到 :check_connectivity 消息时, 会计算当前已知的节点连接数, 如果还未到达上限, 就会调用 add_peer/1 函数, 来添加新的连接.

## bitcoin/node/storage.ex 模块名 Bitcoin.Node.Storage

**类型:** GenServer

**职责:** 区块数据和交易数据的持久化.

**APIs:**

- start_link/1

        启动 GenServer

- store/2

        存储交易或者区块.

- max_height/0

        已知的最大区块高度.

- get_block_with_height/1

        根据高度获取到区块数据.

- store_block/2

        存储区块数据.

- block_height/1

        根据区块数据来得出区块高度.

**行为:**

在 Storage 进程启动时, 首先会启动存储引擎, 例如 PostgreSQL 的客户端进程. 在存储引起启动成功后, 判断一下是否已经有区块数据, 如果没有, 就将创世区块的数据存入存储引擎.

在本项目目前的代码中, Storage GenServer 在存储区块之前, 还要兼具验证区块的工作. 包括每个交易的所有输入是否已经存在, 等等.

## bitcoin/node/inventory.ex 模块名 Bitcoin.Node.Inventory

**类型:** GenServer

**职责:** 从远程节点(peers) 那里获取缺失的数据, 并在验证后广播. 获取到的数据应当被添加到 storage 或者 mempool 中.

**APIs:**

- start_link/1

        启动 GenServer

- seen/1

        其它 peers 看到了新的INV 消息时, 通过调用次函数来向本 GenServer 报告.

- add/1

        将具体的区块数据存储起来.

- request_item/2

        想某个 peer 请求某种数据.

- check_sync/1

        检查本 GenServer 是否处于等待接收区块的状态, 如果不是, 则进行更多的 sync.

- check_orphans/1

        检查本 GenServer 收到的孤块是否已经找到父块. 如以找到, 则将孤块的状态改为 :present.

- block_locator_hashes/0, block_locator_hashes/4

        计算从某个区块回溯得到的这条链的 block_locator_hashes.

        > block_locator 是一系列的区块哈希, 用于描述一条链.从最高的区块开始回溯, 前十个块步长为1, 之后每往前一个块, 步长翻倍.

**行为:**

在 Inventory 进程启动时, 会给自己发送以下两条信息: :periodical_sync 和 :periodical_cleanup.

在收到 :periodical_sync 消息后, Inventory 进程会先给自己发送一个 :sync 消息, 并且在 20 秒后再次给自己发送 :periodical_sync 消息.

在收到 :sync 消息后, 首先判断是否和其它节点有网络连接, 如果有, 则向随机的一个远程节点发送获取区块的请求. 如果没有, 则 10 秒后再次给自己发送 :sync 消息.

----

以上, 就是本项目中主要的 "进程定义模块"(Application, GenServer, Supervisor). 接下来的文档是关于其它类型的模块(数据结构定义, 工具函数).

## Bitcoin.Base58Check

**类型:** 工具函数模块.

**介绍:** 比特币地址才用 Base58 编码, 本模块提供了一系列 Base58 编码解码函数.

**APIs:**

- encode/1

        由 binary 格式编码成 Base58 格式.

- decode/1

        将 Base58字符串(普通比特币地址) 解码成 binary 格式.

- decode!/1

        将 Base58字符串(普通比特币地址) 解码成 binary 格式, 解码失败时抛出异常.

- valid?/1

        判断一个字符串是否是合法的 Base58 格式.

- base_encode/1

        由 binary 格式编码成 Base58 格式.(不含 checksum).

- base_decode/1

        将 Base58字符串(普通比特币地址) 解码成 binary 格式(不含 checksum).

- base_decode!/1

        将 Base58字符串(普通比特币地址) 解码成 binary 格式(不含 checksum). 解码失败时抛出异常.

- base_valid?/1

        判断一个字符串是否是合法的 Base58 格式. (不含 checksum).

## Bitcoin.Crypto

封装了一些需要用到的加密函数.

**APIs:**

- ripemd160/1

        哈希160.

- sha1/1

        sha1 哈希.

- sha256/1

        sha256 哈希.




# 代码测试

在本项目中, 包含了大量的测试代码, 以此保证代码的正确运行, 且便于在修改和新增功能的时候, 确保旧的代码没有受到影响.
本项目使用 elixir 通用的测试工具 ExUnit 来进行单元测试. 对于不同类型的模块, 采用的方法也是不同的. 最简单的是工具函数模块, 只需要准备好测试输入以及预期的正确结果, 执行该模块中的工具函数, 判断结果是否正确即可. 这种测试可以被称为无状态测试.

而有一些业务代码涉及到数据库, 进程的启动和终结, 错误处理, 等等具有副作用的函数. 针对这类代码, 测试流程就更为复杂, 需要启动虚拟的数据库(或专门的测试数据库), 或者在测试开始时启动一系列的进程, 在测试结束后关闭进程. 这种测试可以被称为有状态测试.

打开命令行, 在项目根目录下, 运行 "mix test" 即可开始测试.

## 测试签名

首先, 要确保签名算法是正确的. 最简单的方法就是找到一个比较流行的开源 bsv 钱包实现, 来签名并广播一笔交易. 然后获取到完整的签名数据, 和我们自己的实现来作对比.

这里使用 https://github.com/AustEcon/bitsv 来作为对照. 首先导入我们自己的私钥. 签发一笔交易.

我们得到了这笔交易: https://blockchair.com/bitcoin-sv/transaction/1a0884356fd9cdee5322b038ea20c4f7d006d20670f97aafc4aa5aadf3c62d2e

raw transaction:
```
0100000001f6006dbbeda24ff5e8d032d8f97c05bf5d0392f6adcc3462cacc180435e52d1f000000006a473044022064d13442cc47d55add49898a8c618a601dce110d67b56b6654fec1b0e95b2d13022015cba3c4b0f0fd36912192dd75ec72c9c5613c9bd00544b40f85ff78e8f436a24121024da90ca8bf7861e2bee6931de4588ebba3850a1ad3f05ccd45cad2dd17ba7ae7ffffffff0210270000000000001976a914f84e64817bcb214871a90d0dce34685377cbf48788ac16edbf00000000001976a914926f915bd7285586ae795ba40461d3d4ae53760888ac00000000
```

bitcoinsv-elixir 解码后的格式:
```ex
%Bitcoin.Protocol.Messages.Tx{
  inputs: [
    %Bitcoin.Protocol.Types.TxInput{
      previous_output: %Bitcoin.Protocol.Types.Outpoint{
        hash: <<246, 0, 109, 187, 237, 162, 79, 245, 232, 208, 50, 216, 249,
          124, 5, 191, 93, 3, 146, 246, 173, 204, 52, 98, 202, 204, 24, 4, 53,
          229, 45, 31>>,
        index: 0
      },
      sequence: 4294967295,
      signature_script: <<71, 48, 68, 2, 32, 100, 209, 52, 66, 204, 71, 213, 90,
        221, 73, 137, 138, 140, 97, 138, 96, 29, 206, 17, 13, 103, 181, 107,
        102, 84, 254, 193, 176, 233, 91, 45, 19, 2, 32, 21, 203, 163, 196, 176,
        240, 253, 54, 145, 33, 146, 221, 117, 236, 114, 201, 197, 97, 60, 155,
        208, 5, 68, 180, 15, 133, 255, 120, 232, 244, 54, 162, 65, 33, 2, 77,
        169, 12, 168, 191, 120, 97, 226, 190, 230, 147, 29, 228, 88, 142, 187,
        163, 133, 10, 26, 211, 240, 92, 205, 69, 202, 210, 221, 23, 186, 122,
        231>>
    }
  ],
  lock_time: 0,
  outputs: [
    %Bitcoin.Protocol.Types.TxOutput{
      pk_script: <<118, 169, 20, 248, 78, 100, 129, 123, 203, 33, 72, 113, 169,
        13, 13, 206, 52, 104, 83, 119, 203, 244, 135, 136, 172>>,
      value: 10000
    },
    %Bitcoin.Protocol.Types.TxOutput{
      pk_script: <<118, 169, 20, 146, 111, 145, 91, 215, 40, 85, 134, 174, 121,
        91, 164, 4, 97, 211, 212, 174, 83, 118, 8, 136, 172>>,
      value: 12578070
    }
  ],
  version: 1
}
```

Blockchair docoded transaction:
```json
{"txid":"1a0884356fd9cdee5322b038ea20c4f7d006d20670f97aafc4aa5aadf3c62d2e","hash":"1a0884356fd9cdee5322b038ea20c4f7d006d20670f97aafc4aa5aadf3c62d2e","size":225,"version":1,"locktime":0,"vin":[{"txid":"1f2de5350418ccca6234ccadf692035dbf057cf9d832d0e8f54fa2edbb6d00f6","vout":0,"scriptSig":{"asm":"3044022064d13442cc47d55add49898a8c618a601dce110d67b56b6654fec1b0e95b2d13022015cba3c4b0f0fd36912192dd75ec72c9c5613c9bd00544b40f85ff78e8f436a2[ALL|FORKID] 024da90ca8bf7861e2bee6931de4588ebba3850a1ad3f05ccd45cad2dd17ba7ae7","hex":"473044022064d13442cc47d55add49898a8c618a601dce110d67b56b6654fec1b0e95b2d13022015cba3c4b0f0fd36912192dd75ec72c9c5613c9bd00544b40f85ff78e8f436a24121024da90ca8bf7861e2bee6931de4588ebba3850a1ad3f05ccd45cad2dd17ba7ae7"},"sequence":4294967295}],"vout":[{"value":0.0001,"n":0,"scriptPubKey":{"asm":"OP_DUP OP_HASH160 f84e64817bcb214871a90d0dce34685377cbf487 OP_EQUALVERIFY OP_CHECKSIG","hex":"76a914f84e64817bcb214871a90d0dce34685377cbf48788ac","reqSigs":1,"type":"pubkeyhash","addresses":["bitcoincash:qruyueyp009jzjr34yxsmn35dpfh0jl5su6wtf3gyr"]}},{"value":0.1257807,"n":1,"scriptPubKey":{"asm":"OP_DUP OP_HASH160 926f915bd7285586ae795ba40461d3d4ae537608 OP_EQUALVERIFY OP_CHECKSIG","hex":"76a914926f915bd7285586ae795ba40461d3d4ae53760888ac","reqSigs":1,"type":"pubkeyhash","addresses":["bitcoincash:qzfxly2m6u59tp4w09d6gprp6022u5mkpqkwa7997y"]}}]}
```

以上是可以确定正确的数据. 接下来使用我们自己的实现, 对同样的原料进行签名, 得到的结果是:

```
0100000001f6006dbbeda24ff5e8d032d8f97c05bf5d0392f6adcc3462cacc180435e52d1f000000006b483045022100e85678e98cac4040f441831ea246a7ba9522c69d6a29c41ffccbd71364823c8b02200a999bc6eb0b57c12fb79317fb76327a8d4a74533542cb6a650141747ed9de7a4121024da90ca8bf7861e2bee6931de4588ebba3850a1ad3f05ccd45cad2dd17ba7ae7ffffffff0210270000000000001976a914f84e64817bcb214871a90d0dce34685377cbf48788ac16edbf00000000001976a914926f915bd7285586ae795ba40461d3d4ae53760888ac00000000
```

bitcoinsv-elixir 解码后的格式:
```ex
%Bitcoin.Protocol.Messages.Tx{
  inputs: [
    %Bitcoin.Protocol.Types.TxInput{
      previous_output: %Bitcoin.Protocol.Types.Outpoint{
        hash: <<246, 0, 109, 187, 237, 162, 79, 245, 232, 208, 50, 216, 249,
          124, 5, 191, 93, 3, 146, 246, 173, 204, 52, 98, 202, 204, 24, 4, 53,
          229, 45, 31>>,
        index: 0
      },
      sequence: 4294967295,
      signature_script: <<72, 48, 69, 2, 33, 0, 232, 86, 120, 233, 140, 172, 64,
        64, 244, 65, 131, 30, 162, 70, 167, 186, 149, 34, 198, 157, 106, 41,
        196, 31, 252, 203, 215, 19, 100, 130, 60, 139, 2, 32, 10, 153, 155, 198,
        235, 11, 87, 193, 47, 183, 147, 23, 251, 118, 50, 122, 141, 74, 116, 83,
        53, 66, 203, 106, 101, 1, 65, 116, 126, 217, 222, 122, 65, 33, 2, 77,
        169, 12, 168, 191, 120, 97, 226, 190, 230, 147, 29, 228, 88, 142, 187,
        163, 133, 10, 26, 211, 240, 92, 205, 69, 202, 210, 221, 23, 186, 122,
        231>>
    }
  ],
  lock_time: 0,
  outputs: [
    %Bitcoin.Protocol.Types.TxOutput{
      pk_script: <<118, 169, 20, 248, 78, 100, 129, 123, 203, 33, 72, 113, 169,
        13, 13, 206, 52, 104, 83, 119, 203, 244, 135, 136, 172>>,
      value: 10000
    },
    %Bitcoin.Protocol.Types.TxOutput{
      pk_script: <<118, 169, 20, 146, 111, 145, 91, 215, 40, 85, 134, 174, 121,
        91, 164, 4, 97, 211, 212, 174, 83, 118, 8, 136, 172>>,
      value: 12578070
    }
  ],
  version: 1
}
```

进过对比, 我们发现只有 `signature_script` 部分存在差异. 在P2PKH 交易中, signature_script 由三部分组成: 1.签名 2.sighash_type 3.public_key. 上述两个交易中的三部分拆分出来得到:

结果1:
```
1. 47
3044022064d13442cc47d55add49898a8c618a601dce110d67b56b6654fec1b0e95b2d13022015cba3c4b0f0fd36912192dd75ec72c9c5613c9bd00544b40f85ff78e8f436a2
2. 41
3. 21
024da90ca8bf7861e2bee6931de4588ebba3850a1ad3f05ccd45cad2dd17ba7ae7
```

结果2:
```
1. 48
3045022100e85678e98cac4040f441831ea246a7ba9522c69d6a29c41ffccbd71364823c8b02200a999bc6eb0b57c12fb79317fb76327a8d4a74533542cb6a650141747ed9de7a
2. 41
3. 21
024da90ca8bf7861e2bee6931de4588ebba3850a1ad3f05ccd45cad2dd17ba7ae7
``

可以看出, sighash_type 和 public_key 部分是一样的. 只有签名部分有差异.

Bitcoin 的签名使用的是 DER encoding, 所以先使用 `Bitcoin.DERSig.parse/1` 解码一下签名:

结果1:
```ex
%Bitcoin.DERSig{
  length: 68,
  r: <<100, 209, 52, 66, 204, 71, 213, 90, 221, 73, 137, 138, 140, 97, 138, 96,
    29, 206, 17, 13, 103, 181, 107, 102, 84, 254, 193, 176, 233, 91, 45, 19>>,
  r_type: 2,
  s: <<21, 203, 163, 196, 176, 240, 253, 54, 145, 33, 146, 221, 117, 236, 114,
    201, 197, 97, 60, 155, 208, 5, 68, 180, 15, 133, 255, 120, 232, 244, 54,
    162>>,
  s_type: 2,
  type: 48
}
```

结果2:
```ex
%Bitcoin.DERSig{
  length: 69,
  r: <<0, 232, 86, 120, 233, 140, 172, 64, 64, 244, 65, 131, 30, 162, 70, 167,
    186, 149, 34, 198, 157, 106, 41, 196, 31, 252, 203, 215, 19, 100, 130, 60,
    139>>,
  r_type: 2,
  s: <<10, 153, 155, 198, 235, 11, 87, 193, 47, 183, 147, 23, 251, 118, 50, 122,
    141, 74, 116, 83, 53, 66, 203, 106, 101, 1, 65, 116, 126, 217, 222, 122>>,
  s_type: 2,
  type: 48
}
```

并不能因此确认本项目中的实现是错误的, 因为 ECDSA 签名算法中会引入随机数, 所以每次签名的结果会不同.

**在 bitcoinsv-elixir 中签名的方法**
```
alias Bitcoin.Crypto
alias Bitcoin.Key

msg = "hello"
hashed = Crypto.sha256(msg)
privkey = :crypto.strong_rand_bytes(64)
pubkey = Key.privkey_to_pubkey(privkey)
sig = Crypto.sign(privkey, hashed)

# 得到sig:
# 3045022100d8cf8ab7f2e0ff53d7bb9aedcd3fcff8f35d56733938a333696d51d46898bba10220168a3c7688966152b3aab392548630ce0c7da1bffc2227e28beadcb75bfc0eaf
# pubkey:
# 03864fcb0e3dfdd244ce5df56aebcf66869012752640ee3db9921cdbbfefdf10a5
```




# 交易构造

比特币的每笔交易, 广播到网络中之后, 矿工会对其进行验证, 验证的步骤包括检查其输入是否存在, 输出的总金额是否小于输入的总金额, 脚本运行是否可以得到 true.

人们常说的验证签名, 其实只是在脚本中包含了验证签名的 opcodes, 如果一个 UTXO(未花费的交易输出) 里没有包含验证签名的 opcodes, 那么是不会进行签名验证的.

比特币脚本类似于 Forth 语言, 是基于双栈的. 脚本执行的结果只有 true 或 false. 在本项目中, 实现了一个比特币脚本的运行时(模块 Bitcoin.Script).

脚本习惯性地被分为两个部分, pk_script 和 sig_script, pk_script 又称锁定脚本, 被放在 output 里. sig_script 又称解锁脚本, 被放在 input 里.

本项目中, 比特币脚本在执行的时候, 需要提供的数据有:

- tx: 完整的交易.
- input_number: 该交易的输入个数.(用于 sighash, 即特殊方法的签名验证)
- sub_script: 同样被用于 sighash, 通常等于 pk_script.
- flags: 脚本验证的选项. (例如 %{p2sh: true, dersig: true})

以下是与交易构造有关的模块:

## bitcoin/protocol/messages/tx.ex 模块名 Bitcoin.Protocol.Messages.Tx

**类型:** 数据结构定义模块.

比特币交易的数据结构定义在此模块中, 有以下几个字段:

- version
- inputs
- outputs
- lock_time

此模块定义了交易的编码和解码函数.

**APIs:**

- parse_stream/1

        将交易从 binary 格式转换为 tx 结构体, 并保留剩余部分.

- parse/1

        将交易从 binary 格式转换为 tx 结构体, 不返回剩余部分

- serialize/1

        将tx 结构体转换为 binary 格式.

## bitcoin/protocol/types/outpoint.ex 模块名 Bitcoin.Protocol.Types.Outpoint

**类型:** 数据结构定义模块.

Outpoint 相当于是交易输出的坐标, 在交易的每个 Input 中都要用到. 它包含以下几个字段:

- hash: 交易的哈希
- index: 此 output 是这笔的第几个 output

**APIs:**

- parse_stream/1

        从 binary 格式转换为 Outpoint 结构体, 并保留剩余部分.

- parse/1

        从 binary 格式转换为 Outpoint 结构体, 不返回剩余部分

- serialize/1

        将 Outpoint 结构体转换为 binary 格式.


## bitcoin/protocol/types/tx_input.ex 模块名 Bitcoin.Protocol.Types.TxInput

**类型:** 数据结构定义模块.

TxInput 表示交易中的输入, 包含以下几个字段:

- previous_output: UTXO, 以 Outpoint 结构体的形式
- signature_script: 解锁脚本.
- swquence: 用于实现"交易替换"功能, 目前未启用

**APIs:**

- parse_stream/1

        从 binary 格式转换为 TxInput 结构体, 并保留剩余部分.

- serialize/1

        将 TxInput 结构体转换为 binary 格式.

## bitcoin/protocol/types/tx_output.ex 模块名 Bitcoin.Protocol.Types.TxOutput

**类型:** 数据结构定义模块.

TxOutput 表示交易中的输出, 包含以下几个字段:

- value: 金额
- pk_script: 锁定脚本

**APIs:**

- parse_stream/1

        从 binary 格式转换为 TxOutput 结构体, 并保留剩余部分.

- serialize/1

        将 TxOutput 结构体转换为 binary 格式.

## bitcoin/tx.ex 模块名 Bitcoin.Tx

**类型:** 工具函数模块.

本模块中提供了一系列用于操作 Tx 数据结构的方法函数.

**APIs:**

- sighash/4

        计算出交易签名的哈希.
        得到的结果与真正的 sighash 相差一次哈希运算, 这是因为 :crypto.verify/5 函数在验证签名时会先多做一次哈希运算.

- hash/1

        计算 Tx 结构体的哈希(首先将结构体序列化).

- total_output_value/2

        计算出交易的总输出(单位是 satoshis).

- total_input_value/2

        计算出交易的总输入(单位是 satoshis).

- fee/2

        计算出交易的手续费(总输入减去总输出).

- validate/2

        验证交易的合法性.
        在这个函数中, 会检查以下内容:

        1. 之前的 outputs 是否存在
        2. 脚本执行的结果是否合适
        3. 总输入是否大于总输出
        4. 之前的 output 是否已经被花费

## bitcoin/der_sig.ex 模块名 Bitcoin.DERSig

**类型:** 数据结构定义模块 + 工具函数模块.

本模块中定义了 DERSig 结构体, 同时也提供了一系列用于操作 DER 编码签名的方法函数. DER 格式的签名被用于 Bitcoin Script 中.

由于 :crypto.verify 函数在验证签名时, 如果签名的 R 或 S 值是有补0的, 则不能通过验证. 所以在验证签名之前, 需要对签名进行 normalize 操作.

DER 签名包含的字段有:

- type: 类型
- total_length: 总长度
- r_type
- r_length
- r
- s_type
- s_length
- s

**APIs:**.

- parse/1

        从 binary 格式转换为 DERSig 结构体.

- serialize/1

        将 DERSig 结构体转换为 binary 格式.

- normalize/1

        规范化签名. 步骤有:
        1. 消除 R 和 S 值之前的补0
        2. 修正 total_length
        3. 修正负的 S
        4. 修正负的 R
        5. 保证 S 为 low S

- low_s?/1

        判断 S 值是否小于等于 order/2.

**Low S 签名**:

在比特币中, 要求签名的 S 值必须是在 0x1 到 0x7FFFFFFF FFFFFFFF FFFFFFFF FFFFFFFF 5D576E73 57A4501D DFE92F46 681B20A0 (包含) 范围内. 如果 S 值过高, 只需要让 S' =  0xFFFFFFFF FFFFFFFF FFFFFFFF FFFFFFFE BAAEDCE6 AF48A03B BFD25E8C D0364141 - S 即可.

- strict?/1

        根据 BIP66, 判断签名是否是严格的 DER 签名.

**BIP66:**

BIP66 是 2015年 BitcoinCore 开发者引入的. 原因是要减少对 OpenSSL 签名验证的依赖. 由于 OpenSSL 实现的不确定性, 可能会引起签名验证在不同版本的 OpenSSL 上会有不同的结果. 所以需要对签名验证流程做详细的规范. 最终目标是让所有的比特币节点实现都不再依赖 OpenSSL.

每个传送给 OP_CHECKSIG, OP_CHECKSIGVERIFY, OP_CHECKMULTISIG, 或 OP_CHECKMULTISIGVERIFY 的签名, 都需要用到 ECDSA 验证, 都必须遵守严格 DER 编码.

这些操作符都是在栈(stack)上对 pubkey/signature pair(公钥签名对) 做 ECDSA 签名验证. 验证过程中, 如果发现签名的编码格式不正确, 应该立刻让整个脚本的运行结果返回 false. 如果签名格式正确, 但没有通过 ECDSA 验证, 那么应该继续执行后续的脚本.

有以下情况出现时, 即判定 DER 签名格式不是严格的:

1. 长度小于8字节
2. 长度大于72字节
3. der类型 不等于 0x30
4. der长度 不等于 签名长度减去2
5. der长度 不等于 r_length + s_length + 4
6. r_type 不等于 0x02
7. r 等于 空字符串
8. r 为 负值
9. r 有 前补0
10. s_type 不等于 0x02
11. s 等于 空字符串
12. s 为 负值
13. s 有 前补0

以上情况都没有出现时, 即判定 DER 签名时严格的.


# 比特币脚本

与脚本执行相关的模块有:

- Bitcoin.Script.Serialization: 将脚本从 binary 格式转换为 opcode list.
- Bitcoin.Script.Control: 用于解析 OP_IF 这类条件语句.
- Bitcoin.Script.Number: 用于编码解码整数(即原始 bitcoin 节点代码中的 CScriptNum).
- Bitcoin.Script.Interpreter: 解释器, 用于运行 OPCODEs.
- Bitcoin.Script.Opcodes: 定义 Opcodes 的名称和值之间的关系.

## bitcoin/script.ex 模块名 Bitcoin.Script

**类型:** 工具函数模块.

该模块是脚本相关功能的入口.

**APIs:**

- parse/1

        将 binary 格式的脚本转换成类似于 "[:OP_10, :OP_10, :OP_ADD, <<20>>, :OP_EQUAL]" 这样的列表.

**P2PKH:**

P2PKH 是 "Pay to Public Key Hash" 的缩写, 是比特币中最常见的交易脚本格式.

例如:
```ex
 [:OP_DUP, :OP_HASH160, <<195, 152, 239, 169, 195, 146, 186, 96, 19, 197, 224, 78, 231, 41, 117, 94,  247, 245, 139, 50>>, :OP_EQUALVERIFY, :OP_CHECKSIG]
 ```

 就是一个 P2PKH 脚本, 中间的 binary 是公钥的哈希, 是由比特币地址变换而来的.


- to_bianry/1

        将 opcodes 列表格式的脚本转换成 binary 格式.

- to_string/1

        将 opcodes 列表格式的脚本转换成 bitcoind 兼容的解码后的脚本格式.

- parse_string/1

        将 bitcoind 解码的脚本格式转换为 opcodes 列表的格式.

- parse_string/1

        将测试用例中的格式转换为 opcodes 列表的格式.

- exec/2

        执行给定的脚本, 返回stack.

- verify_sig_pk/2

        分别验证 sig_script 和 pk_script, 返回运行结果的布尔值.

- verify/2

        执行一段脚本, 返回运行结果的布尔值.

- cast_to_bool/1

        将exec 的运行结果变为布尔值.

## bitcoin/script/interpreter.ex 模块名 Bitcoin.Script.Interpreter

**类型:** 工具函数模块.

**APIs:**

- validate/1

        如果脚本有以下任一情况出现, 则判定其不合法:
        1. 脚本中包含任何已禁用了的操作符;
        2. 超过操作符数量上限(OP_0..OP_16, 以及 OP_RESERVED 不计在内).

- run/3

        运行已变换成 opcode list 格式的脚本.

**一般情况下, 脚本是按顺序执行的, 除了以下特殊情况:**

1. 控制语句(IF, ELSE, NOTIF, ENDIF):

例如 "IF [a] ELSE [b] ENDIF" , 需要在运行之前, 先解析出 [a] 和 [b] 的脚本. 在本项目中, Bitcoin.Script.Control 模块中的 extract 系列函数是专门用于解析此类控制语句的. 步骤如下:

当读取到 IF 操作符时, 对脚本的剩余部分调用 parse_if 函数. 该函数包含这些参数: if_block(即代码块 a), else_block(即代码块 b), script(即剩余脚本), depth(控制语句的嵌套深度).

当 parse_if 函数读取到 ENDIF 操作符, 且嵌套深度为 0 时, 返回 if_block 和 else_block, 以及剩余的 script.

当 parse_if 函数读取到 ELSE 操作符, 且嵌套深度为 0 时, 调用 parse_else 函数.

当 parse_if 函数读取到 IF, NOTIF, ENDIF 中的任意一个, 且嵌套深度不为0 时, 会根据情况改变嵌套深度.

parse_else 的行为与 parse_if 类似, 只是会将操作符读取到 else_block 中.

每次调用 extract 系列函数, 只会解析出一层的控制语句(即 {if_block, else_block, rest_script}), 然后根据条件的真或假来继续调用 run 函数运行 if_block 或者 else_block.

2. 签名验证操作符(CHECKMULTISIG[VERIFY], CHECKSIG[VERIFY]).

这类操作符的运行机制将在 Bitcoin.Tx 模块中详细解释.

## bitcoin/script/number.ex 模块名 Bitcoin.Script.Number

**类型:** 工具函数模块.

本模块定义了一系列用于操作 Script Integer 的方法函数. 比特序列被转换成 little-endian 的可变长度整数. 最高位(小于0x80的需要在签名补0)表示整数的符号. 因此 0x81 (即[1, 0, 0, 0, 0, 0, 0, 1])表示 -1, 0x80 (即[1, 0, 0, 0, 0, 0, 0, 0])表示表示负0. 正0使用无长度的字节串来表示.

数学运算操作符可接受的最大整数是 4 个字节, 但是运算结果可能会超出限制.

**APIs:**

- num/2

        将 binary 格式的 ScriptInteger, 转换为普通的整数.

- bin/1

        将整数转换为 binary 格式的 ScriptInteger.

**更多例子:**

        <<0xFF>>(即 [1, 1, 1, 1, 1, 1, 1, 1]) 表示 -127
        <<0x82>>(即 [1, 0, 0, 0, 0, 0, 1, 0]) 表示 -2
        <<0x11>>(即 [0, 0, 0, 1, 0, 0, 0, 1]) 表示 17
        <<0xFF, 0xFF, 0xFF, 0xFF>>(即 [255, 255, 255, 255], 第一个 byte 用二进制表示是 [1, 1, 1, 1, 1, 1, 1, 1], 按 ScriptInteger 格式表示的是 -127), 表示 -2147483647.

# 签名验证

参考 https://en.bitcoin.it/wiki/OP_CHECKSIG

对于普通的交易 input, 如果当前这笔交易的创造者可以成功地制造一个 sigscript , 其中使用了正确的 pubkey(对应要花费的 output 里面的 pkscript), 那么这个交易 input 就被认为是合法的.

验证签名的过程实际上是 OP_CHECKSIG 的执行过程. 所需要的参数, 除了 stack 上的参数, 以及 script code 本身, 还需要知道当前的交易, 以及当前 input 的 index.

OP_CHECKSIG 的运行流程如下:

1. pubkey 和 signature 从 stack 上被取出,  signature 的格式是 `[<DER signature> <1 byte hash-type>]`. hashtype 的值是签名的最后一字节.

2. 新的 subscript 被创造出来. 从最先遇到的 OP_CODESEPARATOR 一直到 script 的结尾, 都属于 subscript. 如果没有 OP_CODESEPARATOR, 那么整个 script 变成 subscript.

3. 任何 sig 都会被从 subscript 里删除.

4. 剩余的 OP_CODESEPARATOR 会被从 subscript 删除.

5. hashtype 被从 sig 的最后 1 字节取下来并存储.

6. 复制一份当前的交易 txcopy.

7. txcopy 中所有 input 的 script 都被设置成空. (1 字节 0x00).

8. txcopy 中当前的 input 的 script 被设置成 subscript. (前缀verint 格式的长度).

接下来根据 txcodpy 中不同的 hashtype, 处理方法也不同.

一般的比特币转账交易采用 sighhash_all 类型: 不需要额外的处理.

![opchecksig](Bitcoin_opchecksig.jpg)

> 图片来自 Bitcoin Wiki

由此, 我们可以逆推出构造签名原文的步骤.

