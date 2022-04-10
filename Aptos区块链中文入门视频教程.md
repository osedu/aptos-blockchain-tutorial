[toc]

# Aptos公链简介

Aptos是由 Diem 原团队成员成立的新公链项目. 

Aptos的目标是建立一个更具可扩展性的区块链，使用Move编程语言以及BFT共识协议，旨在为数十亿用户提供服务，尽早满足大型企业客户的需求。

> ==A Layer 1 for everyone. Building the safest and most scalable Layer 1 blockchain.==

- 官网：https://aptoslabs.com/
- Github：https://github.com/aptos-labs/
- 开发者网络：https://aptos.dev/
- REST API：https://fullnode.devnet.aptoslabs.com
- 水龙头：https://faucet.devnet.aptoslabs.com
- 区块链浏览器：https://explorer.devnet.aptos.dev/
- 开发者网络状态：https://status.devnet.aptos.dev/
- 全节点中提供的RESTful API规范文档：https://fullnode.devnet.aptoslabs.com/spec.html#/
- REST API: https://github.com/aptos-labs/aptos-core/blob/main/api/doc/openapi.yaml
- https://github.com/aptos-labs/aptos-core/tree/main/api


# 1 基本概念
## 账号（Accounts）

- 账号：16字节地址标识
- 账号：是Move模块和Move资源的容器


每个帐号的状态包括代码和数据：
- 代码（Code）：Move模块包含代码（类型type和过程procedure声明），但它们不包含数据。该模块的procedures对更新区块链全局状态的规则进行编码。
- 数据（Data）：Move资源包含数据但没有代码。每个资源值都有一种类型。

**一个帐号可以包含任意数量的 Move 资源和 Move 模块。**

### 初始化帐号
16个字节长的帐号地址：从该帐号的初始公共验证密钥（initial public verification key(s)）导出。

aptos支持两种签名方案：Ed25519（用于单一签名交易）和MultiEd25519（用于多重签名交易）。

每个帐号都保存一个顺序号（`sequence_number`），`sequence_number`代表下一个交易序列号，以防止交易的重放攻击。帐号创建时将`sequence_number`初始化为0。

### 认证密钥
每个帐号都会保存一个身份验证密钥。此身份验证密钥使帐号所有者能够轮换(rotate)与帐号相关联的私钥，而无需更改其帐号的地址。在轮换期间，身份验证密钥根据新生成的私钥-公钥对进行更新。

#### 单一签名验证
生成认证密钥和帐号地址的过程:
1. 生成密钥对：首先生成一个密钥对`(pubkey_A, privkey_A)`，具体方法参考：RFC 8032.
2. 导出32字节的认证密钥：`auth_key = sha3-256(pubkey_A | 0x00)`, `0x00`代表单一签名。auth_key的前16个字节是认证密钥前缀（authentication key prefix），后16个字节是帐号地址。

任何创建帐号的交易都需要帐号地址和认证密钥前缀；与现有帐号交互的交易只需要帐号地址即可。

#### 多重签名验证
多重签名：使用K-of-N多重签名认证，账户总共有N个签名人，至少K个签名认证后才能进行交易。

创建K-of-N多签名认证密钥过程:
1. 生成密钥对:生成N个ed25519公共密钥`p_1, ..., p_n`
2. 导出32字节的认证密钥: `auth_key = sha3-256(p_1 | … | p_n | K | 0x01)`, `0x00`代表多重签名，k表示至少k个以上签名认证才能进行交易。auth_key的前16个字节是认证密钥前缀（authentication key prefix），后16个字节是帐号地址。 

### 帐号资源
Aptos中帐号数据存储在资源中。初始资源是帐号数据本身(身份验证密钥`authentication key`和序列号`sequence_number`)。创建帐号后，可以添加代币或NFT等额外资源。

