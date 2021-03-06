---
layout:     post
title:      "区块链基础理论入门"
subtitle:   "区块链基础理论相关笔记"
date:       2018-07-04
author:     "jiangydev"
header-img: "img/post-bg-blockchain.jpg"
header-mask: 0.5
catalog:    true
tags:
    - BlockChain
    - Ethereum
    - Bitcoin
---

[TOC]

## 一、知识普及

### 1 比特币

#### 1.1 简介

- 数字货币银行系统
  - 无现钞、无银行网点
  - 所有账目公开可查
  - 货币发行方式
- 分布式的系统
  - 基于P2P网络
- 基于非对称密码学的交易
  - 公钥锁定比特币、私钥解锁
- 区块链作为银行账本

#### 1.2 BTC的产生

- 比特币由挖矿而产生-通过计算出一个随机数字nonce（只被使用一次的任意或非重复的随机数值 ）。
- 生成的BTC被记录在矿工的名下
  - 通过矿工的公钥的哈希值锁定
  - 交易的输出被称为“未花费交易”**UTXO** - Unspent Transaction Output
  - 比特币钱包余额就是根据众多UTXO计算出来的

#### 1.3 UTXO的生成和销毁

- 交易包含以下三项
  - 交易的输入（UTXO指针）
  - 交易的输出（UTXO）
  - 解锁脚本（私钥签名、公钥）



#### 1.4 交易UTXO+区块链=比特币系统

#### 1.5 数字货币的黄金

- 比特币的市场价格地位
  - 1比特币 > 1盎司黄金（＝31.103481g）
- 硬通货 - Hard Currency
  - 全球大部分国家都可以交易
- 易携带
  - 只需携带一个私钥
- 比特币以外的数字货币统称“山寨币”



### 2 区块链

#### 2.1 区块的生成和链接

- 共识机制 POW - Proof-of-Work
  - 通过挖矿保证自己是一个善意的节点，并获得生成区块的和在这个区块里记账权力
- 基于P2P网络，每个全节点都存储了一个历史完整的账本，抗攻击强
- 新区块通过包含前一个区块头部的哈希值（**区块的唯一标识符**）建立链接关系
  - 区块链是一列火车，每个区块是一节车厢，每节车厢里装满了交易记录
- 经过6个以上区块确认的交易才是安全确认的，因为篡改的成本巨大
- 区块链有时会产生临时的分叉二生成两条链，最终较短的链将被舍弃



#### 2.2 区块链硬分叉 - BCC/BCH

- 由于交易结构的变化，或区块结构的变化引起的

>硬分叉不会废弃较短的链，新旧节点互相拒绝对方的区块。
>
>由于交易或区块的结构变化，会使得没有及时更新软件客户端的节点出矿速度慢，但不会被废弃。



#### 2.3 区块链软分叉

- 由比特币交易的数据结构改变引起，区块的数据结构未改变。

  - 矿工激活软分叉 MASF - Miner Activated Soft Fork

  - 用户激活软分叉 UASF - User Activated Soft Fork

    - 隔离见证 Segwit - Segregation Witness

      矿工认可，但要求扩容，扩容会多收交易费，交易阻塞严重。

    - 香港共识

    - 纽约共识 Segwit2x - 2M扩容

    - 利益之争：Core为闪电网络，矿工为了交易费

> 无论节点软件是否更新，都会认可。

#### 2.4 篡改历史交易

- 使用比特币交易后，想通过其强大的算力篡改比特币交易
- 交易记录被包含在区块高度为2的区块中，为了篡改记录，从区块高度为1的区块开始挖。凭借超过51%的算力，最终在区块高度X的地方将自己变成最长的链，则篡改成功。



### 3 ICO - 首次公开代币发售

- ICO - Initial Coin Offering：获得代币，并不是获得股份，无任何股权
- 效仿IPO - Initial Public Offering
- ICO的机会和风险
- 如何判断ICO项目
  - 免费送币的直接pass
  - 白皮书用简单易懂的语言表达（模式是否清晰）
  - 解决了什么问题，区块链是必须的吗？
  - 发行代币有无必要，使用系统是否消耗代币
    - 如果没有必要，公司的成败和投资者也许没有太大关系
  - 能否快速上交易所，有交易所的背书
  - 代码是否开源，Github代码更新频率和数量
  - 创世团队的背景（学历、工作）、技术团队是否有区块链开发经验
  - 站台的早期投资人、相关领域专家

![ICO vs IPO](/img/in-post/blockchain/ICO vs IPO.png)

|          ICO           |          IPO          |
| :--------------------: | :-------------------: |
|      融资 - 早期       |     融资 - 成熟期     |
| 获得代币，没有公司股权 |     获得公司股权      |
|        风险极大        |       风险极小        |
|      关注产品本身      | 关注财务数据 + 现金流 |
|    上市交易 - 代币     |    上市交易 - 股份    |

### 4 EOS

- 以太坊的升级版
  - 每秒支持数百万交易，每三秒产出一个区块（以太坊14秒）
  - 运行智能合约免费，APP开发者根据拥有的EOS代币的数量使用系统的存储资源（以太坊运行智能合约需付费）
  - DPOS共识机制、无需挖矿、不易产生分叉（以太坊POW）
  - 用非硬分叉的手段来处理一些异常的智能合约（以太坊的TheDAO漏洞）
  - 兼容多种开发语言和VM虚拟机（支持以太坊虚拟机和智能合约在EOS上运行）
  - 系统使用费用与EOS代币当前价格无关，解决了以太坊Gas价格的问题
