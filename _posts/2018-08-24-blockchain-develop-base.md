---
layout:     post
title:      "区块链技术开发入门"
subtitle:   "区块链技术开发相关笔记"
date:       2018-08-24
author:     "jiangydev"
header-img: "img/post-bg-blockchain.jpg"
header-mask: 0.3
catalog:    true
tags:
    - BlockChain
    - Ethereum
    - Solidity
    - Contract
---

[TOC]

## 一、区块链介绍

- 参考资料
  - [以太坊白皮书 - 一个下一代智能合约和分布式应用平台](https://github.com/ethereum/wiki/wiki/White-Paper)
  - [DAPPs 平台](https://www.stateofthedapps.com)



## 二、客户端安装及运行

### 1 下载 Ethereum 浏览器 和 钱包

- 最新版下载地址：[https://github.com/ethereum/mist/releases](https://github.com/ethereum/mist/releases)
- 要下载两个软件包
  - Ethereum-Wallet：一个只绑定了以太坊钱包应用的Mist浏览器，因为它只绑定了这一个应用：以太坊钱包。所以被 称作为“Ethereum Wallet”。
  - Mist：一个去中心化的分散的web3.0应用的浏览器。

> 注意点：
>
> Mist = Ethereum Wallet + Web3 浏览器

### 2 区块存储路径

- Mist 默认路径：`C:\Users\<替换成计算机名>\AppData\Roaming\Ethereum`

- 更换默认的区块存储路径

  ```shell
  $ mklink /j "C:\Users\<替换成计算机名>\AppData\Roaming\Ethereum" "D:\Ethereum"
  ```

  如果有一个区块链数据已经同步完整的以太坊节点，那么可以导出节点数据，然后导入一个新的以太坊节点。

  ```shell
  $ geth -testnet export filename
  $ geth -testnet import filename
  ```


### 3 设置系统环境变量

将 `C:\Users\HS\AppData\Roaming\Mist\binaries\Geth\unpacked\` 设置到环境变量 Path 中，便于在任意目录的 CMD 中可以使用 `geth` 命令。

> 小提示：
>
> 设置完环境变量后，若已有打开的 CMD 窗口，需要重新打开，geth 才生效。



### 4 以太坊账户创建与管理

#### 4.1 外部账号和合约账号介绍

- 账号在以太坊中扮演着核心的角色。以太坊共有`两种账号类型`：外部账号(EOA)和合约账号。在这里我们先重点关注外部账号，简称账号。合约账号简称合约。外部账户和合约账户都是账户的通用概念，这些账户其实都是状态对象。外部账户的余额就是外部账户的一个状态对象，合约账户的状态除了有余额还有合约存储。所有账户的状态都是以太坊网络的状态，以太坊网络的状态随着每一个区块的更新而变化。用户通过交易和以太坊区块链进行交互，在这个过程中，账户起着至关重要，不可缺少的作用。
- 如果限制以太坊只有外部账号，并且限制它们只能交易，那么我们就是只是做了一个山寨币，而且是只能交易以太币（ ether）。
- 账号代表了使用者的一个对外的身份，用户使用公钥去签名一个交易，然后以太坊虚拟机就可以安全的 校验这交易发起者的身份。
- 秘钥文件
  - 每一个账号都有一对密钥，一个私钥和一个公钥。账号和地址是一一对应的。账号被来自密钥的最后20个字节的地址索引着的。每一个私钥/地址对都被编码进一个密钥文件。密钥文件是一个json格式的文本文件 ，可以用任何的文本工具打开和编辑它。密钥文件的重要组成部分—你账号的私钥，是使用你在创建账号时输入的密码来加密保护的。密钥文件存储在你的以太坊客户端的`keystore`子目录中。确保定期备份你的key 文件！
  - 创建一个密钥等同于创建一个账号
    - 不需要告诉别人你创建了一个账号
    - 不需要和区块链进行同步
    - 不需要运行一个客户端
    - 甚至不需要联网
- 在创建账户之后，要注意：一定要记住你的密码和备份你的密钥文件！！！因为发送交易，甚至发送以太币都是必须要同时使用到密码和密钥文件的。
- 丢失了密钥文件或密码，那账户中的所有的以太币也就全部都丢失了。没有密码是绝对无法访问你的账户的。并且以太坊没有“忘记密码？找回密码”这一功能选项。    



#### 4.2 账号创建

- 创建主网络账号

  ```shell
  $ geth account new
  ```

- 创建测试网络账号

  ```shell
  # Ropsten network:
  $ geth --testnet account new
  # Rinkeby network:
  $ geth --rinkeby account new
  ```



#### 4.3 账号备份

- 以太坊网络没有所谓“找回账户密码”的功能。所以一旦丢失账户或忘记密码，将永远的失去你的账户，和账户中的所有以太币！
- 备份账户
  - C:\Users\你的账户\AppData\Roaming\Ethereum\ 中的“keystore”目录整个备份一遍
  - 并且同时备份好你的账户密码
- 下次，你只需要把你备份的“keystore”目录中的文件，复制到你的新的客户端对应的目录，就可以在你的新客户端上操作你的账户了。



### 5 多重签名

- 你可以使用以太坊钱包的多重签名来保护你账户中的余额。使用多重签名的好处是，当要从账户中提取 较大额度的金额时，需要多个账户的共同认证才能成功提取。在创建一个多重签名的钱包前，你首先至 少需要创建2个账户。
- 在以太坊钱包中创建账号很容易，只需要在 Accounts 选项卡，点击 ‘Add Account’ 。创建至少2个 账号。
- 你现在需要向你的主账号添加不少于0.02的以太币（用于创建多重签名钱包的账号），这是创建多重签 名钱包合约的交易费用。另外还需要至少1以太币，因为当前mist需要有足够的 ‘gas’ 来确保多重签 名钱包合约能够正确的执行交易。所以一开始总共需要1.02的以太币。    



### 6 以太币

- Gas 成本 ：一个计算步骤的成本，即一个计算消耗多少个Gas
- Gas 价格：以以太币计的价值
- Gas 限额：一个交易中允许的Gas消耗的上线
- Gas 费用：一个交易中所消耗的Gas总数，即这个交易成本。Gas费用反应网络中的计算负荷、交易量，以及区块的大小。Gas费用是支付给矿工的。



### 7 挖矿

- 挖矿需要下载全节点（Full）

- 挖矿命令

  ```shell
  # 在 Mist 客户端打开的情况下执行下面的指令，否则会连接不上
  $ geth attach ipc:\\.\pipe\geth.ipc
  # 启动挖矿（非全节点，会提示未定义 miner）
  $ miner.start()
  ```




## 三、以太坊网络

> 相关网站：
>
> 1. [ethstats.net](ethstats.net)：以太网网络的实时的统计数据信息；这网站上包含了许多重要的数据，如当前区块，交易，gas价格等。这页面上展示的节点只是实际网络中的节点的一部分。项目地址：[https://github.com/cubedro/eth-netstats](https://github.com/cubedro/eth-netstats)
> 2. ethernodes.com：统计了当前和历史上的有关节点的数量，包括了主网和测试网络。
> 3. [etherchain.org](https://etherchain.org/)：实时区块链统计信息。



### 1 以太坊网络类型

#### 1.1 介绍

- **公有链**：世界上任何一个人都可以参与的区块链。用户可以查看，可以发送交易，也可以参与保持数据一致性的运算等。
- **私有链**：完全的私有链是指写权限是由一个人或一个单个组织控制的链。私有链的读权限是可以公开的或者是有限度的在一定范围公开的。比如私有链可以用在数据库的管理，公司内部的管理等。
- **联盟链**：联盟链是指，数据一致性的运算被预先设定好的几个节点共同控制的链。比如，有15家银行组成了一个财团链，在这个链上的每一个节点的每一次的操作都需要10个节点的共同签名才能被验证。这区快链上的读权限可能是公开的，也有可能是部分公开的。

#### 1.2 主网络

一条区块链由一个创世区块开始，也就是说，一个创世区块可以创造和代表一条区块链。如果我们给钱包客户端设定不同的创世区块，它就将工作在不同的区块链上。

**工作在同一条区块链上的全部节点，我们称之为一个网络。**

绝大多数人在使用的网络被称为**主网络(Mainnet)**，用户在其上交易、构建智能合约，矿工在其上挖矿。由于使用的人数众多，主网络的鲁棒性很强，能够对抗攻击，区块链也不易被篡改，因此主网络是具有功能的，其上的以太币是有价值的。

通常一种区块链只有一个主网络，比如比特币，莱特币，以太坊，都只有一个主网络。主网络之外可以有若干个测试网络。



#### 1.3 测试网络

以太坊可以搭建私有的测试网络，不过由于以太坊是一个去中心化的平台，需要较多节点共同运作才能得到理想的测试效果，因此并不推荐自行搭建测试网络。

以太坊公开的测试网络共有4个，目前仍在运行的有3个。每个网络都有自己的创世区块和名字，按开始运行时间的早晚，依次为：

- **Morden(已退役)**

Morden是以太坊官方提供的测试网络，自2015年7月开始运行。到2016年11月时，由于难度炸弹已经严重影响出块速度，不得不退役，重新开启一条新的区块链。Morden的共识机制为PoW。

- **Ropsten(**[区块链浏览器](https://ropsten.etherscan.io/)**)**

Ropsten也是以太坊官方提供的测试网络，是为了解决Morden难度炸弹问题而重新启动的一条区块链，目前仍在运行，共识机制为PoW。测试网络上的以太币并无实际价值，因此Ropsten的挖矿难度很低，目前在755M左右，仅仅只有主网络的0.07%。这样低的难度一方面使一台普通笔记本电脑的CPU也可以挖出区块，获得测试网络上的以太币，方便开发人员测试软件，但是却不能阻止攻击。

PoW共识机制要求有足够强大的算力保证没有人可以随意生成区块，这种共识机制只有在具有实际价值的主网络中才会有效。测试网络上的以太币没有价值，也就不会有强大的算力投入来维护测试网络的安全，这就导致了测试网络的挖矿难度很低，即使几块普通的显卡，也足以进行一次51%攻击，或者用垃圾交易阻塞区块链，攻击的成本及其低廉。

2017年2月，Ropsten便遭到了一次利用测试网络的低难度进行的攻击，攻击者发送了千万级的垃圾交易，并逐渐把区块Gas上限从正常的4,700,000提高到了90,000,000,000，在一段时间内，影响了测试网络的运行。攻击者发动这些攻击，并不能获得利益，仅仅是为了测试、炫耀、或者单纯觉得好玩儿。

- **Kovan(**[区块链浏览器](https://kovan.etherscan.io/)**)**

为了解决测试网络中PoW共识机制的问题，以太坊钱包Parity的开发团队发起了一个新的测试网络Kovan。Kovan使用了权威证明(Proof-of-Authority)的共识机制，简称PoA。

PoW是用工作量来获得生成区块的权利，必须完成一定次数的计算后，发现一个满足条件的谜题答案，才能够生成有效的区块。

PoA是由若干个权威节点来生成区块，其他节点无权生成，这样也就不再需要挖矿。由于测试网络上的以太币无价值，权威节点仅仅是用来防止区块被随意生成，造成测试网络拥堵，完全是义务劳动，不存在作恶的动机，因此这种机制在测试网络上是可行的。

Kovan与主网络使用不同的共识机制，影响的仅仅是谁有权来生成区块，以及验证区块是否有效的方式，权威节点可以根据开发人员的申请生成以太币，并不影响开发者测试智能合约和其他功能。

Kovan目前仍在运行，但仅有Parity钱包客户端可以使用这个测试网络。

- **Rinkeby(**[区块链浏览器](https://rinkeby.etherscan.io/)**)** `开发常用`

Rinkeby也是以太坊官方提供的测试网络，使用PoA共识机制。与Kovan不同，以太坊团队提供了Rinkeby的PoA共识机制说明文档，理论上任何以太坊钱包都可以根据这个说明文档，支持Rinkeby测试网络，目前Rinkeby已经开始运行。

#### 1.4 连接测试网络及创建钱包



#### 1.5 获取测试网络上的以太币

- 地址：[https://faucet.rinkeby.io/](https://faucet.rinkeby.io/)
  - 支持 3 种包含 Ethereum Address 的社交网络 URL，分别是 Tweet, Google Plus, Facebook。
  - 具体操作说明：在上述的 3 个社交网络上发布一条含有 Ethereum Address 的动态，并复制该动态的 URL 到 [https://faucet.rinkeby.io/](https://faucet.rinkeby.io/) 上提交即可。



> 参考文档
>
> 1. [玩转以太坊(Ethereum)的测试网络](https://zhuanlan.zhihu.com/p/29010231)



### 2 查看网络状态

- 查看链接状态

```shell
$ net.listening
true
$ net.peerCount
4
```

- 查看自己的伙伴的网络信息

```shell
$ admin.peers
```

- 查看自己的网络信息

```shell
$ admin.nodeInfo
```



### 3 加快下载速度

#### 3.1 注意点

以太坊客户端一旦运行，就会自动去下载区块链数据。下载的速度跟客户端的设置，链接网路的速度，同伴的数量有关。

#### 3.2 以太坊节点分类

- Light，Download only the headers and terminate afterwards ；

- Fast，Quickly download the headers, full sync only at the chain head；

- Full；Synchronise the entire blockchain history from full blocks 

  只有下载了 Full 节点，才能 mine ETH。

#### 3.3 静态节点

- 命令配置

  ```shell
  $ admin.addPeer("enode://5757ec68b973a5ad02f27644dd459ddf5dfc4a709af46ef406e1147436115cb87923590bcf441c05f7e496f2578809560709b3af77b071d8b1dffcd681dcb44d@192.168.5.20:30303)
  ```

- 静态节点信息

  存放目录：`<datadir>/geth/static-nodes.json`

  ```json
  [
      "enode://5757ec68b973a5ad02f27644dd459ddf5dfc4a709af46ef406e1147436115cb87923590bcf441c05f7e496f2578809560709b3af77b071d8b1dffcd681dcb44d@192.168.5.20:30303",
      "enode://pubkey@ip:port" 
  ]
  ```

> 参考文档：
>
> 1. [Connecting to the network](https://github.com/ethereum/go-ethereum/wiki/Connecting-to-the-network)



### 4 构建本地私有网络

#### 4.1 构建私有网络所需内容

- 使用 geth 命令行工具构建本地私有测试网络需要指定以下参数信息
  - 自定义genesis文件
  - 自定义数据目录
  - 自定义网络ID
  - （推荐）关闭节点发现协议

- 参数释义

  - --nodiscover

    添加这个参数，确保没有人能发现你的节点。不然的话，可能有人无意中会链接到你的私有区块链。

  - --maxpeers 0

    使用maxpeers 0,如果你不希望其他人连接到您的测试链。当然，您也可以调整这个数,如果你知道有多少同伴会连接你的节点。

  - --rpc

    在你的节点上激活RPC接口。这参数在geth中默认启用。

  - --rpcapi "db,eth,net,web3"

    这个命令描述哪些接口可以通过RPC来访问，默认情况下，geth开启的是web3接口。

  - --rpcport "8080"

    将端口号设置成8000以上的任何一个你网络中打开的端口。默认是8080。

  - --rpccorsdomain http://chriseth.github.io/browser-solidity/

    设置可以连接到你的节点的url地址，以执行RPC客户端的任务。最好不要使用通配符 *，这样将允许任何url都可以链接到你的RPC实例。

  - --datadir "/home/TestChain1"

    私有链的数据目录，确保与公共以太坊链的数据目录区分开来。

  - --port "30303"

    这是“网络监听的端口”,您可以用它手动的和你的同伴相连。

  - --identity "TestnetMainNode"

    为你的节点设置一个ID。用于和你们的一系列同伴进行区分。

#### 4.2 创世区块文件

- genesis（创世）区块 `CustomGenesis.json`

```json
{
    "nonce": "0x0000000000000042",     
    "timestamp": "0x00",
    "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "extraData": "0x00",
    "gasLimit": "0x8000000",
    "difficulty": "0x400",
    "mixhash": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "coinbase": "0x3333333333333333333333333333333333333333",     
    "alloc": {
     },
     "config": {
        "chainId": 15,
        "homesteadBlock": 0,
        "eip155Block": 0,
        "eip158Block": 0
    }
}
```
- 参数释义

  - Mixhash

    一个256位的哈希值，和 Nonce 配合，一起用来证明在区块链上已经做了足够的计算量（工作证明）。Nonce 和 Mixhash 的组成，必须满足一个在黄皮书中所描述的数学条件，黄皮书 4.3.4。

  - Nonce

    一个64位的哈希值，和 Mixhash 配合，一起用来证明在区块链上已经做了足够的计算量（工作证明）

  - Difficulty

    定义挖矿的目标，可以用上一个区块的难度值和时间戳计算出来，值越高，矿工越难挖到区块

  - Alloc

    预先填入一些钱包和余额

  - Coinbase

    160位的钱包地址。在创世区块中可以被定义成任何的地址，因为当每挖到一个区块的时候，这个值会变成矿工的etherbase地址

  - Timestamp  一个unix的time()函数的输出值，时间戳

  - extraData  32字节长度，可以为私有链留下一些信息，如你的姓名等，用以证明这个私有链是你创建的

  - gasLimit   当前链，一个区块所能消耗的gas上限



#### 4.3 运行私有网络

- 初始化创世区块

  ```shell
  $ geth --identity "MyPrivateNode" --rpc --rpcport "8086" --rpccorsdomain "*" --datadir "C:\Users\HS\Desktop\test\ethtest" --port "30303" --nodiscover --rpcapi "db,eth,net,web3" --networkid 521 init /path/to/CustomGenesis.json
  ```

- 运行私有网络

  ```shell
  $ geth --identity "MyPrivateNode" --rpc --rpcport "8086" --rpccorsdomain "*" --datadir "C:\Users\HS\Desktop\test\ethtest" -- port "30303" --nodiscover --rpcapi "db,eth,net,web3" --networkid 520 console
  ```


## 四、智能合约编程入门

### 1 Solidity 介绍

- 官网： http://solidity.readthedocs.io/en/develop/

- Solidity 是一个面向合约的高级语言，其语法类似于JavaScript 。是运行在以太坊虚拟机中的代码。

- Solidity 是静态类型的编程语言，编译期间会检查其数据类型。支持继承、类和复杂的用户定义类型。

- 在线体验： https://remix.ethereum.org

  但是这平台只能撰写和编译Solidity代码，如果想真正运行代码的话，需要有一个以太坊的本地环境。



### 2 搭建开发环境

#### 2.1 构建多节点私有链网络

- 节点间会自动发现

- 使用命令，主动添加

  ```shell
  > admin.addPeer("pubkey@ip:port")
  ```


#### 2.2 在多节点私有网络中创建使用多重签名钱包

- 创建账号

![研发-多重签名钱包1](/img/in-post/blockchain/研发-多重签名钱包1.png)

- 选择“钱包合约类型”

![研发-多重签名钱包2](/img/in-post/blockchain/研发-多重签名钱包2.png)



#### 2.3 智能合约

- 创建一个智能合约

![研发-部署第一个合约1](/img/in-post/blockchain/研发-部署第一个合约1.png)

- 将官网上的案例复制到合约中并部署![研发-部署第一个合约2](/img/in-post/blockchain/研发-部署第一个合约2.png)

  ```solidity
  pragma solidity ^0.4.24;
  
  contract Coin {
      // The keyword "public" makes those variables
      // easily readable from outside.
      address public minter;
      mapping (address => uint) public balances;
  
      // Events allow light clients to react to
      // changes efficiently.
      event Sent(address from, address to, uint amount);
  
      // This is the constructor whose code is
      // run only when the contract is created.
      constructor() public {
          minter = msg.sender;
      }
  
      function mint(address receiver, uint amount) public {
          require(msg.sender == minter);
          require(amount < 1e60);
          balances[receiver] += amount;
      }
  
      function send(address receiver, uint amount) public {
          require(amount <= balances[msg.sender], "Insufficient balance.");
          balances[msg.sender] -= amount;
          balances[receiver] += amount;
          emit Sent(msg.sender, receiver, amount);
      }
  }
  ```

- 部署合约

![研发-部署第一个合约3](/img/in-post/blockchain/研发-部署第一个合约3.png)

- 挖矿使合约确认

![研发-部署第一个合约4](/img/in-post/blockchain/研发-部署第一个合约4.png)

- 合约的测试

![研发-部署第一个合约5](/img/in-post/blockchain/研发-部署第一个合约5.png)

> 小提示：
>
> 1. Balances 变量下填写 Address，可以查询合约中该地址的余额；
> 2. 一定要 `miner.start()`，使交易确认；
> 3. 合约中的钱包地址余额与外部地址余额不一样哦！要使用`payable`才会真正使用 ETH；
> 4. 在 A 节点部署的合约，B 节点要添加`观察合约`才能发现该合约。



### 3 Solidity IDE - Remix

Remix IDE 是 Mist  自带的，开发智能合约的开发环境。

打开方式：Mist 工具栏 > Develop > Open Remix IDE。

> 小提示：
>
> Remix 在线浏览器版地址：[http://remix.ethereum.org/](http://remix.ethereum.org/)

在 Remix IDE 中，编写的文件不方便找到其位置，因此需要使用`npm`安装`remixd`来指定项目的 workspace，如下。

#### 3.1 安装 Nodejs

官网：[https://nodejs.org/en/](https://nodejs.org/en/)

> 小提示：
>
> 1. 一定要下载最新稳定版 nodejs，否则会导致安装 remixd 的过程出现多个问题。
> 2. 若您已安装过 nodejs，请检查版本。

#### 3.2 安装 remixd

以管理员身份打开cmd，执行以下命令。

```shell
# 安装python2.7和依赖的组件
$ npm install --global --production windows-build-tools
$ npm install --global node-gyp
$ npm install -g remixd
```

> 小提示：
>
> 1. 要注意，remixd 要求的 python 版本介于 2.5 和 3.0 之间。

#### 3.3 指定 remix IDE workspace

- 指定 workspace

```shell
$ remixd -s C:\solidity\workspace\
```

- 连接本地 workspace

看到 localhost 说明配置成功。

![研发-Solidity IDE Remix1](/img/in-post/blockchain/研发-Solidity IDE Remix1.png)



### 4 基本变量类型

#### 4.1 Integers

`int` / `uint`: 有符号/无符号整型。关键字 `uint8` 到 `uint256` 和 `int8` 到 `int256`，声明一个 8 的倍数. `uint` 和 `int` 是 `uint256` 和 `int256` 的别名。

操作符:

- 比较符: `<=`, `<`, `==`, `!=`, `>=`, `>` (evaluate to `bool`)
- 位 操作符: `&`, `|`, `^` (bitwise exclusive or), `~` (bitwise negation)
- 算数 操作符: `+`, `-`, unary `-`, unary `+`, `*`, `/`, `%` (remainder), `**` (exponentiation), `<<` (left shift), `>>` (right shift)

Division by zero and modulus with zero throws a runtime exception.

> 小提示：
>
> 1. 采用一个合适的整数位数，可以减少 gas；



#### 4.2 Address

`address`: 拥有 20 位 (一个 Ethereum address 的大小). Address 类型也有成员，并作为基础为所有的合约提供服务。

操作符:

- `<=`, `<`, `==`, `!=`, `>=` and `>`

成员

- `balance` and `transfer`

- `send`

- `call`, `callcode`, `delegatecall` and `staticcall`

  - 这四个方法等级非常低，它们会导致 Solidity 类型安全问题，应该只作为最后的手段。
  - callcode 将在未来版本中被移除；
  - `delegatecall`的目的是使用存储在其他合约中的库代码；
  - `call`用来返回布尔值，指代调用方法终止（true）或 EVM 引起异常（false）

  ```solidity
  pragma solidity ^0.4.0;
  contract CA {
      uint public p;
      event e(address add, uint p);
      function fun(uint u1, uint u2) {
          p = u1 + u2;
          e(msg.sender);
      }
  }
  contract CB {
      uint public q;
      uint public b;
      
      function call1(address add) returns(bool) {
          b = add.call(bytes4(keccak256("fun(uint256, uint256)")), 2, 3);
          return b;
      }
      function call2(address add) returns(bool) {
          b = add.delegatecall(bytes4(keccak256("fun(uint256, uint256)")), 1, 2);
          return b;
      }
  }
  ```


#### 4.3 Array

```solidity
pragma solidity ^0.4.16;

contract C {
    function f(uint len) public pure {
        uint[] memory a = new uint[](7);
        bytes memory b = new bytes(len);
        assert(a.length == 7);
        assert(b.length == len);
        a[6] = 8;
    }
}
```



## 五、Solidity 复杂变量类型

### 1 Enums

枚举是在Solidity中创建用户定义类型的一种方法。它们可以显式转换为所有整数类型，但不允许隐式转换。显式转换在运行时检查值范围，并且失败会导致异常。

枚举至少需要一名成员。

数据表示与C中的枚举相同：选项由从0开始。 

```solidity
pragma solidity ^0.4.16;

contract test {
    enum ActionChoices { GoLeft, GoRight, GoStraight, SitStill }
    ActionChoices choice;
    ActionChoices constant defaultChoice = ActionChoices.GoStraight;

    function setGoStraight() public {
        choice = ActionChoices.GoStraight;
    }

    // Since enum types are not part of the ABI, the signature of "getChoice"
    // will automatically be changed to "getChoice() returns (uint8)"
    // for all matters external to Solidity. The integer type used is just
    // large enough to hold all enum values, i.e. if you have more than 256 values,
    // `uint16` will be used and so on.
    function getChoice() public view returns (ActionChoices) {
        return choice;
    }

    function getDefaultChoice() public pure returns (uint) {
        return uint(defaultChoice);
    }
}
```



### 2 Structs & Mapping 

```solidity
pragma solidity ^0.4.0;
contract StructDemo {
    struct Funder {
        address addr;
        uint amount;
    }
    
    mapping (uint => Funder) public funder;
    
    uint public funderId;
    
    event e(string str, address addr, uint amount);
    
    function newFunder(address addr, uint amount) returns (uint) {
        ++funderId;
        funders[funderId] = Funder(addr, amount);
        e("new funder", addr, amount);
    }
    
    function setFunder(uint order, uint amount) {
        Funder f = funders[order];
        f.amount = amount;
        e("set funder", f.addr, f.amount);
    }
}
```



### 3 Delete

将变量初始化。

```solidity
pragma solidity ^0.4.0;
contract DeleteDemo {
    uint public data = 100;
    // 数字默认 uint8
    uint[] public dataArray = [uint(1), 2, 3];
    event e1(string str, uint u);
    event e2(string str, uint[] u);
    
    function doDelete() {
        uint x = data;
        e1("before delete - x", x);
        delete x;
        e1("after delete - x", x);
        
        e1("before delete - data", data);
        delete data;
        e1("after delete - data", data);
        
        uint[] y = dataArray;
        e2("after delete - y", y);
        delete y;
        e1("after delete - y", y);
    }
}
```



### 4 区块和交易属性

- `block.blockhash(uint blockNumber) returns (bytes32)`: hash of the given block - only works for 256 most recent, excluding current, blocks - deprecated in version 0.4.22 and replaced by `blockhash(uint blockNumber)`.
- `block.coinbase` (`address`): current block miner’s address
- `block.difficulty` (`uint`): current block difficulty
- `block.gaslimit` (`uint`): current block gaslimit
- `block.number` (`uint`): current block number
- `block.timestamp` (`uint`): current block timestamp as seconds since unix epoch
- `gasleft() returns (uint256)`: remaining gas
- `msg.data` (`bytes`): complete calldata
- `msg.gas` (`uint`): remaining gas - deprecated in version 0.4.21 and to be replaced by `gasleft()`
- `msg.sender` (`address`): sender of the message (current call)
- `msg.sig` (`bytes4`): first four bytes of the calldata (i.e. function identifier)
- `msg.value` (`uint`): number of wei sent with the message
- `now` (`uint`): current block timestamp (alias for `block.timestamp`)
- `tx.gasprice` (`uint`): gas price of the transaction
- `tx.origin` (`address`): sender of the transaction (full call chain)



## 六、Solidity 方法

### 1 代码中创建合约

```solidity
pragma solidity ^0.4.0;

contract C {
    uint private data;

    function f(uint a) private pure returns(uint b) { return a + 1; }
    function setData(uint a) public { data = a; }
    function getData() public view returns(uint) { return data; }
    function compute(uint a, uint b) internal pure returns (uint) { return a + b; }
}

// This will not compile
contract D {
    function readData() public {
        C c = new C();
        uint local = c.f(7); // error: member `f` is not visible
        c.setData(3);
        local = c.getData();
        local = c.compute(3, 5); // error: member `compute` is not visible
    }
}
// 继承
contract E is C {
    function g() public {
        C c = new C();
        uint val = compute(3, 5); // access to internal member (from derived to parent contract)
    }
}
```



### 2 方法

Functions 必须指出 `external`, `public`, `internal` or `private`。为了声明变量，不能使用 `external`。

- `external`:

  External functions 是合约接口的一部分, 意味着它们能从其他合约并通过交易被调用。一个 external function `f` 不能被内部调用 (i.e. `f()` 无效, 但是 `this.f() `可以)。当收到大数据数组时，External functions 有时更有效率。

- `public`:

  Public functions 是合约接口的一部分，也能被内部调用或通过 messages。为 public 声明的变量自动生成一个 getter 方法.

- `internal`:

  仅能被内部访问的方法和声明变量(i.e. 从当前合约或源于它的合约), 不需要使用 `this`.

- `private`:

  私有方法和声明的变量，仅对当前定义的合约可见。



### 3 匿名方法

- 一个合约可以有一个匿名函数。此函数不能有参数，也不能任何返回值，当我们企图去执行一个合约上没有的函数时，那么合约就会执行这匿名函数。
- 此外，当合约在只收到以太币的时候，也会调用这个匿名函数，而且一般情况下，只会当你接收到以太币后，想要执行一些操作的话，尽可能把要的操作写到这匿名函数里，因为这样做成本便宜。



### 4 修改器

- 修改器可以用来改变方法的行为，比如在方法正式执行前，检查方法是否满足条件，如果满足条件，则执行方法，不满足则可以抛出异常。

```solidity
pragma solidity ^0.4.0

contract modifierDemo {
    address public owner;
    uint public u;
    
    function modifierDemo() {
        owner = msg.sender;
    }
    
    modifier onlyOwner {
        if (msg.sender != owner) {
            throw;
        } else {
            _;
        }
    }
    
    function set(uint _u) onlyOwner {
        u = _u;
    }
}
```





## 七、Solidity 继承和事件

### 1 继承

- 使用`is`继承一个合约，子类可以访问父类的除 private 限制的属性和方法；
- 包括 internal 方法和变量，注意：不可以使用`this`来访问；
- 构造函数参数传递；

### 2 自毁

- selfdestruct(address recipient);
- 销毁当前合约，并且把当前合约的余额放大指定地址。

### 3 事件

- 打印出信息



## 八、智能合约 - “投票”

```solidity
pragma solidity ^0.4.24;
contract voteDemo {
    
    //定义投票人的结构
    struct Voter {
        uint weight; 			//投票人的权重
        bool voted ;			// 是否已经投票
        address delegate; 		//委托代理人投票
        uint vote; 				// 投票主题的序号
    }
    
    //定义投票主题的结构
    struct Posposal {
        bytes8 name;  			//投票主题的名字
        uint voteCount; 		//主题得到的票数
    }

    //定义投票的发起者
    address public chairperson;

    //所有人的投票人
    mapping(address=>Voter) public voters;

    //具体的投票主题
    Posposal[] public posposals;

    //构造函数
    constructor(bytes8[] peposposalName) public {
        //初始化投票的发起人，就是当前合约的部署者
        chairperson = msg.sender;
        //给发起人投票权
        voters[chairperson].weight = 1;
        
        //初始化投票的主题
        for(uint i = 0; i < peposposalName.length; i++) {
            posposals.push(Posposal({name:peposposalName[i],voteCount:0}));
        }
    }
    
    //添加投票者
    function  giveRightToVote(address _voter) public {
        //只有投票的发起人才能够添加投票者
        //添加的投票者不能是已经参加过投票了
        assert(msg.sender != chairperson || voters[_voter].voted);
        //赋予合格的投票者投票权重
        voters[_voter].weight = 1;
    }
    
    //将自己的票委托给to来投
    function delegate(address to) public {
        //检查当前交易的发起者是不是已经投过票了
        Voter storage sender = voters[msg.sender];
        //如果是的话，则程序终止
        assert(sender.voted);
        
        //检查委托人是不是也委托人其他人来投票
        while(voters[to].delegate != address(0)){
            //如果是的话，则把委托人设置成委托人的委托人
            to = voters[to].delegate;
            //如果发现最终的委托人是自己，则终止程序
            assert(to == msg.sender);
        }
        
        //交易的发起者不能再投票了
        sender.voted = true;
        //设置交易的发起者的投票代理人
        sender.delegate = to;
        //找到代理人
        Voter storage delegate = voters[to];
        //检测代理人是否已经投票
        if(delegate.voted){
            //如果是：则把票直接投给代理人投的那个主题
            posposals[delegate.vote].voteCount +=sender.weight;
        }else{
            //如果不是：则把投票的权重给予代理人
            delegate.weight +=sender.weight;
        }
    }
    //投票
    function vote(uint pid) public {
        //找到投票者
        Voter storage sender  = voters[msg.sender];
        //检查是不是已经投过票
        assert(sender.voted);
        //如果否：则投票
        sender.voted = true;  //设置当前用户已投票
        sender.vote = pid;    //设置当前用户的投的主题的编号
        posposals[pid].voteCount += sender.weight;  //把当前用户的投票权重给予对应的主题
        
    }
    
    //计算票数最多的主题
    function winid() constant public returns(uint winningid){
        //声明一个临时变量，用来比大小
        uint winningCount = 0;
        //遍历主题，找到投票数最大的主题
        for(uint i = 0; i < posposals.length; i++ ){
            if(posposals[i].voteCount > winningCount ){
                winningCount = posposals[i].voteCount;
                winningid = i;
            }
        }
    }
    
    function winname() constant public returns(bytes8 winnername){
        winnername = posposals[winid()].name;
    }   
}
```



## 九、智能合约 - 代币

```solidity
pragma solidity ^0.4.0;

//创建一个基础合约，有些操作只能是当前合约的创建者才能操作
contract owned{
    //声明一个用来接收合约创建者的状态变量
    address public owner;
    //构造函数，把当前交易的发送者（也就是合约的创建者）赋予owner变量
    function owned(){
        owner = msg.sender;
    }
    
    //声明一个修改器，用于有些方法只有合约的创建者才能操作
    modifier onlyOwner{
        if(msg.sender != owner){
            revert();
        }else{
            _;
        }
    }
    //把该合约的拥有者转给其他人
    function transferOwner(address newOwner) onlyOwner{
        owner = newOwner;
    }
}

contract tokenDemo3 is owned {
    string public name ;//代币名字
    string public symbol; //代币符号
    uint8 public decimals = 0; //代币小数位
    uint public totalSupply; //代币总量
    
    uint public sellPrice = 1 ether ; //设置代币的卖的价格等于一个以太币
    uint public buyPrice = 1 ether ;//设置代币的买的价格等于一个以太币
    
    //用一个映射类型的变量，来记录所有账户的代币的余额
    mapping(address => uint) public balanceOf;
    //用一个映射类型的变量，来记录被冻结的账户
    mapping(address=>bool) public frozenAccount;
    
    
    event e(string _str);
    //构造函数，初始化代币的变量和初始代币总量
    function tokenDemo3(uint initialSupply,string _name , string _symbol, uint8 _decimals,address centralMinter) payable{
        //手动指定代币的拥有者，如果不填，则默认为合约的部署者
        if(centralMinter !=0){
            owner = centralMinter;
        }
        
        balanceOf[owner] = initialSupply;
        name = _name;
        symbol = _symbol;
        decimals = _decimals;
        totalSupply = initialSupply;
    }
    
    //发行代币，向指定的目标账户添加代币
    function mintToken(address target,uint mintedAmount) onlyOwner{
        //判断目标账户是否存在
        if(target != 0){
            //设置目标账户相应的代币余额
            balanceOf[target] = mintedAmount;
            //增加总量
            totalSupply +=mintedAmount;
        }else{
            revert();
        }
    }
    //实现账户的冻结和解冻
    function freezeAccount(address target,bool _bool) onlyOwner{
        if(target != 0){
            frozenAccount[target] = _bool;
        }
    }
    //实现账户间，代币的转移
    function transfer(address _to, uint _value) {
        //检测交易的发起者的账户是不是被冻结了
        if(frozenAccount[msg.sender]){
            revert();
        }
        //检测交易发起者的账户的代币余额是否足够
        if(balanceOf[msg.sender] < _value){
                revert();
        }
        //检测溢出
        if((balanceOf[_to] + _value) <balanceOf[_to] ){
                    revert();
        }
        
        //实现代币转移
        balanceOf[msg.sender] -=_value;
        balanceOf[_to] +=_value;
    }
    
    //设置代币的买卖价格
    function setPrice(uint newSellPrice,uint newBuyPrice) onlyOwner{
        sellPrice = newSellPrice;
        buyPrice = newBuyPrice;
    }
    //实现代币的卖操作
    function sell(uint amount) returns(uint revenue){
        //检测交易的发起者的账户是不是被冻结了
        if(frozenAccount[msg.sender]){
            revert();
        }
        //检测交易发起者的账户的代币余额是否足够
        if(balanceOf[msg.sender] < amount){
            revert();
        }
        //把相应数量的代币给合约的拥有者
        balanceOf[owner] +=amount ;
        //卖家的账户减去相应的余额
        balanceOf[msg.sender] -=amount;
        //计算对应的以太币的价值
        revenue = amount * sellPrice;
        //向卖家的账户发送对应数量的以太币
        if(msg.sender.send(revenue)){
            return revenue;
        }else{
            //如果以太币发送失败，则终止程序，并且恢复状态变量
            revert();
        }
    }
    
    //实现买操作
    function buy() payable returns(uint amount) {
        //检测买家是不是大于0 
        if(buyPrice <= 0){
            //如果不是，则终止
            revert();
        }
        //根据用户发送的以太币的数量和代币的买价，计算出代币的数量
        amount = msg.value / buyPrice;
        //检测合约的拥有者是否有足够多的代币
        if(balanceOf[owner] < amount){
            revert();
        }
        //向合约的拥有者转移以太币
        if(!owner.send(msg.value)){
            //如果失败，则终止
            revert();
        }
        //从拥有者的账户上减去相应的代币
        balanceOf[owner] -=amount ;
        //买家的账户增加相应的余额
        balanceOf[msg.sender] +=amount;
        
        return amount;
    }
}
```



## 十、Web3 接口和在线钱包

### 1 Web3 接口

- [Ethereum JavaScript API](https://github.com/ethereum/wiki/wiki/JavaScript-API)
- 下载 API 项目，使用`npm install`安装

### 2 在线钱包开发

- [以太坊开发框架 - Truffle](https://truffleframework.com/docs/truffle/overview)
- [轻钱包实例 eth-lightwallet](https://github.com/ConsenSys/eth-lightwallet)
- [测试环境 TestRPC](https://www.npmjs.com/package/ethereumjs-testrpc)
  - 一个完整的在内存中的区块链，仅存在于开发的设备上。
  - 相对于 Geth私有链环境，TestRPC 在执行交易时是实时返回，而不等待默认的出块时间，这样可以快速验证新写的代码，当出现错误时，也能即时反馈。 