为了创建帐号，Aptos testnet需要帐号的公钥（public key）和一定数量的`TestCoin`(https://github.com/aptos-labs/aptos-core/blob/5d80b0d5fe09dbbf3e190459cdc376e8333ad6a3/aptos-move/framework/aptos-framework/sources/TestCoin.move#L152)来添加到该帐号，从而使用这两种资源创建一个新帐号。

## 事件（Events）
事件用于获取交易的执行情况，它在交易执行过程中发出，每个Move模块都可以定义自己的事件以及发出事件的时间。

例如，在代币转账过程中，发送方和接收方的帐号将分别发出`SentEvent`和`ReceivedEvent`事件。事件发出的数据被保存在区块链账本中，可以通过REST接口的`Get events by event`（[https://fullnode.devnet.aptoslabs.com/spec.html#/operations/get_events_by_event_handle](https://fullnode.devnet.aptoslabs.com/spec.html#/operations/get_events_by_event_handle)）句柄进行查询。

**事件查询示例：**

查询帐号地址：`0x458bbba3a14d8028ca2c681428b5a846`的交易信息（已经向另一帐号地址发送了一笔交易）。

REST 接口的查询方式：[https://fullnode.devnet.aptoslabs.com/accounts/458bbba3a14d8028ca2c681428b5a846/events/0x1::TestCoin::TransferEvents/sent_events](https://fullnode.devnet.aptoslabs.com/accounts/458bbba3a14d8028ca2c681428b5a846/events/0x1::TestCoin::TransferEvents/sent_events)

输出的JSON信息如下：
```json
[
  {
    "key": "0x0000000000000000458bbba3a14d8028ca2c681428b5a846",
    "sequence_number": "0",
    "type": "0x1::TestCoin::SentEvent",
    "data": {
      "amount": "1000",
      "to": "0xba255632c944553eb275a2a96ac61a00"
    }
  }
]
```
- key:每一个注册的事件拥有唯一的key。上例中的key(`0x0000000000000000458bbba3a14d8028ca2c681428b5a846`)映射到帐号`0xcaa60eb4a01756955ab9b2d1caca52ed`的事件（`0x1::TestCoin::TransferEvents/sent_events`）上。
可以通过事件key来查询事件详情：[https://fullnode.devnet.aptoslabs.com/accounts/458bbba3a14d8028ca2c681428b5a846/events/0x1::TestCoin::TransferEvents/sent_events](https://fullnode.devnet.aptoslabs.com/accounts/458bbba3a14d8028ca2c681428b5a846/events/0x1::TestCoin::TransferEvents/sent_events).输出结果同上。
- sequence_number:从0开始的递增序号
- type: 事件类型。多个事件可以有相同或相似的事件类型，尤其是在使用泛型的时候（especially when using generics）。
- data：事件内包含的具体数据。一般原则是包含所有必要的数据，以便理解资源的变化情况。

## 全节点（FullNodes）
Aptos节点：

- 节点用来跟踪Aptos区块链的状态。
- 客户端通过Aptos节点与区块链进行交互。
- 两种节点类型：
  - Validator nodes：验证器节点
  - FullNodes：全节点

每个Aptos节点都包括如下逻辑组件:

- REST service
- Mempool: 它保存已提交给区块链但尚未达成一致或执行的交易的内存缓冲区。该缓冲区在验证器节点和FullNodes之间复制。
  - FullNode的JSON-RPC服务将交易发送到验证器节点的Mempool。
  - Mempool对交易执行各种检查，以确保交易的有效性并防止DOS攻击。
  - 当一个新交易通过初始验证并被添加到Mempool中时，它将被分发到网络中其他验证器节点的Mempool中。
  - 当一个验证节点暂时成为共识协议中的领导者时，共识协议从内存池中拉出交易，并提出一个新的交易块。该块被广播给其他验证器，并且包含该块中所有交易的总排序。然后每个验证器执行这个块，并提交投票决定是否接受这个新的块提议。
- Consensus (disabled in FullNodes):该组件负责对交易块进行排序并通过与网络中的其他验证器节点一起参与consensus协议来对执行结果达成一致。
- Execution：该组件协调交易块的执行并维护瞬态。对这种短暂状态的一致投票。在consensus将块提交给分布式数据库之前，Execution会维护执行结果的内存表示。Execution使用虚拟机来执行交易。Execution充当系统输入(由交易表示)、存储(提供持久层)和虚拟机(用于执行)之间的粘合层。
- Virtual Machine：用于在每个交易中运行Move程序，并确定执行结果。节点的Mempool使用VM对交易执行验证检查，而Execution使用VM执行交易。
- Storage：将一致同意的交易块及其执行结果保存到本地数据库中。
- State synchronizer：节点使用该组件来“赶上”区块链的最新状态并保持最新。

Aptos-core软件（https://github.com/aptos-labs/aptos-core）可以配置为作为验证器节点或全节点来运行。

任何人都可以运行FullNodes。FullNodes重新执行Aptos区块链历史上的所有交易。全节点通过与上游参与者(例如，其他全节点或验证器节点)同步来复制区块链的整个状态。为了验证区块链状态，FullNodes需要接收由验证器签名的交易集和Merkle累加器的根哈希。

此外，FullNodes接收Aptos客户端提交的交易，并将它们直接(或间接)转发给验证器节点。

全节点和和验证器节点代码相同，但是全节点不参与区块链共识。

第三方区块链浏览器、钱包、交易所和DApps可以通过运行本地全节点实现如下功能：
- 利用全节点提供的REST接口进行区块链交互。
- 获得Aptos账本的一致视图。
- 避免对读取流量的速率限制。
- 对历史数据运行自定义分析。
- 获得特定的链上事件的通知。

## Gas和交易費
在Aptos区块链上执行交易时，使用gas来跟踪和度量资源的使用情况。

Gas可以确保在Aptos区块链上运行的Move程序最终能够终止，以达到限制Move程序所使用的计算资源的目的。Gas还提供收取交易费的能力（基于执行期间消耗的资源）。

当客户端向Aptos区块链提交要执行的交易时，需要指定如下字段:
- max_gas_amount: 消耗的最大gas数量。实现对计算资源的限制。gas消耗达到上线，程序终止。
- gas_price: 指定gas价格，即一个gas代表多少gas currency
- gas_currency: 指定交易费使用的货币。

**向客户端收取的最大交易费用计算公式**：`gas_price * max_gas_amount`。

`gas_price` 和 `max_gas_amount` 随着Aptos区块链的资源供需变化而产生波动。

### 资源使用类型
gas系统需要跟踪交易使用的主要资源。将交易使用的资源分为3个维度:
1. 计算成本：执行交易的计算成本。
2. 网络成本：通过Aptos生态系统传播交易的网络成本。
3. 存储成本：Aptos区块链上交易执行过程中创建和使用的数据的存储成本。

**计算成本和网络成本是暂时性的**。**存储则是长期性的**，一旦分配了数据，该数据将一直存在，直到被删除。对于帐号来说，数据是永久存在的。

这三个资源维度都可以独立波动，它们的gas price相同。必须正确跟踪每个维度的gas使用量，gas price仅作为总gas使用量的单一乘数。因此，交易的gas使用需要与执行交易的真实成本密切相关。

### 用gas计算交易费用

发送交易时，交易费(以指定的gas货币表示)是gas价格乘以资源使用量（由虚拟机给出此交易需要消耗的gas数量）。

![Gas and Transaction Flow](https://aptos.dev/assets/images/using-gas-1d6edf9da2560f49ba68a39a3c1920a7.svg)

上图表中，序言（prologue）和尾声（epilogue）不被计量（unmetered）：
- 序言（prologue）：在序言中，不知道提交交易的账户是否有足够的资金，或者提交交易的用户是否对提交账户拥有权限。由于缺乏这种知识，当prologue被执行时，它需要不被计量。在prologue中失败的交易扣除gas可能会导致未经授权的账户扣除。
- 尾声（epilogue）：负责从提交账户中扣除执行费并分发。因此，即使交易执行已耗尽gas，epilogue也必须运行。同样，我们在扣除提交账户时不希望耗尽gas，因为这将导致在不收取任何交易费用的情况下执行额外的计算。

> In the prologue, it's not known if the submitting account has sufficient funds to cover its gas liability, or if the user submitting the transaction even has authority over the submitting account. Due to this lack of knowledge, when the prologue is executed, it needs to be unmetered. Deducting gas for transactions that fail the prologue could allow unauthorized deductions from accounts.
> The epilogue is in part responsible for debiting the execution fee from the submitting account and distributing it. Because of this, the epilogue must run even if the transaction execution has run out of gas. Likewise, we don't want it to run out of gas while debiting the submitting account as this would cause additional computation to be performed without any transaction fee being charged.

因此，最低交易费`MIN _TXN_FEE`，要能够支付运行`prologue`和`epilogue`的平均成本。

`prologue`执行后，我们已经部分核实了该账户可以支付其gas债务,交易流程的其余部分从"gas tank"加满`max_gas_amount`开始。收取`MIN_TXN_FEE`之后，然后VM执行的每条指令都从"gas tank"中收取费用，直到:

- 第一种情况：交易执行完成后，收取存储交易数据的费用，运行epilogue 部分并扣除执行费用。正常执行完毕，费用被收取，交易结果被持久化在Aptos区块链上
- 第二种情况："gas tank"变空时，抛出`OutOfGas`错误。抛出错误，停止执行，然后，收集交易的总gas负债。

In the former, the fee is collected and the result of the transaction is persisted on the Aptos Blockchain. In the latter, the execution of the transaction stops when the error is raised. After which, the total gas liability of the transaction is collected. No other remnants of the execution are committed other than the deduction in this case.

### 用gas改变交易执行的优先级

交易发起时，根据不同的标准对交易执行的优先级：aptos使用使用标准化的`gas price`来确定交易执行的优先级。

对于非治理交易（non-governance transactions）来说，gas price首先被标准化（normlize）为`Aptos Coins`（通过使用存储在链上的当前gas货币(gas currency)到`Aptos Coins`的转换率来完成），例如，同时有如下两笔交易被提交：
- Bob发送了一笔交易：`gas_price = 10`, `gas_currency` 是 `“BobCoins”`.
- Alice发送了一笔交易：`gas_price = 20`, `gas_currency` 是 `“AliceCoins”`.

假设：`“BobCoins”`到`Aptos Coins`的链上转换率为`2.1`，`“AliceCoins”`到`Aptos Coins`的链上转换率为`1`。那么：
- Bob发送的交易的标准化gas price为：`10 * 2.1 = 21`
- Alice发送的交易的标准化gas price为：`20 * 1 = 20`

所以，Bob的交易优先级币Alice的交易优先级更高。

### 核心设计原则

Aptos和Move的gas设计遵循三个核心原则:

| 设计原则（Design Principle）     | 描述                                                         |
| -------------------------------- | ------------------------------------------------------------ |
| Move是图灵完备的语言             | 因此，确定一个给定的Move程序是否终止不能静态地决定. 但是，确保（1）每个字节码指令都有一个非零的gas消耗；（2）任何程序启动时绑定的gas数量都是有限的 |
| 阻止DDoS攻击并鼓励恰当地使用网络 | 交易的gas使用与该交易的资源消耗相关。gas价格以及交易费用应该随着网络中的竞争而上下波动。在发布时，我们预计gas价格为零或接近零。但是在高度竞争的时期，可以使用gas价格来区分交易的优先级，鼓励在这种时期只发送重要的交易。 |
| 项目的资源使用需要达成共识       | 这意味着计算资源消耗的方法需要具有确定性。这排除了跟踪资源使用的其他方法，例如周期计数器或任何类型的基于时间的方法，因为它们不能保证在节点之间是确定的。跟踪资源使用的方法需要是抽象的。 |



## 证明（Proof）
Aptos区块链使用proofs作为验证区块链数据真实性和正确性的方式。

Aptos区块链中的所有数据都存储在单版本分布式数据库中。每个验证器和FullNode的存储负责将一致同意的交易块及其执行结果保存到数据库中。区块链被表示为一棵不断增长的Merkle树，树的每一片叶子代表区块链执行的一个交易。

区块链执行的所有操作和所有账户状态都可以用密码验证。这些加密的proofs确保验证器节点在状态上一致。通过支持proofs，客户端不需要信任从其接收数据的实体。

例如，如果client从一个账户中取出最新的`n`笔交易，一个证据(proof)可以证明在响应中没有交易被添加、省略或修改。客户端还可以查询帐号的状态，询问是否处理了特定的交易，等等。


## 节点网络与同步

### 节点网络拓扑

Validator节点和FullNodes形成了一个层次结构，validator节点位于根节点，FullNodes可以位于其他任何位置。

Aptos区块链区分了两种类型的FullNodes:
1. 验证器全节点（Validator FullNodes）：Validator FullNodes直接连接到validator nodes，并提供可扩展性和缓解DDoS攻击。
2. 公共全节点（Public FullNodes）：公共全节点连接到验证器全节点(或其他公共全节点)，以获得对Aptos网络的低延迟访问。

![](https://aptos.dev/assets/images/v-fn-network-20283e9f73bf516237c0979d969af1db.svg)


### 独立的网络栈
Aptos区块链支持不同网络拓扑的不同网络堆栈。例如，验证器网络独立于全节点网络。

独立网络栈的优势：
- 不同网络之间的清晰分离。
- 更好地支持安全首选项(e.g., bidirectional vs server authentication).
- 允许隔离发现协议：(验证器节点公共端点的链上发现 vs 私有组织的手动配置)(i.e., on-chain discovery for validator node's public endpoints vs. manual configuration for private organizations).

### 节点同步
Aptos节点通过两种机制同步到Aptos区块链的最新状态:共识(consensus)或状态同步(state synchronization)。**验证器节点将使用这两种机制来保持最新状态，而FullNodes只使用状态同步(state synchronization)。**

例如，当Validator节点第一次联机或重启时(例如，在脱机一段时间后)，它将调用状态同步。一旦验证器更新了区块链的最新状态，它将开始参与共识并完全依赖共识来保持更新。全节点仅仅依赖状态同步来获得并保持最新的区块链状态。

### 状态同步器
每个Aptos节点都包含一个状态同步器组件（https://github.com/aptos-labs/aptos-core/tree/main/state-sync），用于将节点的状态与其对等节点同步。该组件对所有类型的Aptos节点都具有相同的功能:它利用专用的对等网络不断请求和传播区块链数据。

验证器节点在验证器节点网络内分发区块链数据，而全节点依赖于其他全节点(即验证器全节点或公共全节点)。

### 同步API
Aptos节点的状态同步器与其他节点的状态同步器通信，以获取和发送交易块。在这里了解更多关于这是如何工作的规范。

https://github.com/aptos-labs/aptos-core/tree/main/documentation/specifications/state_sync

## 交易与状态
交易(transactions)和状态(states)是Aptos区块链两个核心的基本概念：
- Transactions：交易代表Aptos区块链上账户之间的数据交换(如:`Aptos Coins`或`NFT`)。
- States：状态(即，当前区块链账本状态)代表当前区块链的快照。

**执行交易时，Aptos区块链的状态会发生变化。**

### Transactions
当Aptos区块链客户端提交交易时，他们请求用他们的交易更新账本状态。

区块链上的一笔签名交易包含以下信息:
- Signature: 发送者使用数字签名来验证他们签署了交易(即认证).
- Sender address: 交易发起者的地址.
- Sender public key: 交易发起者的公钥（与签名本交易的私钥对应的公钥）。
- Program: 程序包括一下内容:
    - Move模块和函数名或Move字节码交易脚本
    - 脚本的可选输入列表。对于点对点交易，包含收款人信息和金额。
    - 要发布的Move字节码模块的可选列表。
- Gas price (in specified currency/gas units):发送者愿意支付的每单位gas对应的金额（也就是一个gas多少钱）。gas是支付计算和存储的一种方式。
- Maximum gas amount: 该交易消耗gas数量的上限。
- Gas currency code: 用于支付gas费用的货币代码。指定的货币最终需要按照链上比率换算成`Aptos Coins`
- Sequence number: 这是一个无符号整数，在执行时必须等于发送方的帐号中的`sequence number`（每发送一笔交易其值增1）。
- Expiration time: 一个时间戳，在该时间戳之后，交易将不再有效。

### 账本状态
Aptos区块链的账本状态(或者称为:global state)包括区块链所有账户的状态。

区块链中的每个验证器节点必须知道最新版本的区块链分布式数据库(版本化数据库)的全局状态，才能执行交易。（未同步节点不允许执行交易）

### 版本化数据库
Aptos区块链中的所有数据都保存在一个单一版本的分布式数据库中。版本号为一个无符号64位整数，对应于系统已经执行的交易数量。

此版本化数据库(versioned database)允许验证器节点:
- 根据最新版本的账本状态执行交易
- 响应客户端关于当前版本的查询，响应之前版本历史的查询。

### 交易改变状态

下图展示了交易`T_n`如何将账本状态从`S_n-1`改变到`S_n`:
![TRANSACTIONS CHANGE STATE](https://aptos.dev/assets/images/transactions-7d6b830b61b1b992811649a3e3ec85de.svg)

图中的`F`这是一个确定性的函数。对于特定的初始状态和一个特定的交易，`F`总是返回相同的最终状态。

Aptos使用Move语言实现确定性执行函数`F`。


## 验证器节点
Aptos节点：
- 节点用来跟踪Aptos区块链的状态。
- 客户端通过Aptos节点与区块链进行交互。
- 两种节点类型：
    - Validator nodes：验证器节点
    - FullNodes：全节点

每个Aptos节点都包括如下逻辑组件:
- REST service
- Mempool: 它保存已提交给区块链但尚未达成一致或执行的交易的内存缓冲区。该缓冲区在验证器节点和FullNodes之间复制。
    - FullNode的JSON-RPC服务将交易发送到验证器节点的Mempool。
    - Mempool对交易执行各种检查，以确保交易的有效性并防止DOS攻击。
    - 当一个新交易通过初始验证并被添加到Mempool中时，它将被分发到网络中其他验证器节点的Mempool中。
    - 当一个验证节点暂时成为共识协议中的领导者时，共识协议从内存池中拉出交易，并提出一个新的交易块。该块被广播给其他验证器，并且包含该块中所有交易的总排序。然后每个验证器执行这个块，并提交投票决定是否接受这个新的块提议。
- Consensus (disabled in FullNodes):该组件负责对交易块进行排序并通过与网络中的其他验证器节点一起参与consensus协议来对执行结果达成一致。
- Execution：该组件协调交易块的执行并维护瞬态。对这种短暂状态的一致投票。在consensus将块提交给分布式数据库之前，Execution会维护执行结果的内存表示。Execution使用虚拟机来执行交易。Execution充当系统输入(由交易表示)、存储(提供持久层)和虚拟机(用于执行)之间的粘合层。
- Virtual Machine：用于在每个交易中运行Move程序，并确定执行结果。节点的Mempool使用VM对交易执行验证检查，而Execution使用VM执行交易。
- Storage：将一致同意的交易块及其执行结果保存到本地数据库中。
- State synchronizer：节点使用该组件来“赶上”区块链的最新状态并保持最新。

Aptos-core软件（https://github.com/aptos-labs/aptos-core）可以配置为作为验证器节点或全节点来运行。

![Validator node components](https://aptos.dev/assets/images/validator-444a20a96f90a1d3e9c1d7b84b402f68.svg)

当交易提交到Aptos区块链时，验证器节点运行分布式共识协议，执行交易，并将交易和执行结果存储在区块链上。验证器节点决定将哪些交易添加到区块链中，以及添加的顺序。

Aptos区块链使用拜占庭容错(BFT,Byzantine Fault Tolerance)共识协议在验证器节点间达成共识。验证器节点处理交易，并将它们包含在区块链数据库的本地副本中。这意味着最新的验证器节点总是在本地维护区块链当前状态的副本。

验证器节点通过专用网络直接与其他验证器节点通信。全节点是最终交易历史的外部验证和/或传播资源。它们接收来自对等体的交易，并在本地重新执行它们(与验证器执行交易的方式相同)。FullNodes将重新执行的交易的结果保存到本地存储。这样，他们可以挑战验证者的任何违规行为，并在有人试图重写或修改区块链历史时提供证据。这有助于减少验证器的腐败(corruption)和/或共谋(collusion)。

**AptosBFT共识协议提供了高达三分之一的恶意验证器节点的容错能力。**


# 2 交易（Transactions）

## 交易的生命周期

本部分内容将从具体操作的角度来深入理解Aptos链上交易的生命周期：从一笔交易被提交给全节点（FullNode）到交易最终上链（即交易被提交到Aptos区块链上并确认交易完成）。

### 假定前提

- Alice和Bob在Aptos区块链分别有两个帐号（他们各自对应一个帐号地址）
- Alice的帐号地址上有**110**个Aptos代币
- Alice向Bob发送**10**个Aptos代币
- Alice帐号当前的`sequence number`等于5.（5代表：Alice的账号地址已经发送出了5笔交易）
- Aptos的区块链网络上目前总共有100个验证器（validator）节点：编号从`V1`到 `V100`
- Aptos客户端（例如，自己编写的Python脚本、web应用程序等）将Alice的交易提交给一个全节点的REST服务。此全节点将该笔交易转发给一个验证器全节点（validator FullNode），验证器全节点再将交易转发给签证器`V1`：具体过程如下：
  - Aptos client --> Public FullNode --> Validator FullNode ---> Validator Node (V1)
- 验证器`V1称为本笔交易的**提议者(propser)/领导者(leader)。**

### 客户端构建并提交一笔交易

将客户端构建的原始交易(raw transaction)表示为`Traw5`：

- `Traw5`交易的具体内容：==从Alice的账户中给Bob发送10个Aptos代币==。
- Aptos客户端使用Alice的私钥对交易进行签名，签名后的交易用`T5`表示

`T5`包括下面的内容：

- 原始交易，即`Traw5`的内容
- Alice的公钥（Alice's public key）
- Alice的签名（signature）

原始交易`Traw5`包括如下字段：

| 字段(Fields)       | 描述(Description)                                            |
| ------------------ | ------------------------------------------------------------ |
| Account Address    | Alice的帐号地址：交易发起者的地址                            |
| Move Module        | 交易要执行的具体操作，包括:1. 一个Move字节码点对点`transaction script`;2. `transaction script`的输入列表(本例为，Bob的帐号地址和发送的Aptos代币数量) |
| Maximum gas amount | 交易允许支付的最大gas数量。Gas用于支付交易消耗的计算和存储费用。 |
| Gas price          | Gas价格，即一个 gas 等于多少Aptos代币                        |
| Expiration time    | 交易的过期时间，超过这个期限的交易不会上链                   |
| Sequence number    | `sequence number`表示由本账户提交的（submitted）并且已经上链的（committed）交易数。本例中，Alice帐号地址已经提交了5笔交易（包括`Traw5`，本人认为`sequence number`表示的是下一笔Traw5目前还没有执行）. |
| Chain ID           | Aptos网络标识符，防止跨网络攻击，`devnet`的`ChainID`是3      |

> 关于`transaction script`：
>
> - 用户提交的每一笔交易都包含一个`transaction script`
> - `transaction script`表示客户端需要验证器执行的具体操作：
>   - 具体操作可以是：１.转账操作；２.与链上已发布智能合约（Move模块）的交互
> - `transaction script`是一个任意程序，通过调用模块内的过程，与Aptos区块链全局存储中发布的资源进行交互。它对交易的逻辑进行编码。
> - 一个`transaction script`可以向多个收款人发送资金，调用多个不同模块中的过程。
> - `transaction script`不存储在全局状态中，并且不能被其他`transaction script`调用。这是一个一次性程序。

### 交易的生命周期

本部分介绍`T5`交易的生命周期：从客户端提交交易到链上确认（committed）的整个过程。

![交易生命周期图](https://aptos.dev/assets/images/validator-sequence-9ef2017c79e2949422c93ca446f5e3b9.svg)

一笔交易的生命周期要经历如下5个阶段：

1. Accepting：即全节点从Aptos客户端接受交易
2. Sharing：与其他验证器节点共享交易
3. Proposing：提出区块（Proposing the block）
4. Executing and Consensus：执行区块并达成共识（Executing the block and reaching consensus）
5. Committing：区块上链（Committing the block）

下面详细介绍上面的5个阶段。

#### Stage1：接受客户端的交易

| 过程描述                                                     | Aptos节点组件交互              |
| ------------------------------------------------------------ | ------------------------------ |
| 1.**Client → REST service**: a. 客户机将T5交易提交给全节点的REST Service; b. 全节点将接收到的交易加入自己的内存池`mempool`,同时交易转发给网络中的其他节点; c. 交易最终将被转发到运行一台验证器全节点（validator Fullnode）上的`mempool`; d. 验证器全节点再将交易转发给一台验证器节点（validator node，本例中为`V1`）. | 1. REST Service                |
| 2.**REST service → Mempool**: 全节点的`REST Service`将`T5`交易传送给验证器节点`V1`的`mempool` | 2. REST Service, 1. Mempool    |
| 3. **Mempool → Virtual Machine (VM)**:Mempool将使用虚拟机(VM)组件来执行交易验证，例如签名验证、帐户余额验证、基于`sequence number`的抗重放攻击。 | 4. Mempool, 3. Virtual Machine |

#### Stage2：其他验证器节点共享交易

| 过程描述                                                     | 对应的Aptos节点组件交互 |
| ------------------------------------------------------------ | ----------------------- |
| 4. **Mempool**: `mempool`将 `T5`交易保存在内存中缓冲区（`memory buffer`）中。`Mempool`内可能已经包含了从Alice地址发出的多笔交易。 | Mempool                 |
| 5. **Mempool → Other Validators**: 验证器节点`V1`将使用共享内存池协议`shared-mempool protocol`, 将其内存池中的交易 (including T5) 共享给其他验证器节点，并且把从其他验证器节点接收到的交易放入自己的`mempool` | 2. Mempool              |

#### Stage3：提出区块（Proposing the block）

| 过程描述                                                     | 对应的Aptos节点组件交互  |
| ------------------------------------------------------------ | ------------------------ |
| 6. **Consensus → Mempool**:  验证器节点`V1`作为`T5`交易的提议者(leader), 它从自己的`mempool`中拉取一个交易区块，并且通过共识组件（consensus component）将此区块作为一项提议复制给其他验证器节点。it will pull a block of transactions from its mempool and replicate this block as a proposal to other validator nodes via its consensus component. | 1. Consensus, 3. Mempool |
| 7. **Consensus → Other Validators**: :`V1`的共识组件负责协调所有验证器就提议区块内的交易顺序达成共识。 | 2. Consensus             |

#### Stage4：执行区块并达成共识

| 过程描述                                                     | 对应的Aptos节点组件交互          |
| ------------------------------------------------------------ | -------------------------------- |
| 8. **Consensus → Execution**: 交易区块（块内包括`T5`和其他交易）被共享给执行组件（execution component） | 3. Consensus, 1. Execution       |
| 9. **Execution → Virtual Machine**: 执行组件管理VM中交易的执行。这里的执行是speculatively 发生在区块中的交易达成一致之前。Note that this execution happens speculatively before the transactions in the block have been agreed upon. | 2. Execution, 3. Virtual Machine |
| 10. **Consensus → Execution**: 执行完区块中的交易之后，执行组件把这些区块中的交易（包括`T5`）追加到`Merkle accumulator（of ledger history）`。这是一个在内存/临时（in-memory/temporary）版本的Merkle累加器。执行这些交易的提议/推测结果的必要部分被返回到共识组件以达成一致（The necessary part of the proposed/speculative result of executing these transactions is returned to the consensus component to agree on.） 从 "consensus" 到 "execution" 的箭头表示：执行交易的请求是由共识组件发起的。 | 3. Consensus                     |
| 11. **Consensus → Other Validators**: V1 (the consensus leader) attempts to reach consensus on the proposed block's execution result with the other validator nodes participating in consensus. `V1`(the consensus leader)尝试和其他参与共识的验证器节点，就`V1`提议区块的执行结果达成共识。 | 3. Consensus                     |

#### Stage5：区块上链

| 过程描述                                                     | 对应的Aptos节点组件交互                              |
| ------------------------------------------------------------ | ---------------------------------------------------- |
| 12. **Consensus → Execution**, **Execution → Storage**: 如果`V1`提议的区块的执行结果被达成了共识（结果即被其他验证器节点认可），并且签名的验证器节点数目达到法定的投票数目，`V1`的执行组件从`speculative execution cache`中读取建议区块的所有执行结果，这些结果与建议区块中的所有交易信息都被持久化存储。If the proposed block's execution result is agreed upon and signed by a set of validators that have the quorum of votes, validator V1's execution component reads the full result of the proposed block execution from the speculative execution cache and commits all the transactions in the proposed block to persistent storage with their results. | 4. Consensus, 3. Execution, 4. Execution, 3. Storage |

至此，Alice的帐号中拥有100Aptos代币，她的`sequence number`将变成6. 如果`T5`交易被Bob重放的话，这笔交易将被拒绝，因为Alice帐号的`sequence number = 6`比`T5`交易中的`sequence number = 5`要大。

### Aptos节点内相关组件之间的交互

本部分对于理解Aptos系统内部的运作机制有帮助，如果希望参与Aptos开源项目的话，这部分内容将会很有用。

这里，我们假设**一个Aptos客户端提交一笔交易`TN`到一个验证器`VX`。**

交易生命周期中使用到的Aptos节点的核心组件如下：

- FullNode
  - REST Service
- Validator node
  - Mempool
  - Consensus
  - Execution
  - Virtual Machine
  - Storage

#### REST Service

客户端发出的任何请求：

1. 首先，客户端的任何请求首先发送给给FullNode的REST服务。
2. 然后，FullNode将提交的交易转发给validator FullNode
3. 接着，validator FullNode再将其发送到validator node `VX`。

##### 1.Client → REST Service

客户端向Aptos FullNode的REST服务提交交易。

##### 2.REST Service → Mempool

REST服务将交易转发给validator FullNode，然后validator Fullnode将交易发送给validator node `VX`的`mempool`。

只有当`TN`的`sequence number`大于或等于发送方账户的当前`sequence number`时，`mempool`才会接受交易`TN`(注意，未满足`sequence number`条件的交易不会被传递到consensus组件)。

##### 3. REST Service → Storage

客户端查询Aptos区块链的链上数据时，FullNode的REST Service将访问持久化存储组件。例如，查询账户余额。

#### Virtual Machine(VM)

![Figure 1.2 Virtual machine](https://aptos.dev/assets/images/virtual-machine-988ed2a790611c6c93ac7400cbd90aee.svg)

Move 虚拟机负责验证和执行交易脚本（`transaction script`）。The Move VM verifies and executes transaction scripts written in Move bytecode.

##### 1.Virtual Machine → Storage

当内存池`mempool`通过调用虚拟机的`VM::ValidateTransaction`接口来验证一笔交易时，虚拟机从存储中加载交易发送方的帐号，并且完成验证工作，验证要做很多工作，下面仅列出部分内容（完整的工作列表参看：https://github.com/aptos-labs/aptos-core/tree/main/documentation/specifications/move_adapter）：

- 检查已签名交易的输入签名是否正确(拒绝未正确签名的交易)。 
- 检查发送者的帐户验证密钥是否与公钥(对应于用于签署交易的私钥)的散列相同。
- 验证交易的序列号是否大于或等于发送方帐户的当前序列号。完成这一检查可以防止对发送者的账户重放相同的交易。 
- 验证签名交易中的程序（program）是否是畸形的（malformed），因为畸形的程序不能被虚拟机执行。
- 验证发送方的帐户余额**大于等于**`max gas amount * gas price`，确保帐号可以为本笔交易所使用的资源付费。

##### 2. Execution → Virtual Machine

执行组件利用VM（`通过VM::ExecuteTransaction()`）执行交易。

**执行交易和更新账本状态并将结果保存在存储器中是不同的**，理解这一点很重要。例如：在共识期内，交易`TN`首先被执行，被当作**在区块上达成共识的一个必要步骤而被执行**：如果最终和其他验证器节点在`交易顺序和交易执行结果`达成了共识，最后才将结果永久的保存在存储中，同时还要更新账本状态。

##### 3. Mempool → Virtual Machine

当`mempool`接收到一笔交易时(可以从其他验证器节点`shared mempool`得到，也可以是从`REST Service`接收到到的交易)，`mempool`通过调用VM的`VM::ValidateTransaction()`接口来验证这笔交易。

实现细节请参考VM的README文档：https://github.com/diem/move/tree/main/language/move-vm

#### Mempool

mempool是一个共享的缓冲区，用来保存哪些等待(`waiting`)执行的交易。当一笔新的交易添加进`mempools`时，mempool将交易共享给其他验证器节点。为了降低网络开销，在共享交易时，验证器只负责把自己的交易传递给其他验证器。当验证器从其他验证器节点的mempool接收到一笔交易时，把这笔交易添加进`mempool`.

![Figure 1.3 Mempool](https://aptos.dev/assets/images/mempool-6d5b50789261f471c1a3abf3a29468e0.svg)

##### 1. REST Service → Mempool

- 从客户机接收到事务后，REST服务将交易代理给一个验证器FullNode。然后，交易被发送到验证器节点的内存池。
- 仅当TN的序列号大于或等于发送者账户的当前序列号时，验证者节点VX的mempool才接受发送者账户的交易TN。

##### 2. Mempool → Other validator nodes

- 验证器节点`VX`的内存池与同一网络上的其他验证器共享交易`TN`。 

- 其他验证器将它们各自的内存池中的事务与`VX`的内存池共享。

##### 3. Consensus → Mempool

- 当交易被转发到一个验证器节点时，一旦该验证器节点成为此交易的领导者，它的共识组件将从它的`mempool`中拉出一个交易块，并将所提议的区块复制到其他验证器。这样做是为了在提议的区块中的交易排序和交易执行结果上达成共识。
- 注意，交易`TN`仅仅是包括在一个新提议的共识区块中，但是它并不保证`TN`最终能够被持久存储在Aptos块链分布式数据库中。

##### 4. Mempool → VM

当`mempool`从其他验证器接收到一笔交易时，`mempool`调用VM上的`VM::ValidateTransaction()`来验证交易。

实现细节参考`Mempool`的README文档：https://github.com/aptos-labs/aptos-core/tree/main/documentation/specifications/mempool.

#### Consensus

共识组件负责对交易区块进行排序，并通过与网络中的其他验证者一起参与共识协议来对执行结果达成一致。

![Figure 1.4 Consensus](https://aptos.dev/assets/images/consensus-8ed6339c4de3081c4b3c3db8522827f9.svg)

##### 1. Consensus → Mempool

当验证者`VX`是领导者/提议者(leader/proposer)时，`VX`的共识组件通过`mempool::GetBlock()`从其Mempool中拉出一个事务块，并形成一个提议的交易区块。

##### 2. Consensus → Other Validators

如果`VX`是提议者/领导者(proposer/leader)，其共识组件将提议的交易区块复制给其他验证者。

##### 3. Consensus → Execution, Consensus → Other Validators

- 为了执行交易区块，consensus组件与执行组件进行交互。共识组件通过`execute:ExecuteBlock()`来执行一个交易区块(参考：Consensus → execution)
- 在建议的区块中执行交易后，执行组件将这些交易的执行结果作为响应送给共识组件。
- 共识组件对执行结果进行签名，并尝试与其他验证者就该结果达成一致。

##### 4. Consensus → Execution

如果有足够多的验证者投票赞成同一个执行结果，`VX`的共识组件通过`Execution::CommitBlock()`通知执行，这个区块已经准备好，可以提交了。

实现细节请参考共识README文档：https://github.com/aptos-labs/aptos-core/tree/main/documentation/specifications/consensus

####　Execution

![Figure 1.5 Execution](https://aptos.dev/assets/images/execution-8bd7359d714d6a5eebccee2cd70fd209.svg)

执行组件协调一个交易块（通常含多笔交易）的执行，==并维护一个可以一致表决的临时状态(maintains a transient state that can be voted upon by consensus)==。如果这些交易成功，它们将被提交到存储。

##### 1. Consensus → Execution

- 共识组件通过调用执行组件的`Execution::ExecuteBlock()`接口来执行一个交易块。
- 执行维护一个“暂存区”(`scratchpad`)，其中保存了Merkle累加器(https://aptos.dev/reference/glossary/#merkle-accumulator)相关部分的内存副本。这些信息用于计算Aptos区块链当前状态的根哈希。
- 将当前状态的根散列与关于所提议的块中的交易的信息相结合，以确定累加器的新的根散列。这是在持久存储任何数据之前完成的，以确保在法定数量的验证者达成一致之前，不会存储任何状态或交易。The root hash of the current state is combined with the information about the transactions in the proposed block to determine the new root hash of the accumulator. This is done prior to persisting any data, and to ensure that no state or transaction is stored until agreement is reached by a quorum of validators.
- 执行组件计算推测的根哈希，然后VX的共识组件对该根哈希进行签名，并尝试与其他验证者就该根哈希达成一致。Execution computes the speculative root hash and then the consensus component of VX signs this root hash and attempts to reach agreement on this root hash with other validators.

 ##### 2. Execution → VM

当consensus通过`execute::execute block()`请求执行交易块时，执行使用VM来确定执行交易块的结果。

When consensus requests execution to execute a block of transactions via `Execution::ExecuteBlock()`, execution uses the VM to determine the results of executing the block of transactions.

##### 3. Consensus → Execution

如果验证器的法定人数对块执行结果达成一致，每个验证器的一致性组件通过`Execution::CommitBlock()`通知其执行组件该块已准备好提交。对执行组件的调用将包括验证者的签名，以提供他们同意的证据。

##### 4. Execution → Storage

执行从其“暂存区”( “scratchpad”)中获取值，并通过`Storage::SaveTransactions()`将它们发送到存储区进行持久化。然后，执行从“暂存”中删除不再需要的旧值(例如，无法提交的并行块,parallel blocks that cannot be committed)。

#### Storage

存储组件将达成共识的交易区块及其执行结果保存到Aptos块链。当参与共识的验证器节点，有超过法定人数`(2f+1)`的节点达成一致时，交易区块(包括交易`TN`)才通过存储组件被永久保存。共识（Agreement）必须包括以下所有内容：

- 区块中的所有交易
- 交易的执行顺序 
- 区块中交易的执行结果

如何将交易追加（append）到表示Aptos区块链的数据结构里，请参考：Merkle accumulator，https://aptos.dev/reference/glossary#merkle-accumulator。

![Figure 1.6 Storage](https://aptos.dev/assets/images/storage-4060c8a24c78205696646ee2f6a4467b.svg)

##### 1. VM → Storage

当`mempool`调用`VM::ValidateTransaction()`验证一笔交易时，`VM::ValidateTransaction()`会从存储系统中加载交易发送者的帐号，并在交易上完成只读验证检查。

##### 2. Execution → Storage

当共识组件调用`Execution::ExecuteBlock()`时，执行组件从存储中读取当前状态，并结合内存中的“暂存”数据（in-memory “scratchpad” data）来确定执行结果。

##### 3. Execution → Storage

一旦一个交易块（含有多笔交易）达成共识，执行组件将通过调用存储组件的`storage::SaveTransactions()`接口将区块永久的保存到链上，同时也会将同意这个交易块的所有验证器节点的签名存储到链上。此数据块的“暂存”内存中的数据将被传递到更新存储并持久化这些交易(The in-memory data in “scratchpad” for this block is passed to update storage and persist the transactions.)。当存储被更新时，与这些交易相关的每个账户对应的`sequence number`都将递增1。

> **注意**：对于源自该账户提交的每一笔交易，都会使Aptos区块链该账户的序列号`sequence number`递增1。

##### 4.REST Service → Storage

对于从区块链读取信息的客户端查询，REST服务直接与存储交互以读取所请求的信息。

## 与Aptos区块链交互

Aptos用Move虚拟机来执行相关操作。虽然许多区块链实现了一组本地操作，但Aptos将所有的操作都委托给Move来执行（包括:帐户创建、资金转账和发布Move模块）。

为了支持这些操作，构建在Move之上的区块链必须提供一个与区块链交互的框架(类似于计算机的操作系统或最小可用的功能集)。在本节中，我们将讨论这些**通过Aptos框架的脚本函数暴露出来**的功能集。

本指南(搭配：`Move module tutorial`，下面的**第一个Move模块** )将提供在Aptos区块链上构建丰富应用所需的最少量信息。

> **注意**:Aptos框架正在密集的开发中，这里的内容可能不是最新的。最新的框架可以在源代码中找到：https://github.com/aptos-labs/aptos-core/tree/main/aptos-move/framework/aptos-framework/sources。

Aptos框架内为用户提供的核心功能包括：

- 发送和接收测试币`TestCoin`
- 创建新用户
- 发布一个新的Move模块

> 注:本文档假设读者已经熟悉提交交易，在下文的**发起第一笔转账交易**有详细介绍）

### 发送和接收TestCoin代币

当提交和执行交易时需要消耗`TestCoin`代币。可以通过开发者网络水龙头（Devnet Faucet）来获取`TestCoin`代币。具体的示例参看下文的：**发起第一笔转账交易**

让区块链去执行一个交易的有效载荷（payload）如下所示：

```json
{
  "type": "script_function_payload",
  "function": "0x1::TestCoin::transfer",
  "type_arguments": [],
  "arguments": [
    "0x737b36c96926043794ed3a0b3eaaceaf",
    "1000",
  ]
}
```

上面的示例，告诉Move虚拟机去执行`0x1::TestCoin::transfer`脚本。第一个参数是接收帐号的地址：`0x737b36c96926043794ed3a0b3eaaceaf`；第二个参数是发送代币的数量：1000。发出代币的地址就是发出本交易的帐号地址。

### 创建一个新账号

告诉区块链创建新账户的有效载荷示例如下：

```json
{
  "type": "script_function_payload",
  "function": "0x1::AptosAccount::create_account",
  "type_arguments": [],
  "arguments": [
    "0x0c7e09cd9185a27104fa218a0b26ea88",
    "0xaacf87ae9d8a5e523c7f1107c668cb28dec005933c4a3bf0465ffd8a9800a2d900",
  ]
}
```

上面的示例中，告诉Move虚拟机去执行脚本`0x 1::AptosAccount::create _ account`。第一个参数是要创建的帐户的地址，第二个参数是身份验证密钥pre-image(在基础知识中的**帐号**一节有提到)。对于单一签名者身份验证，是与0字节连接的公钥`(pubkey_A | 0x00)`。这是为了防止账户地址被抢占。该指令的执行验证了认证密钥的后16个字节与16个字节的账户地址相同。我们正在积极改进这个API，以支持32字节的帐户地址，这将消除对土地掠夺或帐户操纵（`around land grabbing or account manipulation`）的担忧。

### 发布一个新的Move模块

告诉区块链发布一个新的Move模块的有效载荷示例如下：

```json
"type": "module_bundle_payload",
"modules": [
    {"bytecode": "0x..."},
],
```

上面的示例，告诉虚拟机在发送者帐号地址下发布Move模块字节码，详见**第一Move模块**。

**注意**：Move字节码必须指定与发送方帐户相同的地址，否则交易将被拒绝。==例如，假设帐户地址为`0xe110`，则需要更新Move模块，因为模块`0xe110::Message`、模块`0xbar::Message`将被拒绝(For example, assuming account address `0xe110`, the Move module would need to be updated as such `module 0xe110::Message`, `module 0xbar::Message` would be rejected.)。或者，可以使用别名地址，例如模块`HelloBlockchain::Message`，但是HelloBlockchain别名需要在Move.toml文件中更新为0xe110(Alternatively an aliased address could be used, such as `module HelloBlockchain::Message` but the `HelloBlockchain` alias would need to updated to `0xe110` in the `Move.toml` file. )。我们正在与Move团队合作，并计划将一个编译器集成到我们的REST接口中来缓解这个问题。==


# 3 上手指南（Tutorials）

## 发起第一笔转账交易
https://aptos.dev/tutorials/your-first-transaction

启动本机全节点，并确认节点同步完成：

- cargo run -p aptos-node --release -- -f ./public_full_node.yaml
- target/release/aptos-node -f ./public_full_node.yaml
- curl 127.0.0.1:9101/metrics 2> /dev/null | grep aptos_state_sync_version | grep type

涵盖：交易生成、提交和验证链上交易。

- 创建帐号
- REST接口Wrapper
- Faucet接口Wrapper
- 构建应用程序，执行并验证

官方提供Python、Rust、Typescript三种编程语言的示例代码。

Python示例代码：

- https://github.com/aptos-labs/aptos-core/tree/main/developer-docs-site/static/examples/python
- `<source doc directory>/developer-docs-site/static/examples/python/`

### 第1步：创建帐号

每个aptos帐号有一个唯一的帐号地址（也就是钱包地址）。

每个帐号地址对应：1. 一对`公钥-私钥`对（public key - private key）；2. 认证密钥（authentication key ）。

```python
class Account:
    """Represents an account as well as the private, public key-pair for the Aptos blockchain."""

    def __init__(self, seed: bytes = None) -> None:
        if seed is None:
            self.signing_key = SigningKey.generate()
        else:
            self.signing_key = SigningKey(seed)

    def address(self) -> str:
        """Returns the address associated with the given account"""

        return self.auth_key()[-32:]

    def auth_key(self) -> str:
        """Returns the auth_key for the associated account"""

        hasher = hashlib.sha3_256()
        hasher.update(self.signing_key.verify_key.encode() + b'\x00')
        return hasher.hexdigest()

    def pub_key(self) -> str:
        """Returns the public key for the associated account"""

        return self.signing_key.verify_key.encode().hex()
```

测试生成帐号的代码：
```
root@i-pxsnz3pq:~/aptos/python# ipython3
Python 3.8.10 (default, Nov 26 2021, 20:14:08)
Type 'copyright', 'credits' or 'license' for more information
IPython 7.13.0 -- An enhanced Interactive Python. Type '?' for help.

In [1]: from first_transaction import Account

In [2]: alice = Account()

In [3]: alice.address()
Out[3]: '4ace5af003af0af6893cef1496f428a6'

In [4]: alice.pub_key()
Out[4]: '23efdd2d4590896563ffdfedd57f4bfc4ac30257ae6c9eb9cee031d5613042a9'

In [5]: alice.auth_key()
Out[5]: '290a39dd3c2fdb669cafbfbe87a9f9b64ace5af003af0af6893cef1496f428a6'

In [6]:

```

### 第2步：REST API接口

Aptos提供了与区块链交互的REST接口。

-  https://fullnode.devnet.aptoslabs.com/spec.html
- https://127.0.0.1:8080/spec.html
- https://public_ip:8080/spec.html

虽然可以直接调用REST接口实现相关操作，但是使用wrpper更方便。官方示例代码提供了一个简单的REST接口wrapper，以方便的实现：从FullNode检索账本数据（包括帐号和帐号资源数据），构造由JSON格式表示的签名交易等等。

```python
class RestClient:
    """A wrapper around the Aptos-core Rest API"""

    def __init__(self, url: str) -> None:
        self.url = url
```

#### 2.1 读取帐号信息

```python
    def account(self, account_address: str) -> Dict[str, str]:
        """Returns the sequence number and authentication key for an account"""

        response = requests.get(f"{self.url}/accounts/{account_address}")
        assert response.status_code == 200, f"{response.text} - {account_address}"
        return response.json()

    def account_resources(self, account_address: str) -> Dict[str, Any]:
        """Returns all resources associated with the account"""

        response = requests.get(f"{self.url}/accounts/{account_address}/resources")
        assert response.status_code == 200, response.text
        return response.json()
```

#### 2.2 向链上提交一笔交易

```python
    def generate_transaction(self, sender: str, payload: Dict[str, Any]) -> Dict[str, Any]:
        """Generates a transaction request that can be submitted to produce a raw transaction that
        can be signed, which upon being signed can be submitted to the blockchain. """

        account_res = self.account(sender)
        seq_num = int(account_res["sequence_number"])
        txn_request = {
            "sender": f"0x{sender}",
            "sequence_number": str(seq_num),
            "max_gas_amount": "1000",
            "gas_unit_price": "1",
            "gas_currency_code": "XUS",
            "expiration_timestamp_secs": str(int(time.time()) + 600),
            "payload": payload,
        }
        return txn_request

    def sign_transaction(self, account_from: Account, txn_request: Dict[str, Any]) -> Dict[str, Any]:
        """Converts a transaction request produced by `generate_transaction` into a properly signed
        transaction, which can then be submitted to the blockchain."""

        res = requests.post(f"{self.url}/transactions/signing_message", json=txn_request)
        assert res.status_code == 200, res.text
        to_sign = bytes.fromhex(res.json()["message"][2:])
        signature = account_from.signing_key.sign(to_sign).signature
        txn_request["signature"] = {
            "type": "ed25519_signature",
            "public_key": f"0x{account_from.pub_key()}",
            "signature": f"0x{signature.hex()}",
        }
        return txn_request

    def submit_transaction(self, txn: Dict[str, Any]) -> Dict[str, Any]:
        """Submits a signed transaction to the blockchain."""

        headers = {'Content-Type': 'application/json'}
        response = requests.post(f"{self.url}/transactions", headers=headers, json=txn)
        assert response.status_code == 202, f"{response.text} - {txn}"
        return response.json()

    def transaction_pending(self, txn_hash: str) -> bool:
        response = requests.get(f"{self.url}/transactions/{txn_hash}")
        if response.status_code == 404:
            return True
        assert response.status_code == 200, f"{response.text} - {txn_hash}"
        return response.json()["type"] == "pending_transaction"

    def wait_for_transaction(self, txn_hash: str) -> None:
        """Waits up to 10 seconds for a transaction to move past pending state."""

        count = 0
        while self.transaction_pending(txn_hash):
            assert count < 10, f"transaction {txn_hash} timed out"
            time.sleep(1)
            count += 1
```

#### 2.3 应用程序逻辑

```python
    def account_balance(self, account_address: str) -> Optional[int]:
        """Returns the test coin balance associated with the account"""

        resources = self.account_resources(account_address)
        for resource in resources:
            if resource["type"] == "0x1::TestCoin::Balance":
                return int(resource["data"]["coin"]["value"])
        return None

    def transfer(self, account_from: Account, recipient: str, amount: int) -> str:
        """Transfer a given coin amount from a given Account to the recipient's account address.
        Returns the sequence number of the transaction used to transfer."""

        payload = {
            "type": "script_function_payload",
            "function": "0x1::TestCoin::transfer",
            "type_arguments": [],
            "arguments": [
                f"0x{recipient}",
                str(amount),
            ]
        }
        txn_request = self.generate_transaction(account_from.address(), payload)
        signed_txn = self.sign_transaction(account_from, txn_request)
        res = self.submit_transaction(signed_txn)
        return str(res["hash"])
```

### 第3步：水龙头接口

水龙头（Faucet）负责给帐号发放测试token。用户利用测试币来支付gas费、给其他帐号转账等操作。如果指定的帐号不存在，水龙头可以自动创建帐号。

```python
class FaucetClient:
    """Faucet creates and funds accounts. This is a thin wrapper around that."""

    def __init__(self, url: str, rest_client: RestClient) -> None:
        self.url = url
        self.rest_client = rest_client

    def fund_account(self, pub_key: str, amount: int) -> None:
        """This creates an account if it does not exist and mints the specified amount of
        coins into that account."""

        txns = requests.post(f"{self.url}/mint?amount={amount}&pub_key={pub_key}")
        assert txns.status_code == 200, txns.text
        for txn_hash in txns.json():
            self.rest_client.wait_for_transaction(txn_hash)
```

### 第4步：执行应用程序并验证执行结果

```python
if __name__ == "__main__":
    rest_client = RestClient(TESTNET_URL)
    faucet_client = FaucetClient(FAUCET_URL, rest_client)

    # Create two accounts, Alice and Bob, and fund Alice but not Bob
    alice = Account()
    bob = Account()

    print("\n=== Addresses ===")
    print(f"Alice: {alice.address()}")
    print(f"Bob: {bob.address()}")

    faucet_client.fund_account(alice.pub_key(), 1_000_000)
    faucet_client.fund_account(bob.pub_key(), 0)

    print("\n=== Initial Balances ===")
    print(f"Alice: {rest_client.account_balance(alice.address())}")
    print(f"Bob: {rest_client.account_balance(bob.address())}")

    # Have Alice give Bob 10 coins
    tx_hash = rest_client.transfer(alice, bob.address(), 1_000)
    rest_client.wait_for_transaction(tx_hash)

    print("\n=== Final Balances ===")
    print(f"Alice: {rest_client.account_balance(alice.address())}")
    print(f"Bob: {rest_client.account_balance(bob.address())}")
```

输出结果：

```
=== Addresses ===
Alice: e26d69b8d3ff12874358da6a4082a2ac
Bob: c8585f009c8a90f22c6b603f28b9ed8c

=== Initial Balances ===
Alice: 1000000000
Bob: 0

=== Final Balances ===
Alice: 999998957
Bob: 1000
```

输出结果解释：

- 首先，生成了两个帐号（注意此时帐号在区块链上不存在）
- 利用水龙头：给帐号申请测试币（同时创建帐号）
- Alice给Bob转账1000个测试币，gas费为43个测试币

可以查询链上数据验证上述操作：

- https://fullnode.devnet.aptoslabs.com/accounts/e26d69b8d3ff12874358da6a4082a2ac/resources
- https://aptos-explorer.netlify.app/account/c8585f009c8a90f22c6b603f28b9ed8c



## 第一个Move模块
内容包括：
- Move模块的编写、编译、测试
- 将Move模块发布到链上
- 初始化Move模块，与链上Move模块交互。


官方提供了Python、Rust和Typescript三种语言版本，我们选择Python语言。

复用了一部分“发起第一笔转账交易”中的代码（`first_transaction.py`）。

源码链接（hello_blockchain.py）：https://github.com/aptos-labs/aptos-core/tree/main/developer-docs-site/static/examples/python

### 第1步：编写和测试Move模块
#### 1.1 准备aptos源码和开发环境

```
git clone https://github.com/aptos-labs/aptos-core.git
cd aptos-core
./scripts/dev_setup.sh
source ~/.cargo/env
```

aptos-core源码目录下的`move-examples`子目录，能够使我们不需要下载额外的资源而方便的build和测试Move模块。

关于Move语言入门：https://github.com/diem/move/tree/main/language/documentation/tutorial

#### 1.2 Move模块代码

进入`<aptos_core>/aptos-move/move-examples`目录:

```
root@VM-0-8-ubuntu:~# cd /data/aptos-core/aptos-move/move-examples/
root@VM-0-8-ubuntu:/data/aptos-core/aptos-move/move-examples# ls
build  Cargo.toml  Move.toml  sources  src  tests
root@VM-0-8-ubuntu:/data/aptos-core/aptos-move/move-examples# ls sources/
HelloBlockchain.move  HelloBlockchainTest.move
root@VM-0-8-ubuntu:/data/aptos-core/aptos-move/move-examples#
```

查看 Move 代码`sources/HelloBlockchain.move`：

> 这是一个Move模块，能够在用户下创建一个String类型的资源，用户仅能设置自己的资源，不能设置其他用户的资源。

- struct MessageHolder：资源
- set_message:是一个脚本函数，允许通过提交交易的方式来调用此脚本函数：
    - 被调用时，判断当前用户是否存在资源MessageHolder，
    - 如果不存在则创建和MessageHolder资源并将传入的message保存到此资源。
    - 如果判断当前用户资源MessageHolder已存在则重新写入新传入的内容。

Move模块的完整代码：
```move
module HelloBlockchain::Message {
    use Std::ASCII;
    use Std::Errors;
    use Std::Event;
    use Std::Signer;

    struct MessageHolder has key {
        message: ASCII::String,
        message_change_events: Event::EventHandle<MessageChangeEvent>,
    }

    struct MessageChangeEvent has drop, store {
        from_message: ASCII::String,
        to_message: ASCII::String,
    }

    /// There is no message present
    const ENO_MESSAGE: u64 = 0;

    public fun get_message(addr: address): ASCII::String acquires MessageHolder {
        assert!(exists<MessageHolder>(addr), Errors::not_published(ENO_MESSAGE));
        *&borrow_global<MessageHolder>(addr).message
    }

    public(script) fun set_message(account: signer, message_bytes: vector<u8>)
    acquires MessageHolder {
        let message = ASCII::string(message_bytes);
        let account_addr = Signer::address_of(&account);
        if (!exists<MessageHolder>(account_addr)) {
            move_to(&account, MessageHolder {
                message,
                message_change_events: Event::new_event_handle<MessageChangeEvent>(&account),
            })
        } else {
            let old_message_holder = borrow_global_mut<MessageHolder>(account_addr);
            let from_message = *&old_message_holder.message;
            Event::emit_event(&mut old_message_holder.message_change_events, MessageChangeEvent {
                from_message,
                to_message: copy message,
            });
            old_message_holder.message = message;
        }
    }

    #[test(account = @0x1)]
    public(script) fun sender_can_set_message(account: signer) acquires MessageHolder {
        let addr = Signer::address_of(&account);
        set_message(account,  b"Hello, Blockchain");

        assert!(
          get_message(addr) == ASCII::string(b"Hello, Blockchain"),
          ENO_MESSAGE
        );
    }
}
```
#### 1.3 测试Move模块

- get_message：获取当前的message内容
- sender_can_set_message: 测试设置message内容是否成功。

测试Move模块的命令：
```
root@VM-0-8-ubuntu:~# cd /data/aptos-core/aptos-move/move-examples/
root@VM-0-8-ubuntu:/data/aptos-core/aptos-move/move-examples# cargo test --package move-examples
    Finished test [unoptimized + debuginfo] target(s) in 0.54s
     Running unittests (/data/aptos-core/target/debug/deps/move_examples-95eb31b6cc56b72f)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running tests/move_unit_tests.rs (/data/aptos-core/target/debug/deps/move_unit_tests-30c935ba45b64f86)

running 1 test
Running Move unit tests
[ PASS    ] 0xe110::MessageTests::sender_can_set_message
[ PASS    ] 0xe110::Message::sender_can_set_message
Test result: OK. Total tests: 2; passed: 2; failed: 0
```

> `sources/HelloBlockchainTest.move`里有另一种测试方法。

### 第2步：Move模块的发布与交互
将Move模块发布（或部署）到链上，并与之交互。

#### 2.1 将Move模块发布到链上

> 将Move模块发布到链上和提交一笔交易到链上的区别：设置的payload类型不同

```python
class HelloBlockchainClient(RestClient):

    def publish_module(self, account_from: Account, module_hex: str) -> str:
        """Publish a new module to the blockchain within the specified account"""

        payload = {
            "type": "module_bundle_payload",
            "modules": [
                {"bytecode": f"0x{module_hex}"},
            ],
        }
        txn_request = self.generate_transaction(account_from.address(), payload)
        signed_txn = self.sign_transaction(account_from, txn_request)
        res = self.submit_transaction(signed_txn)
        return str(res["hash"])
```

#### 2.2 读取用户的MessageHolder资源
Move模块将被发布到一个地址上，这个地址即`contract_address`。
- `contract_address`：和发布此合约的用户地址相同。
之前的TestCoin的合约地址为`0x1`。

```python
    def get_message(self, contract_address: str, account_address: str) -> Optional[str]:
        """ Retrieve the resource Message::MessageHolder::message """

        resources = self.account_resources(account_address)
        for resource in resources:
            if resource["type"] == f"0x{contract_address}::Message::MessageHolder":
                return resource["data"]["message"]
        return None
```

#### 2.3 修改资源
通过交易的方式调用链上的Move模块。

为了实现初始化和操纵资源，Move模块必须暴露脚本函数（script functions），脚本可以被提交的一笔交易调用。

> 虽然 REST 接口可以显示字符串，但由于 JSON 和 Move 的限制，它无法确定参数是字符串还是十六进制编码的字符串。因此，交易参数始终假定后者。因此，在此示例中，消息被编码为十六进制字符串。

```python
    def set_message(self, contract_address: str, account_from: Account, message: str) -> str:
        """ Potentially initialize and set the resource Message::MessageHolder::message """

        payload = {
            "type": "script_function_payload",
            "function": f"0x{contract_address}::Message::set_message",
            "type_arguments": [],
            "arguments": [
                message.encode("utf-8").hex(),
            ]
        }
        txn_request = self.generate_transaction(account_from.address(), payload)
        signed_txn = self.sign_transaction(account_from, txn_request)
        res = self.submit_transaction(signed_txn)
        return str(res["hash"])
```

### 第3步：Move模块的初始化和交互

- `https://github.com/aptos-labs/aptos-core/tree/main/developer-docs-site/static/examples/python`
- `<aptos_core_dir>/developer-docs-site/static/examples/python/`

```
pip3 install -r requirements.txt.
python3 hello_blockchain.py Message.mv

=== Addresses ===
Alice: a52671f10dc3479b09d0a11ce47694c0
Bob: ec6ec14e4abe10aaa6ad53b0b63a1806

=== Initial Balances ===
Alice: 10000000
Bob: 10000000

Update the module with Alice's address, build, copy to the provided path, and press enter.
(提示：将编译后的Move文件复制到当前python代码目录下，之后再按下回车。编译方法见下)

=== Testing Alice ===
Publishing...
Initial value: None
Setting the message to "Hello, Blockchain"
New value: Hello, Blockchain

=== Testing Bob ===
Initial value: None
Setting the message to "Hello, Blockchain"
New value: Hello, Blockchain
```

编译Move代码：
- 将上面输出的alice地址复制到Move.towl中（`<aptos-core-dir>/aptos-move/move-examples/Move.towl`），替换`0xe110`
- `cd <aptos-core-dir>/aptos-move/move-examples/`
- `cargo run -- srouces`
- `cp build/Examples/bytecode_modules/Message.mv <python_source_dir>`


链上验证Alice和Bob的资源：
- Alice通过REST interface：https://fullnode.devnet.aptoslabs.com/accounts/a52671f10dc3479b09d0a11ce47694c0/
- Bob通过区块链浏览器：https://explorer.devnet.aptos.dev/account/ec6ec14e4abe10aaa6ad53b0b63a1806


## 运行本地测试网

### 准备aptos开发环境

下载源码并准备aptos开发环境

```
git clone https://github.com/aptos-labs/aptos-core.git
cd aptos-core
./scripts/dev_setup.sh
source ~/.cargo/env
```

如果由于网路原因导致rust相关软件安装失败（例如，卡在了`Updating git repository https://github.com/facebookincubator/cargo-guppy`），需要设置使用国内源，创建配置文件：`/root/.cargo/config`,然后重新执行：`./scripts/dev_setup.sh`

```
[source.crates-io]
replace-with = 'tuna'

[source.tuna]
registry = "https://mirrors.tuna.tsinghua.edu.cn/git/crates.io-index.git"
```

### 启动节点

1. 进入aptos-core源码目录下，启动节点命令：`cargo run -p aptos-node -- --test`（首次运行时由于需要下载和编译源码，过程比较长）。
2. 启动成功后，会打印config path 

`cargo run -p aptos-node -- --test`命令会从创世账本状态运行aptos-node，如果想重用账本状态（ledger state），使用命令：`cargo run -p aptos-node -- --test --config <config-path>`


### 使用Docker

https://aptos.dev/tutorials/run-a-local-testnet#using-docker

### 与本地测试验证网络交互

本地测试验证网络启动后，会看到如下的输出信息（Aptos的命令行工具需要使用这些信息）：

```
root@ubuntu-12:/disk2/aptos/aptos-core# cargo run -p aptos-node -- --test
    Finished dev [unoptimized + debuginfo] target(s) in 0.79s
     Running `target/debug/aptos-node --test`
Entering test mode, this should never be used in production!
Completed generating configuration:
        Log file: "/tmp/26a5e8cc3f5fee9f929e4b242d72d7cd/validator.log"
        Config path: "/tmp/26a5e8cc3f5fee9f929e4b242d72d7cd/0/node.yaml"
        Aptos root key path: "/tmp/26a5e8cc3f5fee9f929e4b242d72d7cd/mint.key"
        Waypoint: 0:1383de3ece0cc2153f7269df8c39c1cfc3e6eaf26f85e949b509795abb4c3523
        ChainId: TESTING
        REST API endpoint: 0.0.0.0:8080
        Stream-RPC enabled!
        FullNode network: /ip4/0.0.0.0/tcp/7180

Aptos is running, press ctrl-c to exit
```

- ==Aptos root key path== - The root (also known as a mint or faucet) key controls the account that can mint. Available in the docker compose folder under aptos_root_key.
- ==Waypoint== - a verifiable checkpoint into the blockchain (available in the docker compose folder under waypoint.txt)
- ==REST endpoint== - http://127.0.0.1:8080.
- ==ChainId== - uniquely distinguishes this chain from other chains. 





## 运行全节点

操作系统环境：ubuntu 20.04 x86-64 LTS

可以运行FullNodes来验证状态，并同步到Aptos区块链。任何人都可以运行FullNodes。全节点通过相互查询或直接查询验证器来复制区块链的完整状态。

全节点REST访问链接： `http://localhost:8080`

### 硬件需求

产品级全节点推荐硬件配置：
- CPU: Intel Xeon Skylake or newer, 4 cores
- Memory: 8GiB RAM

产品级全节点推荐硬件配置：
- CPU: 2 cores
- Memory: 4GiB RAM

两种方式配置Public FullNode:
1. Aptos-core源代码方式
2. Docker方式

### 源码方式

1. 准备开发者环境
```
git clone https://github.com/aptos-labs/aptos-core.git
cd aptos-core
./scripts/dev_setup.sh
source ~/.cargo/env
```

如果由于网路原因导致rust相关软件安装失败，需要设置使用国内源，创建配置文件：`/root/.cargo/config`,然后重新执行：`./scripts/dev_setup.sh`

`/root/.cargo/config`的内容如下：

```
[source.crates-io]
replace-with = 'tuna'

[source.tuna]
registry = "https://mirrors.tuna.tsinghua.edu.cn/git/crates.io-index.git"
```

### 启动节点

2. Checkout devnet分支： `git checkout origin/devnet`
3. 准备配置文件

```
root@VM-ubuntu:~/aptos-core# cp config/src/config/test_data/public_full_node.yaml .
root@VM-ubuntu:~/aptos-core# wget https://devnet.aptoslabs.com/genesis.blob
```
修改配置文件：`public_full_node.yaml`:
- `data_dir`(设置数据存放目录)
- `waypoint`: 将文件`https://devnet.aptoslabs.com/waypoint.txt`里的内容替换到`from_config`字段中。
- `genesis_file_location`：创世文件需要从`https://devnet.aptoslabs.com/genesis.blob`下载

配置文件示例：
```
base:
    # Update this value to the location you want the node to store its database
    data_dir: "./data"
    role: "full_node"
    waypoint:
        # Update this value to that which the blockchain publicly provides. Please regard the directions
        # below on how to safely manage your genesis_file_location with respect to the waypoint.
        from_config: "0:5add8f76a230dbf6c7b619d5318b6e8b8ebd8e1f7a1422c217f4a60880757eb8"

execution:
    # Update this to the location to where the genesis.blob is stored, prefer fullpaths
    # Note, this must be paired with a waypoint. If you update your waypoint without a
    # corresponding genesis, the file location should be an empty path.
    genesis_file_location: "/data/aptos-core/genesis.blob"

full_node_networks:
    - discovery_method: "onchain"
      # The network must have a listen address to specify protocols. This runs it locally to
      # prevent remote, incoming connections.
      listen_address: "/ip4/127.0.0.1/tcp/6180"
      network_id: "public"
      # Define the upstream peers to connect to
      seeds:
        {}

api:
    enabled: true
    address: 127.0.0.1:8080
```

可以根据实际情况进行修改配置文件的内容。

具体配置可以参考aptos-core源码目录下`docker/compose/public_full_node/public_full_node.yaml`的内容。

4. 运行全节点接入devnet：`cargo run -p aptos-node --release -- -f ./public_full_node.yaml`
    - `--release`表示将编译release版本到目录`target/release/aptos-node`. 
    - 如果要编译debug版，以获得更多的调试信息的话，去掉`--release`，重新运行。

```
root@VM-0-8-ubuntu:/data/aptos-core# cargo run -p aptos-node --release -- -f ./public_full_node.yaml
    Finished release [optimized + debuginfo] target(s) in 5.49s
     Running `target/release/aptos-node -f ./public_full_node.yaml`
```

### Docker方式

Docker安装和运行全节点相对简单，前提是熟悉docker的基本概念和常用命令。

1. 安装docker和docker-compose
   1. ubuntu 20.04中使用apt 安装的docker-compose版本太低，需要下载官网的最新版
   2. https://github.com/docker/compose/releases
2. 创建目录，例如：public_fullnode，cd进入新创建的目录下。
3. 下载docker_compose配置文件（docker-compose.yaml），aptos-core的public全节点配置文件（public_full_node.yaml）。
   1. https://github.com/aptos-labs/aptos-core/tree/main/docker/compose/public_full_node/docker-compose.yaml
   2. https://github.com/aptos-labs/aptos-core/tree/main/docker/compose/public_full_node/public_full_node.yaml
4. 下载genesis文件和waypoint文件
   1. https://devnet.aptoslabs.com/genesis.blob
   2. https://devnet.aptoslabs.com/waypoint.txt
5. 使用命令：`docker-compose up`来运行容器。如果出错：`apt install gnupg2 pass`

### 理解和验证节点正确性

#### 初始同步

##### 查询节点同步情况

命令：`curl 127.0.0.1:9101/metrics 2> /dev/null | grep aptos_state_sync_version | grep type`

- `aptos_state_sync_version{type="committed"}`: 最新的区块链版本
- `aptos_state_sync_version{type="highest"}`: 已知的最高或最新版本，通常与`target`相同
- `aptos_state_sync_version{type="synced"}`: 存储中可用的最新区块链版本，它可能没有帐本信息支持
- `aptos_state_sync_version{type="target"}`: 状态同步的当前`target`账本版本

```
root@VM-0-8-ubuntu:~# curl 127.0.0.1:9101/metrics 2> /dev/null | grep aptos_state_sync_version | grep type
aptos_state_sync_version{type="committed"} 0
aptos_state_sync_version{type="highest"} 661800
aptos_state_sync_version{type="synced"} 23000
aptos_state_sync_version{type="target"} 661315

root@VM-0-8-ubuntu:~# curl 127.0.0.1:9101/metrics 2> /dev/null | grep aptos_state_sync_version | grep type
aptos_state_sync_version{type="committed"} 0
aptos_state_sync_version{type="highest"} 663613
aptos_state_sync_version{type="synced"} 55000
aptos_state_sync_version{type="target"} 661315

root@VM-0-8-ubuntu:~# curl 127.0.0.1:9101/metrics 2> /dev/null | grep aptos_state_sync_version | grep type
aptos_state_sync_version{type="committed"} 658039
aptos_state_sync_version{type="highest"} 658039
aptos_state_sync_version{type="synced"} 658039
aptos_state_sync_version{type="target"} 658040

root@VM-0-8-ubuntu:~# curl 127.0.0.1:9101/metrics 2> /dev/null | grep aptos_state_sync_version | grep type
aptos_state_sync_version{type="committed"} 659186
aptos_state_sync_version{type="highest"} 659186
aptos_state_sync_version{type="synced"} 659186
aptos_state_sync_version{type="target"} 659187
```

更详细的日志信息可以在控制台终端查看。

##### 查看连接数

`curl 127.0.0.1:9101/metrics 2> /dev/null | grep aptos_connections`

```
root@VM-0-8-ubuntu:/data# curl 127.0.0.1:9101/metrics 2> /dev/null | grep aptos_connections
# HELP aptos_connections Number of current connections and their direction
# TYPE aptos_connections gauge
aptos_connections{direction="inbound",network_id="Public",peer_id="63fd325d",role_type="full_node"} 0
aptos_connections{direction="outbound",network_id="Public",peer_id="63fd325d",role_type="full_node"} 1
```

连接数应该大于0（outbound），==否则==需要添加上游种子节点。

##### 查看账本数据库大小

```
root@VM-0-8-ubuntu:/data/aptos-core# cat public_full_node.yaml
base:
    # Update this value to the location you want the node to store its database
    data_dir: "./data"
    role: "full_node"
    waypoint:
        # Update this value to that which the blockchain publicly provides. Please regard the directions
        # below on how to safely manage your genesis_file_location with respect to the waypoint.
        from_config: "0:5add8f76a230dbf6c7b619d5318b6e8b8ebd8e1f7a1422c217f4a60880757eb8"

execution:
    # Update this to the location to where the genesis.blob is stored, prefer fullpaths
    # Note, this must be paired with a waypoint. If you update your waypoint without a
    # corresponding genesis, the file location should be an empty path.
    genesis_file_location: "/data/aptos-core/genesis.blob"

full_node_networks:
    - discovery_method: "onchain"
      # The network must have a listen address to specify protocols. This runs it locally to
      # prevent remote, incoming connections.
      listen_address: "/ip4/127.0.0.1/tcp/6180"
      network_id: "public"
      # Define the upstream peers to connect to
      seeds:
        {}
api:
    enabled: true
    #address: 127.0.0.1:8080
    address: 0.0.0.0:8888
root@VM-0-8-ubuntu:/data/aptos-core# du -sh ./data
2.4G    ./data
```

#### 添加上游种子节点

官方开发者网络的 validator fullnode最多只接受1000个连接，如果网络流量过大，我们的Fullnode可能无法与其建立连接，出现“NoAvailablePeers”的错误信息。

如果出现这种情况的话，需要在Fullnode配置文件中添加新的`upstream peers`。官方也提供了如下一些额外的Fullnode地址（也可以使用社区成员已经运行的全节点）：

```
seeds:
  4d6a710365a2d95ac6ffbd9b9198a86a:
      addresses:
      - "/dns4/pfn0.node.devnet.aptoslabs.com/tcp/6182/ln-noise-ik/bb14af025d226288a3488b4433cf5cb54d6a710365a2d95ac6ffbd9b9198a86a/ln-handshake/0"
      role: "Upstream"
  52173b436ae1809df4a5fcfc67f8fc61:
      addresses:
      - "/dns4/pfn1.node.devnet.aptoslabs.com/tcp/6182/ln-noise-ik/7fe8523388084607cdf78ff40e3e717652173b436ae1809df4a5fcfc67f8fc61/ln-handshake/0"
      role: "Upstream"
  476222516fdc55869d2b649c614d965b:
      addresses:
      - "/dns4/pfn2.node.devnet.aptoslabs.com/tcp/6182/ln-noise-ik/f6b135a59591677afc98168791551a0a476222516fdc55869d2b649c614d965b/ln-handshake/0"
      role: "Upstream"
```

> 将上面的内容复制到public_full_node.yaml中，重新启动节点。

### 高级指南

这部分介绍关于节点配置的高级部分内容，对节点配置进行更多的配置：

- 为新部署的Fullnode创建静态网络ID(identity)
- 检索其他节点的公共网络ID
- Start a node with or without a static network identity

#### 为全节点创建静态ID

Fullnodes启动时自动创建随机的网络ID（一对PeerId 和 Public Key ），这对于常规的fullnode非常有用，但是如果需要做一些权限设置（if you need another node to allowlist you or provide specific permissions），或者如果希望自己的全节点始终以相同的网络ID来运行Fullnode，那么就需要为节点创建静态网络ID。

1. 使用源码编译`aptos-operational-tool`生成private_key

```
$ git clone https://github.com/aptos-labs/aptos-core.git
$ cd aptos-core
$ ./scripts/dev_setup.sh
$ source ~/.cargo/env
$ cargo run -p aptos-operational-tool -- <command> <args>
```

```
root@VM-0-8-ubuntu:/data/aptos-core# cargo run -p aptos-operational-tool -- generate-key --encoding hex --key-type x25519 --key-file ./private-key.txt
   Compiling aptos-rest-client v0.0.0 (/data/aptos-core/crates/aptos-rest-client)
   Compiling aptos-operational-tool v0.1.0 (/data/aptos-core/config/management/operational)
    Finished dev [unoptimized + debuginfo] target(s) in 41.47s
     Running `target/debug/aptos-operational-tool generate-key --encoding hex --key-type x25519 --key-file ./private-key.txt`
{
  "Result": "Success"
}
root@VM-0-8-ubuntu:/data/aptos-core# cat private-key.txt
C0F2E4FB6530E0A0805B8E6EFBD074F8E1B6B9DE7437FF3A371D6F962187F163
```

上面运行密钥生成器，输出十六进制编码的静态x25519 PrivateKey。

2. 使用docker运行`aptos-operational-tool`生成private_key的方法

```
$ docker run -i aptoslab/tools:devnet sh -x  # 首次运行要连网下载docker镜像
$ aptos-operational-tool <command> <arg>
```

官方提供了一个包含aptos-operational-tool工具的docker容器。

3. 获得公共网络ID

```
root@VM-0-8-ubuntu:/data/aptos-core# cargo run -p aptos-operational-tool -- extract-peer-from-file --encoding hex --key-file ./private-key.txt --output-file ./peer-info.yaml
    Finished dev [unoptimized + debuginfo] target(s) in 0.45s
     Running `target/debug/aptos-operational-tool extract-peer-from-file --encoding hex --key-file ./private-key.txt --output-file ./peer-info.yaml`
{
  "Result": {
    "587605be1a76748e906d81315fb28c0a": {
      "addresses": [],
      "keys": [        "525a38168de6bc2d21a5960b43a91d5b587605be1a76748e906d81315fb28c0a"
      ],
      "role": "Downstream"
    }
  }
}
root@VM-0-8-ubuntu:/data/aptos-core# cat peer-info.yaml
---
587605be1a76748e906d81315fb28c0a:
  addresses: []
  keys:
    - 525a38168de6bc2d21a5960b43a91d5b587605be1a76748e906d81315fb28c0a
  role: Downstream
```

如上输出所示：

- `"587605be1a76748e906d81315fb28c0a"`：peer id
- `ca3579457555c80fc7bb39964eb298c414fd60f81a2f8eedb0244ec07a26e575`：基于上面的private key生成的publick key

生成的`peer-info.yaml`文件（含有当前节点的public identity）的用途：如果你想连接到特定的上游全节点，但是这个上游全节点只允许已知的网络ID节点才能连接。**上游节点要把我们自己节点的Public ID加入进许可列表中，同时，我们的节点要用生成的静态网络ID来启动**

#### 用静态网络ID启动节点

将下面的配置内容加入到`public_full_node.yaml`文件中：

```
full_node_networks:
- network_id: "public"
  discovery_method: "onchain"
  identity:
    type: "from_config"
    key: "<PRIVATE_KEY>"
    peer_id: "<PEER_ID>"
```

具体的示例：

```
full_node_networks:
- network_id: "public"
  discovery_method: "onchain"
  identity:
    type: "from_config"
    key: "C0F2E4FB6530E0A0805B8E6EFBD074F8E1B6B9DE7437FF3A371D6F962187F163"
    peer_id: "587605be1a76748e906d81315fb28c0a"
```

#### 让其他全节点连接到自己的节点

一旦设置了以静态网络ID启动节点，那么就可以允许其他节点连接到我们的节点，进而连入如devnet。（节点默认使用随机的动态网络ID，每次启动节点都会变化）

将自己的节点信息告知其他节点，格式如下：

```
<Peer_ID>:
  addresses:
  # with DNS
  - "/dns4/<DNS_Name>/tcp/<Port_Number>/ln-noise-ik/<Public_Key>/ln-handshake/0"
  role: Upstream
<Peer_ID>:
  addresses:
  # with IP
  - "/ip4/<IP_Address>/tcp/<Port_Number>/ln-noise-ik/<Public_Key>/ln-handshake/0"
  role: Upstream
```

具体示例：

```
4d6a710365a2d95ac6ffbd9b9198a86a:
  addresses:
  - "/dns4/pfn0.node.devnet.aptoslabs.com/tcp/6182/ln-noise-ik/bb14af025d226288a3488b4433cf5cb54d6a710365a2d95ac6ffbd9b9198a86a/ln-handshake/0"
  role: "Upstream"
4d6a710365a2d95ac6ffbd9b9198a86a:
  addresses:
  - "/ip4/100.20.221.187/tcp/6182/ln-noise-ik/bb14af025d226288a3488b4433cf5cb54d6a710365a2d95ac6ffbd9b9198a86a/ln-handshake/0"
  role: "Upstream"
```

### 使用最新devnet版本更新全节点

当官方发布了最新的devnet版本时，需要手动更新我们自己的全节点配置，使其与官方的devnet同步。

步骤如下：

1. 停止全节点
2. 删除区块链数据目录（在配置文件public_full_node.yaml中指定）
3. 删除 `genesis.blob`和`waypoint.txt`两个文件。
4. 重新下载`genesis.blob` 和 `waypoint.txt`
   1. https://devnet.aptoslabs.com/genesis.blob
   2. https://devnet.aptoslabs.com/waypoint.txt
5. 更新`public_full_node.yaml`文件
6. 重启节点
7. 检查同步状态，对比`status dashboard`(每分钟同步一次状态): https://status.devnet.aptos.dev/