- 交易所站台支持
- 主创人：CTO Daniel Larimer -BM（ByteMaster）用低情商伤人的技术天才
- EOS代币分配方案：只接受ETH，总量10亿，1亿归开发团队，2亿第一批发售，剩余7亿分为350份发售
  - 每份2百万枚，每天23小时，上不封顶，**可以通过操控EOS价格吸金？**

### 5 监管

- 防范代币发行ICO融资风险的公告

  - 2017-09-0415:00由人民银行、网信办、工信部、工商总局、银监会、证监会、保监会7部委联合颁布

  - **一种未经批准非法公开融资的行为**，涉娥非法发售代币票券、非法发行证券以及非法集资、金融诈骗、传销等违法犯罪活动

  - 已完成代币发行融资的组织和个人应当做出**清退**

  - 代币融资交易平合不得从事法定货币与代币、“虚拟货币”相互之间的兑换业务

  - 各金融机构和非银行支付机构不得开展与代币发行融资交易相关的业务

- 政府的决策一定是善意的，保护投资者，如股市的涨跌停板制度，熔断机制

### 6 如何面对ICO退币 & BCC

- 在国外交易所是否可交易
  - 根据退币价格和当前价格判断
- 监管前是否只在一个或极少数交易所交易
  - 庄家容易控盘，拉高、维持高价位吸引投资者拒绝退币
- BCC官网推荐钱包
  - https://www.bitcoincash.org/
  - 唯一的非全节点钱包 ElectronCash
    - 匿名黑客基于Electrum钱包发布，**慎重使用**



## 二、比特币基础

### 1 比特币的起源

- 中本聪
- Bitcoin：A peer-to-Peer Electronic Cash System
  - 去中心化，P2P分布式的数字货币系统
  - 共识机制 - POW 工作量证明
  - 运用非对称密码学
  - 区块链作为账本

### 2 比特币特性

- 硬通货（跨境交易）
- 易携带（只需一个私钥）
- 隐秘性（只暴露钱包地址）
- 无货币超发（通货紧缩）

### 3 P2P（Peer-to-Peer）网络

| Server-Based                                               | P2P                                    |
| ---------------------------------------------------------- | -------------------------------------- |
| 中心化服务器 - C/S，B/S架构                                | 去中心化                               |
| 客户端完全信任服务器                                       | 地位对等，无主从之分；用户越多速度越快 |
| DDOS攻击 - Distributed Denial of Service分布式拒绝服务攻击 | 抗攻击                                 |

### 4 拜占庭将军问题 (Byzantine Generals Problem)

Leslie Lamport，一个关于分布式系统容错问题故事

#### 4.1 问题及解决思路

- 背景
  - 拜占庭帝国排除10支队伍，去包围进攻一个强大的敌人
  - 至少6支军队同时进攻才能攻下敌国
- 难题：一些将军可能是叛徒，会发布假的（相反的）进攻意向
- 目的：将军们需找到一种共识机制，可以远程协商、赢取战斗
- 解决方案
  - 每个节点给所有的其他节点发送消息
  - 每个节点根据接收到的所有消息来决定最终的策略
- 缺点：每个节点向全网发送大量的消息

#### 4.2 解决方案

> **前提：**若叛徒数为m，当且仅当将军总数**n>=3m+1**时才有解。
>
> 假设：进攻命令 1，按兵不动 0。

##### 4.2.1 情况一 叛将发送 000

![拜占庭将军问题解决方案 - 情况1](/img/in-post/blockchain/拜占庭将军问题解决方案 - 情况1.png)

##### 4.2.2 情况二 叛将发送 011

![拜占庭将军问题解决方案 - 情况2](.\pic/拜占庭将军问题解决方案 - 情况2.png)

### 5 比特币共识机制 - 工作量证明(POW)

- 怎么证明自己是个好人？
- POW proof of work
  - 通过付出大量的工作代价来证明自己是非恶意节点
  - 计算出一个难题的随机数答案（nonce）
  - 获取记账权利
  - 打包交易并通知其他节点
- 理性人都是逐利的，POW抑制了节点的恶意动机

### 6 常用术语

#### 6.1 挖矿

在全网中和其他节点竞争计算的过程，证明自己是非恶意节点。

- 获得的权利和义务
  - 记账权 - 将交易计入区块，（`先构建区块，后挖矿`）
  - 广播义务 - 把区块在全网广播
- 获得的奖励
  - 挖矿的奖励
  - 收取交易费用

#### 6.2 创世区块

比特币区块链的第一个区块，所有当前链上的祖先区块。

> 第一个区块：
>
> https://www.blockchain.com/en/btc/block/000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f

#### 6.3 Coinbase

- 挖矿产生的比特币

#### 6.4 区块高度&区块深度

- 区块高度：高度越高，挖矿难度越大。
- 区块深度：交易被确认的次数越多，可信度越高（链的增长）。

#### 6.5 交易确认

