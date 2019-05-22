Genesis 是 unwriter 基于 planaria 框架开发的一个比特币应用后端, 它的功能很基础, 就是抓取所有的交易信息, 然后以可索引的形式存储到 MongoDB 中.

以下是 Genesis 的 planaria.js 文件, 我们逐行来解读:
```js
/***************************************
*
* Crawl Everything 抓取所有东西
*
***************************************/
var compact = function(outputs) {
  let newOuts = []
  outputs.forEach(function(o) {
    let newOut = {}
    if (o) {
      Object.keys(o).forEach(function(k) {
        if (!k.startsWith("l") && k !== 'str') {
          newOut[k] = o[k]
        }
      })
    } else {
      newOut = o;
    }
    newOuts.push(newOut)
  })
  return newOuts
}
module.exports = { // 导出以下参数, 作用类似于配置文件
  from: 525470,  // 从次高度开始抓取数据
  name: 'genesis',  // 应用名称
  version: '0.0.5',  // 版本号
  description: 'genesis bitdb',  // 应用描述
  address: '1FnauZ9aUH2Bex6JzdcV4eNX7oLSSEbxtN',  // 每个应用都有一个唯一的比特币地址作为 ID
  index: { // 索引定义
    c: {  // 以 bitquery 的规范, c 一般指已确认的交易
      keys: [
        'tx.h', // 交易哈希
        'blk.i', // 区块高度
        'blk.t', // 区块时间戳
        'blk.h', // 区块哈希
        'in.e.a',
        'in.e.h', 'in.e.i', 'in.i',
        'out.e.a', 'out.e.i', 'out.e.v', 'out.i',
        'in.b0', 'in.b1', 'in.b2', 'in.b3', 'in.b4', 'in.b5', 'in.b6', 'in.b7', 'in.b8', 'in.b9', 'in.b10', 'in.b11', 'in.b12', 'in.b13', 'in.b14', 'in.b15',
        'out.b0', 'out.b1', 'out.b2', 'out.b3', 'out.b4', 'out.b5', 'out.b6', 'out.b7', 'out.b8', 'out.b9', 'out.b10', 'out.b11', 'out.b12', 'out.b13', 'out.b14', 'out.b15',
        'out.s0', 'out.s1', 'out.s2', 'out.s3', 'out.s4', 'out.s5', 'out.s6', 'out.s7', 'out.s8', 'out.s9', 'out.s10', 'out.s11', 'out.s12', 'out.s13', 'out.s14', 'out.s15'
      ],
      unique: ['tx.h'],
      fulltext: ['out.s0', 'out.s1', 'out.s2', 'out.s3', 'out.s4', 'out.s5', 'out.s6', 'out.s7', 'out.s8', 'out.s9', 'out.s10', 'out.s11', 'out.s12', 'out.s13', 'out.s14', 'out.s15', 'in.e.a', 'out.e.a']
    },
    u: {
      keys: [
        'tx.h',
        'in.e.a', 'in.e.h', 'in.e.i', 'in.i',
        'out.e.a', 'out.e.i', 'out.e.v', 'out.i',
        'in.b0', 'in.b1', 'in.b2', 'in.b3', 'in.b4', 'in.b5', 'in.b6', 'in.b7', 'in.b8', 'in.b9', 'in.b10', 'in.b11', 'in.b12', 'in.b13', 'in.b14', 'in.b15',
        'out.b0', 'out.b1', 'out.b2', 'out.b3', 'out.b4', 'out.b5', 'out.b6', 'out.b7', 'out.b8', 'out.b9', 'out.b10', 'out.b11', 'out.b12', 'out.b13', 'out.b14', 'out.b15',
        'out.s0', 'out.s1', 'out.s2', 'out.s3', 'out.s4', 'out.s5', 'out.s6', 'out.s7', 'out.s8', 'out.s9', 'out.s10', 'out.s11', 'out.s12', 'out.s13', 'out.s14', 'out.s15'
      ],
      unique: ['tx.h'],
      fulltext: ['out.s0', 'out.s1', 'out.s2', 'out.s3', 'out.s4', 'out.s5', 'out.s6', 'out.s7', 'out.s8', 'out.s9', 'out.s10', 'out.s11', 'out.s12', 'out.s13', 'out.s14', 'out.s15', 'in.e.a', 'out.e.a']
    }
  },
  onmempool: async function(m) {
    console.log("## onmempool", m.input.tx)
    try {
      await m.state.create({ name: "u", data: m.input })
      // only publish small pushdata
      // otherwise memory issues
      let ev = Object.assign({}, m.input)
      ev.out = compact(ev.out)
      //m.output.publish({ name: "u", data: m.input })
      m.output.publish({ name: "u", data: ev })
    } catch (e) {
      console.log("Error: ", e)
    }
  },
  onblock: async function(m) {
    console.log("## onblock", "Block Size: ", m.input.block.items.length, "Mempool Size: ", m.input.mempool.items.size)
    await m.state.create({
      name: "c", data: m.input.block.items,
    }).catch(function(e) {
      if (e.code != 11000) {
        console.log("# Error", e, m.input, m.clock.bitcoin.now, m.clock.self.now)
        process.exit()
      }
    })
    await m.state.delete({ name: "u", filter: { find: {} } })
    await m.state.create({
      name: "u", data: m.input.mempool.items,
    }).catch(function(e) {
      if (e.code != 11000) {
        console.log("# Error", e, m.input, m.clock.bitcoin.now, m.clock.self.now)
        process.exit()
      }
    })
    console.log("bitcoin clock", m.clock.bitcoin.now)
    console.log("self clock", m.clock.self.now)
    if (m.clock.bitcoin.now <= m.clock.self.now+1) {
      console.log("Trigger socket")
      m.output.publish({name: "c", data: m.input.block.info})
    } else {
      console.log("Initial crawl not finished yet. Don't trigger socket")
    }
  },
  onrestart: async function(m) {
    console.log("## onrestart", m.clock.bitcoin.now, m.clock.self.now)
    await m.state.delete({
      name: 'c',
      filter: {
        find: { "blk.i": { $gt: m.clock.self.now } }
      }
    }).catch(function(e) {
      if (e.code !== 11000) {
        console.log('## ERR ', e, m.clock.bitcoin.now, m.clock.self.now)
        process.exit()
      }
    })
    await m.state.delete({ name: "u", filter: { find: {} } })
  }
}
```