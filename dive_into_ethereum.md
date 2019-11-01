# 深入理解以太坊

![-w1544](https://i.loli.net/2019/10/31/cFGSt3vICn1e9Zr.jpg)

## 目标

- 理解以太坊的底层运行机制
    - 本文主要关注为什么这么做、基本原理、实现效果，2/8法则，花 20% 的精力掌握 80% 的内容
    - 细节实现会留 reference，如果想研究，建议用以下方式：1. 走一遍 demo，搞清楚所有的细节，例如 EVM 合约的执行流程；2. 自己实现一遍，比如 MPT、EVM，甚至整个以太坊客户端。
- 概览以太坊生态全景
    - 建立以太坊/区块链的整体认知
        - 类比其它项目，看项目的方法论
    - 寻找兴趣方向，针对性的研究
    - 发散思维，发现机会
		
## 当你发出一笔交易后，发生了什么？

发送交易的 API
- <https://github.com/ethereum/wiki/wiki/JSON-RPC#eth_sendtransaction>
- <https://github.com/ethereum/wiki/wiki/JavaScript-API#web3ethsendrawtransaction>

```js
var Tx = require('ethereumjs-tx');
var privateKey = new Buffer('e331b6d69882b4cb4ea581d88e0b604039a3de5967688d3dcffdd2270c0fd109', 'hex')
var rawTx = {
 nonce: '0x00',
 gasPrice: '0x09184e72a000', 
 gasLimit: '0x2710',
 to: '0x0000000000000000000000000000000000000000', 
 value: '0x00', 
 data: '0x7f7465737432000000000000000000000000000000000000000000000000000000600057'
}
var tx = new Tx(rawTx);
tx.sign(privateKey);
var serializedTx = tx.serialize();
//console.log(serializedTx.toString('hex'));
//f889808609184e72a00082271094000000000000000000000000000000000000000080a47f74657374320000000000000000000000000000000000000000000000000000006000571ca08a8bbf888cfa37bbf0bb965423625641fc956967b81d12e23709cead01446075a01ce999b56a8a88504be365442ea61239198e23d1fce7d00fcfc5cd3b44b7215f
web3.eth.sendRawTransaction('0x' + serializedTx.toString('hex'), function(err, hash) {
 if (!err)
   console.log(hash); // "0x7f9fade1c0d57a7af66ab4ead79fade1c0d57a7af66ab4ead7c2c2eb7b11a91385"
});
```

交易字段说明：
- from：谁发的
- to：发给谁。如果是合约地址，则为调用合约；如果为空，则为部署合约。
- gas: 为了解决停机问题和收取手续费引入的概念。调用合约节点需要运行计算，gas 用来衡量计算花费，比如你的合约是投票，做简单的 +1，花费会比让合约计算 100！小得多。由于每一步计算都会消耗 gas，因此设定这个数值，可以避免合约逻辑里有死循环，gas 消耗完就当做执行失败，返回结果。
- gasPrice：gas 计费单位，按 eth 支付
- value：随交易发多少个 eth，比如合约逻辑是充值，设置这个值，在发交易的同时，给合约转 eth
- data：调用合约的数据，rlp 编码，后面讲 EVM 的时候会有再详细的解释
- nonce：交易唯一标识，防止[重放攻击](https://www.binance.vision/zh/security/what-is-a-replay-attack)

### web service 接收请求

jsonrpc via http example

```
curl -X POST --data '{"jsonrpc":"2.0","method":"eth_sendTransaction","params":[{see above}],"id":1}'
```

- 接口交互协议
	- [jsonrpc][jsonrpc-spec]
- 通道
	- http
	- ws
	- ipc
	
类比思考：接口交互的一些其他方案
- GraphQL：以太坊 geth 客户端已经支持
- grpc
- thrift

### 验证交易合法性

- 交易结构验证
- 去重
- 签名


### 放入交易池

### p2p 广播交易

![](https://i.loli.net/2019/10/31/ZPSp5OdjsHBiELn.jpg)

将交易广播给全网其它节点，上图显示了广播流程，每个节点够会连接若干个节点，广播时将交易广播给附近的节点，附近的节点重复此操作，直至传播到全网。

类比思考: bt 协议
			
### 矿工挖矿（出块）

#### 打包交易（定序）

从交易池中取出交易，排序后构造一个 tx list

#### 执行交易

本质是状态转移，但是显式的保留了世界状态，以 state_root 呈现。

![-w1236](https://i.loli.net/2019/10/31/2q5RSJGIAmtEYhN.jpg)

btc 是 UTXO 模型，交易本质是记账，比如 A 给 B 3 个 BTC，花掉的时候拿这行记录去花，本身并不算总账。

![-w681](https://i.loli.net/2019/10/31/ZLuqP65A1Mrp2By.jpg)

以太坊的的状态转移是从一个世界状态，在执行交易后变为一个新的世界状态。

上面是一个只支持转账的简单模型，以太坊的世界状态是任意的数据构成的状态，交易可以执行任意的指令

![-w1056](https://i.loli.net/2019/10/31/2f7BgbCU164nYEv.jpg)


- 状态：world state
    - 需要的特性
        - 历史可回溯
        - 需要做的变更少
    - 类比
        - 文档管理：毕业论文最终不改版
        - git
- 如何转移：EVM
    - 确定性
        - 不能有随机数
        - 不能访问外部世界状态，因为不可控，比如网络可能连不上
    - 图灵完备

#### 构造区块

##### 一些密码学概念

- 哈希
    - 从任意数据 data 算出一个定长数据 hash
    - data 不同 hash 一定不同
        - 碰撞概率低
    - 从 hash 无法反推 data
    - 使用场景：盲拍、验证数据一致
- 公钥、私钥
    - 要点
        - 私钥可以推导出公钥，公钥无法推导出私钥
        - 私钥加密的信息公钥可以解开，公钥加密的东西私钥可以解开
    - 使用场景
        - 证明你是你
        - 加密信息传输
- 地址
    - 公钥经过哈希再编码得到的结果
    - 防止公钥暴露，被碰撞出私钥
- 签名
    - 用私钥对一段数据（的哈希）签名得到的数据
    - 证明这个得到了你的授权
        - 比特世界的按指纹/签名

##### 区块结构

![block](https://i.stack.imgur.com/afWDt.jpg)

![-w574](https://i.loli.net/2019/10/31/L4wOXeJZm6vjFV2.jpg)

![-w1211](https://i.loli.net/2019/10/31/J5v6ufoZNhXtrVY.jpg)



##### MPT

![](https://i.loli.net/2019/10/31/n5vHBhgoFcJb1VW.jpg)


参考
- [Merkle Patricia Tree 详解](https://ethfans.org/toya/articles/588)
- 实现
    - [python](https://github.com/ethereum/py-trie)
    - [rust](https://github.com/cryptape/cita-trie)
    - [go](https://github.com/ethereum/go-ethereum/tree/master/trie)

    
##### 算哈希，满足难度要求

通过引入工作量证明，保证大家记的是同一本账
- 通过 block + chain 保证不可篡改
- 矿工可以获得报酬（出块奖励+手续费）
- 博弈论：伪造一个更长的链需要付出算力，而这个算力用来挖矿是收益更高的

#### p2p 广播区块
			
#### 收到新的区块，执行验证

最长链原则

#### 更新本地状态

## EVM

虚拟机是计算机科学的一个经典概念：
- 硬件模拟
    - x86 虚拟机
    - risc-v 虚拟机
- 语言解释
    - jvm：java 虚拟机
    - lua 虚拟机

EVM 的特点：
- 图灵完备
- 栈式虚拟机

想要深入理解可以直接看这个：<https://takenobu-hs.github.io/downloads/ethereum_evm_illustrated.pdf>。
有大量的图，清晰易懂。

EVM 的高级编程语言目前主要有 solidity 和 vyper 两种。

solidity 编程和其它高级语言对比：

```solidity
pragma solidity >=0.4.0 <0.7.0;

contract SimpleStorage {
    uint storedData;

    function set(uint x) public {
        storedData = x;
    }

    function get() public view returns (uint) {
        return storedData;
    }
}
```

```typescript
class SimpleStorage {
    storedData: number;
    constructor(x: number = 0) {
        this.storedData = x;
    }
    set(x: number) {
        this.storedData = x;
    }
    get(): number {
        return this.storedData;
    }
}

let s = new SimpleStorage();
console.log(s.get());   // print 0
s.set(1);
console.log(s.get());   // print 1
```

在 solidity 中，部署合约相当于从 class 初始化一个实例（instance），调用合约相当于调用实例方法。

solidity 和 vyper 都会通过编译变成 evm 字节码，被 evm 解释执行。

![-w1155](https://i.loli.net/2019/10/31/XbWfAHI47lorcYF.jpg)

详细的 evm 执行流程可以参考 [evm guide][evm-guide]，模拟整个字节码指令的运行过程。

## future

- 以太坊 2.0
    - sharding
    - POW -> POS
- 扩展
    - plasma
      - []
    - layer 2
    - state channel

## 问题

- 共识
    - 慢
    - 非确定性共识
- 虚拟机
    - 非通用
        - 编程语言限定
    - 安全性
- 状态爆炸

## 生态和研究方向

- 应用
    - DeFi
        - DEX
        - 借贷
        - 稳定币
          - [稳定数据货币手册][coin-handbook]，从货币银行学的角度阐述稳定币的一些观点，很有意思，值得一看
    - game
- 共识
- 开发者工具
    - remix
    - truffle
    - openzepplin
- 安全
    - 合约审计
- 密码学
    - 零知识证明
- 其它项目
    - pokadot/substrate
        - sharding
        - wasm
    - cosmos
        - 跨链
        - cosmos-sdk
    - nervos ckb
        - risc-v 虚拟机
        - cell 模型
		
## 思维方式

- 全局视野，认知水平
- 寻找问题，发现机会
- 借鉴历史
    - 计算机科学
    - 互联网

## 参考

- [以太坊资源列表][eth-resources]，强烈推荐
- [evm 解释][ethereum_evm_illustrated.pdf]，强烈推荐
- [ethresearsh][ethresearsh]，以太坊的一些前沿研究讨论
- [master ethereum][eth-book]
- [虚拟货币市值排行榜][coin-market]
- [jsonrpc-spec规范][jsonrpc-spec]
- [awesome evm][awesome-evm]
- [evm-guide][evm-guide]
- [稳定数字货币手册][coin-handbook]

[eth-resources]: https://docs.ethhub.io/
[eth-book]: https://github.com/ethereumbook/ethereumbook
[coin-market]: https://coinmarketcap.com/zh/all/views/all/
[jsonrpc-spec]: https://www.jsonrpc.org/specification
[ethereum_evm_illustrated.pdf]: https://takenobu-hs.github.io/downloads/ethereum_evm_illustrated.pdf
[evm-guide]: https://github.com/CoinCulture/evm-tools/blob/master/analysis/guide.md
[awesome-evm]: https://github.com/ethereum/wiki/wiki/Ethereum-Virtual-Machine-(EVM)-Awesome-List
[coin-handbook]: https://wisburg.com/articles/180507
[ethresearsh]: https://ethresear.ch/