- 在当一项交易被区块收录后，就是交易确认
- 在此区块之后，每产生一个区块，此项交易的确认数相应加1
- 比特币钱包对交易确认数有相应设置



## 三、密码学

### 1 进位&存储单位

- 进制 - 一种计数方法

  - 用有限的数字符号来表示无限的数值，如10进制

  - 可使用的计数符号的数目决定了进制数，简称进制

    - 二进制（0，1），计算机能理解的
    - 十六进制（0-9，A-F），每一个十六进制的字符代表4个二进制组合的数字

  - 进制间关系

  - 计算机为什么不用十进制，而采用二进制？

    稳定。在计算机一开始的时候，使用电信号，以一个值来决定0或1。

- 计量术语

  - bit，位 - 最小的数据单位
  - Byte，字节 - 8个bit组成，存储空间的最小单位
  - K，Kilo，表示千：1KB = 1024B
  - M，Million，表示百万：1MB = 1024KB
  - G，Giga，表示10亿：1GB = 1024MB
  - T，Tera，表示1万亿：1TB = 1024GB
  - P，Peta，表示1百万亿：1PB = 1024TB
  - **比特币挖矿算力8359PH/s**



### 2 加密解密

#### 2.1 对称加密

- 用相同的密钥对原文进行加密和解密
- 加密过程：密钥 + 原文 => 密文
- 解密过程：密文 - 密钥 => 原文
- 缺点：无法确保密钥被安全传递

#### 2.2 非对称加密(公钥、私钥)

- 公钥用于加密，私钥用于解密
- 公钥由私钥生成，私钥可以推导出公钥
- 从公钥无法推导出私钥
- 优点：解决了密钥传输中的安全性问题
- 解决了信息传送的问题，如何验证是“确认是发送方发送的，信件没有被篡改”？

### 3 哈希 - Hash

- 将一段数据（任意长度）经过一道计算、转换为一段定长的数据
  - http://www.fileformat.info/tool/hash.htm
- 不可逆性 - 几乎无法通过Hash的结果推导出原文。
- 无碰撞性 - 几乎没有可能找到一个y，使得y的哈希值等于x的哈希值
- 使用场景
  - 发布文件的完整性验证
  - 服务器中保存用户的密码
  - 数字签名

> 哈希用例 - 用户密码的存储
>
> 1. 明文存储 - 无安全防护
> 2. 哈希存储 - 彩虹表攻击
> 3. （盐 + 哈希）存储 - 彩虹表失效



### 4 数字签名 - Digital Signature

![数字签名](/img/in-post/blockchain/数字签名.png)

#### 4.1 

**同一个内容，用密钥加密多次，结果一样？**



#### 4.2 用哈希进行摘要签名

- （重要）如果直接用私钥签名原文，公钥可以解开，会导致原文泄露
- 提高效率



### 5 证书授权中心 CA - Certificate Authority

- CA 解决了电子商务中公钥的可信度问题
- 负责证明“我确实是我”
- CA是受信任的第三方，公钥的合法性检验
- CA证书内容
  - 证书持有人的公钥
  - 证书授权中心名称
  - 证书有效期
  - 证书授权中心的数字签名

> CA证书用例 - HTTPS
>
> - 客户端通过HTTPS向服务器安全链接请求
> - 服务器用私钥加密网页内容，连同CA证书一并发给客户端
> - 客户端会根据CA证书验证CA是否合法
>   - 如果验证失败，客户端弹出警告信息
>   - 如果验证通过，客户端使用CA证书中的公钥向服务器发送加密信息



## 四、比特币交易 - 锁定&解锁

[阅读：中本聪的天才：比特币以意想不到的方式躲开了一些密码学子弹](http://www.8btc.com/satoshis-genius-unexpected-ways-in-which-bitcoin-dodged-some-cryptographic-bullet)

### 1 UTXO - Unspent TransXtion Output

- 未花费交易输出
- 用比特币拥有者的公钥锁定（加密）的一个数字
- UTXO == 比特币
- 比特币系统里没有比特币，只有UTXO
- 比特币系统里没有账户，只有UTXO（公钥锁定）
- 比特币系统里没有账户余额，只有UTXO（账户余额只是比特币钱包的概念）
- UTXO存在全节点的数据库
- 转账将消耗掉属于你自己的UTXO，同时生成新的UTXO，并用接收者的公钥锁定



### 2 交易的结构

- 交易的输出 UTXO
  - 锁定的比特币数量
  - 锁定脚本（用接收者的公钥哈希）
- 交易的输入 UTXO + 解锁脚本
  - 解锁脚本：签名，发送者公钥



### 3 交易验证 - 基于栈的脚本语言 

#### 3.1 栈结构及操作

- 栈 stack - 操作数据的一种结构
  - 只能从一段操作数据，后进先出LIFO（Last In, First Out）
  - 压栈（push）、出栈（pop）
- 交易验证 - 基于栈的脚本语言
  - 对栈的操作：OP_DUP
  - 逻辑运算符：OP_EQUALVERIFY
  - 加解密运算符：OP_HASH160, OP_CHECKSIG
  - 算数运算符：OP_ADD, OP_SUB, OP_MUI, OP_DIV

#### 3.2 逆波兰表示法

- 简易运算规则
  - 所有操作符位于操作数的后面
  - 遇到操作数（数字），则压栈PUSH
  - 遇到二元运算符（+, -, *, /）
    - 先将2个操作数进行出栈POP
    - 然后对运算数进行计算
    - 最后将计算结果压栈
- 传统表达式（中缀表示法）
- 逆波兰表达式（后缀表示法）



### 4 交易验证

- 锁定脚本

  - OP_DUP OP_HASH160 <**发送者**公钥哈希> OP_EQUALERFITY OP_CHECKSIG

  > 小提示：
  >
  > 发送者公钥哈希：谁拥有UTXO，这个公钥就是谁的，私钥也是UTXO所有者的。

- 解锁脚本

  - <发送者签名> <发送者公钥>

- 交易验证：运行解锁脚本 + 锁定脚本 => True

- 后缀表达式如下

  ![交易验证过程](/img/in-post/blockchain/交易验证过程.png)



## 五、区块的生成和链接

### 1 交易的传播 & 验证

- 交易包含两个部分，n输入和m输出，n>=0, m>0 (n=0的情况：挖矿产生)
  - 输入 == 要被花费的 UTXO + 解锁脚本
  - 输出 == UTXO (币值 + 锁定脚本)
- 钱包软件生成交易，并向邻近节点传播
- 节点对收到的交易进行验证，并丢弃不合法交易
  - 交易的size要小于区块size的上限
  - 交易输入UTXO是存在的
  - 交易输入 UTXO 没有被其他交易引用 - 防止双花（Double Spending）
  - 输入总金额 > 输出的总金额
  - 解锁脚本验证
- 将合格的交易加入到本地的 Transaction 数据库中，并将合法交易转给临近节点

### 2 区块的生成

- 矿工在挖矿前要组件区块
  - 将 coinbase 交易打包进区块

  - 将交易池中高优先级的交易打包进区块
    - 优先级 = 交易的额度 * UTXO 的深度 / 交易的 size
    - 防粉尘攻击：类似 DOS 攻击

  - 创建区块的头部

    ![区块的头部](/img/in-post/blockchain/区块的头部.png)

- 挖矿成功后，将计算出来的随机数 nonce 填入区块头部，向邻近节点传播



### 3 区块的链接验证

- 相邻节点收到新区块后，立即做以下检查：
  - 验证 POW 的 nonce 值是否符合难度值
  - 检查时间戳是否小于当前时间2小时
  - 检查 Merkle 树根是否正确
  - 检查区块 size 要小于区块 size 的上限
  - 第一笔交易必须是 coinbase 交易
  - 验证每个交易
- 查看源代码
  - http://btc.yt/lxr/satoshi/ident?_=CheckBlock

### 4 Merkle Tree

- 树 - 由多个节点组成的一种数据结构
  - 每个节点储存数据
  - 根节点 Root
  - 父节点、子节点、兄弟节点
- 构建二叉搜索树
- Merkle Tree
  - 防止数据篡改
  - 快速验证某个交易是否存在
  - 节点存储  Hash 值
  - 从叶子节点构造树
    - 如先是 Hash交易 a得到 Ha，Hash交易 b得到 Hb，再 Hash前两个 Hash（也就是 Ha和 Hb）得到 Hab，其他节点也是同理递归，最终得到 Merkle Root ；
    - 交易是偶数个，如果交易是奇数，生成 Merkle Tree 时，复制其中一个交易即可。

![Merkle Tree](/img/in-post/blockchain/Merkle Tree.jpg)

> Merkle Path 验证路径实例：
>
> merkle proof从某处出发向上遍历，算出 merkle Root的所需要经过的路径节点。在上图的例子中，如果轻量钱包要验证 Txb（红色方框）是否已经包含在区块中，可以向全量节点请求 merkle Proof，用于证明 Txb的存在，过程为：
>
> 1. 全量节点只要返回黄色部分的节点信息（Ha与 Hcd）
> 2. 轻量节点执行计算 `Hash(Txb)=Hb à Hash(Ha + Hb)=Hab à Hash(Hab + Hcd)=Habcd`，计算出来的 merkleRoot(也就是 Habcd)跟已知区块头中的 merkleRoot比较，如果一样则认为交易确实已经入块。
>
> 参考资料：
>
> 1. [区块链的区块结构和持久化｜迅雷来鑫](https://mp.weixin.qq.com/s/6u6f5M5iX6-qV-yJACLaaw)



## 六、区块链分叉

### 1 硬分叉 & 软分叉

#### 1.1 软分叉

- 由比特币交易的数据结构改变引起，区块的数据结构未改变。

- 老节点接收旧格式的区块，新节点只接受新区快

  - 矿工激活软分叉 MASF - Miner Activated Soft Fork

  - 用户激活软分叉 UASF - User Activated Soft Fork

    - 隔离见证 Segwit - Segregation Witness （`BIP141`）

      矿工认可，但要求扩容，扩容会多收交易费，交易阻塞严重。

    - 香港共识

    - 纽约共识 Segwit2x - 2M扩容

    - 利益之争：Core为闪电网络，矿工为了交易费

> 无论节点软件是否更新，都会认可。

#### 1.2 硬分叉 - 如 BCC/BCH

- 由于交易结构的变化，或区块结构的变化引起的

> 硬分叉不会废弃较短的链，新旧节点互相拒绝对方的区块。
>
> 由于交易或区块的结构变化，会使得没有及时更新软件客户端的节点出矿速度慢，但不会被废弃。



### 2 临时分叉

- 仅发生于几乎同时爆块的情况
- 分叉是暂时的
- 根据共识机制、矿工最终切换到最长链上挖矿
- 短链上的交易全部无效，包括矿工费
- 矿工是否会继续在短链上挖矿以弥补损失？ 不会，除非算力能达到50%，使该链长度更长。



### 3 隔离见证

- 隔离见证 Segwit - Segregation Witness
  - 安全隐患，黑客通过改变交易签名信息改变交易 ID
  - 将签名部分从交易中移除，从而间接扩容
- 香港共识：2016年2月香港会议，Core团队的代表同一在实施隔离见证后扩容到2M，后来反悔
- 纽约共识Segwit2x：2017年5月纽约会议，代表全网80%的算力的矿业代表在纽约达成 Segwit2x 扩容
  - 实现 BIP141(Segwit) + BIP91(缓和折中方案) + BIP102(升级到2M)

> 小问题：
>
> Q1：Core 为什么不支持扩容？
>
> A1：闪电网络，见6.4



### 4 Core VS 矿工

- Core 反对原因
  - 不愿意轻易更改系统，如银行仍使用 COBOL 语言编写的系统
  - 以防个人不能运行全节点，因为每个区块太大了
  - 增加 2M 没什么大的作用，用`闪电网络`解决

- 矿工反对原因
  - 闪电网络将会导致交易的中心化、违背比特币点对点交易的初衷
  - 闪电网络隶属 BlockStream 公司，大多数 Core 团队的成员是 BlockStream 的雇员
  - 交易费的损失
- Core 垄断了海外的论坛，利用他们对社区的影响力“封杀”反对 Core 的开发者和社区成员
- 结论：Core 为闪电网络铺平道路，矿工为了交易费



### 5 BCC 分叉

- BU团队 - Bitcoin Unlimited
  - BU 的大方案曾获得了全网 40% 的算力支持
  - 方案的 bug 太多，未能获得获得社区广泛支持
- BitcoinABC 团队（Adjustable Blocksize Cap）
  - 基于 BU 的代码，打造了 BCC，2017年8月1日硬分叉
  - BCC 没有 Segwit，区块容量8M
  - BCC 官网：http://www.bitcoincash.org



## 七、比特币钱包

### 1 比特币官方钱包

- 下载官方钱包、全节点

  http://bitcoin.org/zh_CN/choose-your-wallet

- 运行比特币测试网络

  - `$ bitcoin-qt -testnet`
  - 控制台

- 申请免费比特币

  - 比特币水龙头：https://testnet.manu.backend.hamburg/faucet



### 2 比特币地址生成

#### 2.1 钱包地址生成流程

![钱包地址生成](/img/in-post/blockchain/钱包地址生成.png)

> 小提示：
>
> 1. RANDOM：随机生成私钥；
> 2. SECP256K1：一种椭圆曲线算法；
> 3. RIPEMD160：主要使公钥的长度减短，减小比特币钱包大小，避免交易数减少；
> 4. DUBLE SHA256：通过“公钥哈希”双哈希计算，取出前4字节，用于“检验”；
> 5. BASE58编码：58进制，减少公钥长度。

#### 2.2 Base64 编码

![Base64编码](/img/in-post/blockchain/Base64编码.png)

#### 2.3 Base58 编码

Base58编码：`0, I, O, l, +, /`，这5个字符易混淆，含其他符号，所以去除。

![Base58编码](/img/in-post/blockchain/Base58编码.png)



### 3 钱包私钥格式 WIF

#### 3.1 WIF - Wallet Import Format

- WIF 私钥`格式更短`，标准格式私钥256个bit，16进制长度 = 256 / 4 = 64 字符
  - 经过 Base58 编码，表示长度更短
  - 未压缩格式私钥：5开头，大小`51字节`，第一位存放版本信息
  - 压缩格式私钥：K或L开头，大小`52字节`，第一位放版本信息，多出的最后一个字节存放是否压缩信息
- WIF 格式可以`自动侦测地址错误`
  - 通过私钥的哈希值产生校验码

#### 3.2 公钥 - 压缩格式 & 非压缩格式

> 小提示：
>
> 由64位私钥推导出X和Y轴的值（64位）；
>
> 压缩 - 并非真的”压缩“，因为X，Y的值有一个即可，之后省略X轴的值，并使用 02/03 开头。

![公钥-压缩格式&非压缩格式](/img/in-post/blockchain/公钥-压缩格式&非压缩格式.png)



### 4 轻钱包 & SPV 验证机制

- 轻钱包是比特币的非全节点，存储空间限制
  - 只下载 block header，size 很小只有80字节，区块本身大小1M，1.8M
  - 向邻近全节点发送请求得到 UTXO 信息
- 简单支付验证 SPV - Simplifield Payment Verificaion
  - 非全节点支付验证，判断交易是否已经在区块链中，多少确认数
  - 向邻近节点发送请求关于特定比特币地址和交易的信息
  - 邻近全节点向钱包返回 Merkle path 验证路径和相应 block header
  - 钱包根据 Merkle path 计算出 Merkle root，验证是否匹配 block header 里的 Merkle root 值
  - 确认相应 block header 的深度是否大于 6



### 5 生成自己的私钥和地址

#### 5.1 生成私钥

- 通过某种私钥生成网站、安全性问题
- 掷色子，16面的扔64次
- 用中文、英文、汉语拼音的哈希值
  - http://www.fileformat.info/tool/hash.htm

#### 5.2 私钥转比特币地址

- 保存 bitcoin.sh https://github.com/grondilu/bitcoin-bash-tools/

- bash 检查

  ```shell
  # 检查 bash 版本，如小于 4 则需要升级
  $ bash --version
  # 更新 brew 和 bash
  $ brew update && brew install bash
  # 把新的 shell 添加到 shell 列表
  $ sudo bash -c 'echo /usr/local/bash >> /etc/shells'
  # 启用新的 shell
  $ chsh -s /usr/local/bin/bash
  ```

- 运行脚本，把私钥转换成比特币地址

  ```shell
  # 在 bitcoin.sh 的目录运行
  $ source bitcoin.sh
  $ newBitcoinKey [0x私钥]
  ```

- 重启机器、清除所有记录

## 八、挖矿

### 1 为什么要挖矿

- 比特币系统里为什么要设计挖矿
  - 增加恶意行为的成本
  - 争夺记账权利，获取奖励
- 传统的挖矿：体力劳动、矿工在以一个矿场内不停的尝试，最终挖出黄金
- 比特币挖矿：“脑力劳动”、把挖矿变成计算，矿工使用电脑不停地计算
- 每开采 210,000 个区块，挖矿奖励减半
  - 2009年1月 - 2012年11月，奖励50BTC
  - 2012年11月 - 2016年7月，奖励25BTC
  - 2040年，所有 BTC 被挖出，挖矿没有奖励，矿工以手续费为生



### 2 挖矿的流程

挖矿流程图：

![挖矿流程图](/img/in-post/blockchain/挖矿流程图.png)

> 小提示：
>
> 1. 区块头部的“难度值”，不同于哈希结果数值要比较的“目标值”；
>
> 2. 如果 Nonce 遍历2^256后，Hash还不能满足难度目标值的解决方案
>
>    调整 Merkle root(即改变交易顺序)、改变时间戳等，Nonce 重置，并重新开始计算。
>
> 3. 挖矿可以用纸和笔计算？
>
>    https://gizmodo.com/mining-bitcoin-with-pencil-and-paper-1640353309



### 3 难度调整

- 每2016个区块调整难度
- 新目标值 = 当前目标值 * (过去2016区块用时分钟/20160分钟)
- 难度目标值：区块头部 hash 要小于的值
  - 由系数和指数构成，如 0x1903A30C
  - 目标值 = 0x03A30C * 2^(0x08 * (0x19 - 0x03))
  - 将目标值用16进制表示
- 难度：难度为1的难度目标值/当前难度目标值 >= 1
  - 难度值1为中本聪最初挖矿难度值，为 0x00000000FFFF0000...0000
  - 难度 = <难度为1的难度目标值>/<目标值的16进制表示>



### 4 硬件

CPU -> GPU -> FPGA -> ASIC -> MiningPool



### 5 矿池

- 矿工无需运行全节点，只需安装挖矿软件
- 矿池管理员维护全节点，并把任务分段，给矿工布置挖矿任务，例如让A尝试`0~FFFFFF`，让B尝试`1000000~1FFFFFF`
- 矿池分类
  - PPLNS：Pay Per Last N Shares
    - 根据过去一段时间所做的贡献，获得了多少“份额”来获得比特币奖励
    - 滞后惯性，挖矿收益有一定的延迟
  - PPS：Pay Per Share
    - 按矿工所占矿池算力大小按比例获得比特币报酬，每天都分
    - 某矿工当日所获奖励 = 预估矿池当日可挖到比特币数量 * 某矿工算力所占比例
  - FPPS：在分 coinbase 的基础上，也分得交易费
  - P2P矿池 - P2Pool
    - 防止托管矿池管理者作弊，基于区块链技术，去中心化的矿池管理系统
    - 矿工需要自己运行全节点
    - 根据矿工贡献的算力来确定分红
  - 托管矿池 - 矿工把自己买的矿机托管到矿场，由矿场帮助维护打理



## 九、区块链安全

### 1 可塑性攻击

- 可塑性也称可锻性，是指一个物体的外形改变不引起质量和物理化学属性的变化
- 交易签名具有可塑性，有多种写法
- 修改交易签名引起交易的哈希值改变，即 TXID 改变
- TXID 发生变化会导致原 TXID 无法找到，造成攻击漏洞
- 隔离见证
  - 可伪造的签名部分移出交易数据结构，在另一个地方存放签名
  - 改变签名不影响 TXID 的变化



### 2 如何“偷币”

#### 2.1 构造交易

- 查看UTXO

  ```shell
  $ bitcoin-cli listunspend
  ```

  命令结果：

  ```json
  [
      {
          "txid": "9ca8...",
          "vout": 0,
          "address": "",
          "account": "",
          "scriptPubKey": "76a9...",
          "amount": 0.05000000,
          "confirmations": 7
      }
  ]
  ```

- 生成初始交易

  ```shell
  $ bitcoin-cli createrawtransaction \
  '[{"txid": "9ca8...", "vout": 0}]' \
  '{"<bitcoin address>": 0.025, "<bitcoin address>": 0.0245}'
  ```

  输出结果，16进制的编码格式。

- 查看初始交易

  ```shell
  $ bitcoin-cli decoderawtransaction <初始交易16进制编码>
  ```

  ```json
  {
      "txid": "0793...",
      "version": 1,
      "locktime": 0,
      "vin" : [
          {
              "txid": "9ca8...",
              "vout": 0,
              "scriptSig": {
                  "asm": "",
                  "hex": ""
              },
              "sequence": 1234567890
          }
      ],
      "vout": [
          {
              "value": 0.02500000,
              "n": 0,
              "scriptPubKey": {
                  "asm": "OP_DUP ...",
                  "hex": "76a9...",
                  "reqSigs": 1,
                  "type": "pubkeyhash",
                  "addresses": [
                      "<receiver's bitcoin address>"
                  ]
              }
          }
      ]
  }
  ```

  > 小提示：
  >
  > 1. locktime：指该交易最早可以打包到区块的高度或时间，一般为0；
  > 2. vin 输入，存放解锁脚本；
  > 3. vout 输出，存放锁定脚本；`reqSigs`指“pubkeyhash”，如果大于1，指多重签名；

- UTXO 锁定方式

  - MultiSig - Multi Signature 多重签名

    - 锁定脚本：`2 <PubKey A> <PubKey B> <PubKey C> 3 OP_CHECKMULTISIG`

      解释：指接收这 3 个人的 PubKey，只要其中 2 个人签名即可。

    - 解锁脚本：`OP_0 <Sig B> <Sig C> 2 <PubKey A> <PubKey B> <PubKey C> 3 OP_CHECKMULTISIG`

      如上，只要 B, C的签名即可。

  - P2PK - Pay to Public Key 用公钥锁定

    - 锁定脚本：`<PubKey> OP_CHECKSIG`
    - 解锁脚本：`<Sig> <PubKey> OP_CHECKSIG`

  - P2PKH - Pay to Public Key Hash 用公钥的哈希锁定

    - 锁定脚本：`OP_DUP OP_HASH160 <PubKey Hash> OP_EQUAL OP_CHECKSIG`
    - 解锁脚本：`<Sig> <PubKey> OP_DUP OP_HASH160 <PubKey Hash> OP_EQUAL OP_CHECKSIG`

  - P2SH - Pay to Script Hash 用脚本锁定

    - Redeem Script(偿还脚本)：`2 <PubKey A> <PubKey B> <PubKey C> 3 OP_CHECKMULTISIG`
    - 锁定脚本：`OP_HASH160 <Redeem Script Hash> OP_EQUAL`
    - 解锁脚本：`<Sig1> <Sig2> <redeem script> OP_HASH160 <Redeem Script Hash> OP_EQUAL`

- 对交易签名 & 发送

  - 对交易进行签名

    ```shell
    $ bitcoin-cli signrawtransaction ...
    ```

  - 向全网广播交易

    ```shell
    $ bitcoin-cli sendrawtransaction ...
    ```


#### 2.2 “偷币”

1. 创建私钥，生成对应的公钥和比特币地址
2. 调用相应 API 接口 listunspent
3. 查看 amount 字段的值，有没有比特币，如果没有返回步骤1
4. 调用相应 API 接口 createrawtransaction，创建自己的比特币地址转钱的初始交易
5. 调用相应 API 接口 signrawtransaction，对交易签名（用到生成的公钥）
6. 调用相应 API 接口 sendrawtransaction，将交易向全网广播
7. 回步骤1



### 3 重放攻击 - 比特币系统中，非人为攻击

- 重放攻击 Replay Attach
  - 攻击者重复发送相同的数据包到目的主机，用以欺骗系统
- 比特币重放攻击
  - 并非黑客的主动攻击
  - 从美国寄信到清华大学，大陆清华还是台湾清华，复制一份，都发送
  - 区块链硬分叉后需有防止重放攻击措施，BCC
    - 改变交易或签名的结构和验证规则，例如按位取反
- 以太坊 ETH ETC 被重放攻击



### 4 其他攻击

- 粉尘攻击：大量低额度的交易，目的让网络拥堵
- 空区块攻击：不打包交易，只有coinbase
- 51 攻击：拥有 51% 以上的算力
  - 双重支付攻击，一笔钱支付给两个不同的人（没有等6个交易后就发货的风险）
  - 拒绝服务攻击，拒绝对某个特定的比特币地址提供服务（遇到某个比特币地址则丢弃该区块，或故意从该比特币地址交易区块的前一个区块进行 51 攻击）

## 十、智能合约

### 1 智能合约

- 一份电子形式合同或协议
  - 以一种计算机程序的形式展现
  - 通过计算机自动执行和验证，无需人为干预，例如柜台取款和ATM机取款
  - 通过淘宝下单付款后商家发货，确认收获后系统自动转钱给商家
- 法律层面上是否曾认可，有待商榷
  - 小蚁的股权发行，登记和转让交易
  - 二手房过户，能否绕过住建委
  - 需要政府的推动和背书



### 2 风险案例 - The DAO

- 合约一旦部署成功，将很难更改
- The DAO 事件
  - The mother of all DAOs
  - 一个智能合约形式的 VC 基金，众筹了 1.62 亿美元
  - 股东通过众筹获得代币和投资投票权
  - 代码漏洞，被黑客将币盗走大量代币
  - 被迫分叉，分裂为 ETH 和 ETC 两种代币

> 小提示：
>
> GP：不出钱，或少出钱的人，通过投资经验帮助 LP 投资；
>
> LP：投资人，出钱方，有知情权，但不干预投资；



### 3 以太坊主要特性

- Vitalik 于 2015 年 7 月创建的区块链

- `区块链 2.0`，支持智能合约

- 支持图灵完备语言，Solidity

  - 可图灵：指在可计算性理论中，编程语言或任意其他的逻辑系统如具有等用于通用图灵机的计算能力。
  - 图灵完备：虽然图灵机会受到存储能力的物理限制，图灵完全性通常指具有无限存储能力的通用物理机器或编程语言。简单来说， 一切可计算的问题都能计算，这样的虚拟机或者编程语言。

- `Gas`：衡量在一个计算中要求的费用单位

  - 总费用 = Gas limit * Gas prices
  - gas 不够时交易处理就会被终止，回退到之前的状态，不退费

- 虚拟机：通过软件模拟计算机硬件的一套系统，运行在宿主机系统上

- 以太坊虚拟机 EVM：执行智能合约的安全运行环境，通过执行合约的 bytecode 来执行智能合约

- 货币发行总量无上限，出块时间平均每 12-15 秒，每个区块奖励 5ETH

- 叔区块 `uncle block` 奖励

  - 也称为`孤块(orphan block)`，它是合法的，但是在比特币系统中，由于它不是最长的链而被抛弃；以太坊中，认为叔块可以为主链的安全作出贡献，Ethereum的GHOST协议会支付报酬。

  - 解决的问题

    - 通过鼓励引用叔块，使引用主链获得更多的安全保证(孤块本身也是合法的)
    -  比特币中，采矿中心化(大量的集中矿池)成为一个问题。给叔块报酬，可以一定程度上缓解这个问题 

  - 叔块的引用

    - 区块可以不引用，或者最多引用两个叔块

    - 叔块必须是区块的前2层~前7层的祖先的直接的子块

    - 被引用过的叔块不能重复引用

    - 引用叔块的区块，可以获得挖矿报酬的1/32，也就是5*1/32=0.15625 Ether。最多获得2*0.15625=0.3125 Ether

    - 被引用的叔块，其矿工的报酬和叔块与区块之间的间隔层数有关：

      | 间隔层数 | 报酬比例 | 报酬(ether) |
      | -------- | -------- | ----------- |
      | 1        | 7/8      | 4.375       |
      | 2        | 6/8      | 3.75        |
      | 3        | 5/8      | 3.125       |
      | 4        | 4/8      | 2.5         |
      | 5        | 3/8      | 1.875       |
      | 6        | 2/8      | 1.25        |

- Keccak AHS-3 哈希算法，反 ASIC 挖矿，需要大量内存



### 4 智能合约的部署运行

- 学习资料：http://ethereum.gitbooks.io/frontier-guide/

- 安装 geth 客户端 - go语言版本

  ```shell
  $ brew tap ethereum/etnereum
  $ brew install ethereum
  ```

- 命令行部署

  ```shell
  # 创建私链
  $ geth --datadir "privateChain" init genesis.json
  # 进入控制台
  $ geth --datadir "privateChain" console
  # geth attach ipc:\\.\pipe\geth.ipc
  > personal.listAccounts
  # 创建新的账户
  > personal.newAccount("12345678")
  # 查询账户余额
  > addr0 = <创建账户时返回的值>
  > web3.eth.getBalance(addr0)
  # 转账给 addr1 1.5 个以太币
  > amount = web3.toWei(1.5); eth.sendTransaction({from:addr0, to:addr1, value:amount})
  # 解锁账户
  > personal.unlockAccount(addr0)
  # 挖矿，这里挖一个就停止
  > miner.start(); admin.sleepBlocks(1); miner.stop()
  ```

- 部署智能合约源代码

  - 在线编译器：(原地址)http://ethereum.github.io/browser-solidity，（新地址）http://remix.ethereum.org

    在编译器中，

  - 源代码

    ```
    pragma solidity ^0.4.0;
    
    contract Rating {
        function setRating (bytes32 _key, uint256 _value) public {
            ratings[_key] = _value;
        }
        mapping(bytes32 => uint256) public ratings;
    }
    ```

  - 发布的两种方式（要挖矿）

    - `WEB3DEPLOY`：直接在控制台粘贴运行即可；

    - 字节码方式

      ```shell
      > eth.sendTransaction({from:addr0, code:'<粘贴BYTECODE>', value:web2.toWei(10, 'ether')})
      > brower_ballot_sol_rating.setRating.sendTransaction(1, 3, {from:addr0})
      ```


