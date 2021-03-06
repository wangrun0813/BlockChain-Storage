

# Ethereum

## 一：区块结构

![](./picture/ethereum-structure.png)



### 1.1 区块头

![](./picture/ethereum-structure-2.jpg)

1. ParentHah:父块的哈希值
2. Nonce：一个随机数，使得块头哈希满足POW需求
3. Timestamp: 此区块成立时的unix的时间戳 
4. Ommershash：当前区块ommers列表的Hash值（ ommer就是一个区块的父区块与当前区块父区块的父区块是相同的）
5. Beneficiary：块打包手续费的受益人，也称矿工（接收挖此区块费用的账户地址 ）
6. Logsbloom：由日志信息组成的一个Bloom过滤器 （1.1.1节具体解释） 
7. Difficulty:POW的难度值
8. extraData：与此区块相关的附加数据
9. Number: 当前区块的计数（创世纪块的区块序号为0，对于每个后续区块，区块序号都增加1）
10. GasLimit:Gas限制
11. GasUsed:交易消耗的Gas（此区块中交易所用的总gas量）
12. Mixhash：一个Hash值，当与nonce组合时，证明此区块已经执行了足够的计算 
13. stateRoot：状态树根节点的Hash值
14. transactionsRoot：包含此区块所列的所有交易的树的根节点Hash值
15. receiptsRoot：包含此区块所列的所有交易收据的树的根节点Hash值



#### 1.1.1logsbloom

![](./picture/ethereum-structure-6.jpg)

###### A: a transaction receipt contains the following elements:

1:The post-transaction state: a trie structure holding the state after the execution. Encoded as a byte array.

​               (blockHash: String, 32 Bytes - hash of the block where this transaction was in.
​                blockNumber: Number - block number where this transaction was in.
​                transactionHash: String, 32 Bytes - hash of the transaction.
​                transactionIndex: Number - integer of the transactions index position in the block.
​               from: String, 20 Bytes - address of the sender.
​               to: String, 20 Bytes - address of the receiver. null when its a contract creation transaction.)

2:Gas used: the total amount of gas used.

3:Logs: the set of logs entries created upon transaction execution.  //交易执行可能有很多logs

4:The bloom filter: it is created based on the information found int he logs. Logs entries are reduced are reduced to 256 bytes hashes, which are embedded in the block header as the logs bloom.

###### B: Log Entry

黄皮书里面对日志记录的形式化定义。

O代表我们的日志记录，Oa代表logger的地址，Oto,Ot1代表日志的Topics， Od代表时间。

![](./picture/ethereum-structure-7.jpg)

Oa是20个字节，Ot是32个字节，Od是很多字节

定义了一个布隆过滤器函数M，用来把一个日志对象转换成256字节的hash

![](./picture/ethereum-structure-8.jpg)

以太坊允许日志可以跟踪各种交易和信息。一个合约可以通过定义“事件”来显示的生成日志。

一个日志的实体包含：

（1）记录器的账户地址

（2）代表本次交易执行的各种事件的一系列主题以及与这些事件相关的任何数据

（3）日志被保存在bloom过滤器中，过滤器高效的保存了无尽的日志数据。

收据中的布隆过滤器就是所有的日志的布隆过滤器输出的并集。

同时区块头中的logBloom，就是所有的收据的布隆过滤器的并集。

以太坊中的Events和Logs基本上算是同一个概念。Solidity和web3.js中称为Events，以太坊黄皮书中称为Logs。你可以理解为：以太坊通过Logs实现Events（事件）功能。智能合约代码通过LOG 将日志写入区块链中。 

###### C:topic

![](./picture/ethereum-structure-10.jpg)

从上面的例子中，我们使用`event`关键字定义一个事件，参数列表中为要记录的日志参数的名称及类型。

日志中存储的不同的索引事件就叫不同的主题(topic)。比如，事件定义，event transfer(address indexed _from, address indexed _to, uint value)有三个主题，第一个主题为默认主题，即事件签名transfer(address,address,uint256)，但如果是声明为anonymous的事件，则没有这个主题；另外两个indexed的参数也会分别形成两个主题，可以分别通过_from，_to主题来进行过滤。如果数组，包括字符串，字节数据做为索引参数，实际主题是对应值的Keccak-256哈希值。

###  1.2 区块体

![](./picture/ethereum-structure-3.jpg)


#### 1.2.1 transaction：

![](./picture/ethereum-structure-4.jpg)

​        有两种类型的交易：**消息通信(message calls)**和**合约创建(contract creations)(也就是交易产生一个新的以太坊合约)**。

1：nonce：此交易的发送者已发送过的交易数

2：gasPrice：发送者愿意支付执行交易所需的每个gas的Wei数量 

3：gasLimit：发送者愿意为执行交易支付gas数量的最大值。这个数量被设置之后在任何计算完成之前就会被提前扣掉 

4：to：接收者的地址。在合约创建交易中，合约账户的地址还没有存在，所以值先空着 

5：value：从发送者转移到接收者的Wei数量。在合约创建交易中，value作为新建合约账户的开始余额 

6：v,r,s：用于产生标识交易发生着的签名

7：data（可选域，只有在消息通信中存在）：消息通话中的输入数据(也就是参数)。例如，如果智能合约就是一个域名注册服务，那么调用合约可能就会期待输入域例如域名和IP地址





## 二：存储的过程

​     区块和交易等数据最终都是存储在leveldb数据库中的，数据库的存储位置在datadir/geth/chaindata中。在core/database_util.go中封装了所有与区块存储和读取相关的代码，通过这些代码可以弄清楚区块、交易等数据结构在数据库中是如何存储的。

### 2.1区块存储

区块存储时将区块头和区块体分开存储。

#####    key前缀

   key中的前缀可以用来区分数据的类型，在core/rawdb/scheme.go中定义了各种前缀：

```
 headerPrefix = []byte("h") // headerPrefix + num (uint64 big endian) + hash -> header

 tdSuffix = []byte("t") // headerPrefix + num (uint64 big endian) + hash + tdSuffix -> td

 numSuffix = []byte("n") // headerPrefix + num (uint64 big endian) + numSuffix -> hash

 blockHashPrefix = []byte("H") // blockHashPrefix + hash -> num (uint64 big endian)

 bodyPrefix = []byte("b") // bodyPrefix + num (uint64 big endian) + hash -> block body

 blockReceiptsPrefix = []byte("r") // blockReceiptsPrefix + num (uint64 big endian) + hash -> block receipts

 lookupPrefix = []byte("l") // lookupPrefix + hash -> transaction/receipt lookup metadata

 bloomBitsPrefix = []byte("B") // bloomBitsPrefix + bit (uint16 big endian) + section (uint64 big endian) + hash -> bloom bits
```

   其中headerPrefix定义了区块头key的前缀为h，bodyPrefix定义了区块体key的前缀为b。

#####  区块头存储

​      存储区块头的函数:

![](./picture/ethereum-structure-5.jpg)

​     它是先对区块头进行RLP编码，encodeBlockNumber将区块号转换成大端格式，然后组装key。这里先向数据库中存储一条区块hash->区块号 的记录，最后将区块头的RLP编码写到数据库中。

```
   headerPrefix + num (uint64 big endian) + hash -> rlpEncode(header)
```

   其中key由区块头前缀、区块号（uint64大端格式）、区块hash构成，value是区块头的RLP编码。



#####  区块体存储

​     将区块中的交易数据和叔块头信息进行 RLP 编码从而生成存储值value；

​     将数据类型前缀，编码后的区块高度和区块哈希拼接生成 key；

​     将存储至db数据库中。

```
   bodyPrefix + num (uint64 big endian) + hash -> rlpEncode(block body)
```

   其中key由区块体前缀、区块号（uint64大端格式）、区块hash构成，value是区块体的RLP编码。



###  2.2 交易存储

交易主要在数据库中仅存储交易的Meta信息。

Meta信息结构体：

```
type TxLookupEntry struct {

   BlockHash common.Hash  //块的hash
   BlockIndex uint64  //块号
   Index uint64   //块上第几笔交易

}
```

key："l"+交易hash

value：TxLookupEntry {区块hash，区块高度，交易位于区块的索引} ，Meta的RLP编码

```
// WriteTxLookupEntries stores a positional metadata for every transaction from// a block, enabling hash based transaction and receipt lookups.
func WriteTxLookupEntries(db ethdb.Putter, block *types.Block) error {
// Iterate over each transaction and encode its metadata
for i, tx := range block.Transactions() {
entry := TxLookupEntry{
BlockHash: block.Hash(),
BlockIndex: block.NumberU64(),
Index: uint64(i),
}
data, err := rlp.EncodeToBytes(entry)
if err != nil {
return err
}
if err := db.Put(append(lookupPrefix, tx.Hash().Bytes()...), data); err != nil {
return err
}
}
return nil
}
```

这里，在将交易meta入库时，会遍历块上的所有交易，并构造交易的meta信息，进行RLP编码。然后以"l"+交易hash为key，meta为value进行存储。

![](./picture/ethereum-structure-17.jpg)

从图中可以看出，MPT中是以交易在区块中的索引的RLP编码作为key，存储交易数据的RLP编码。

事实上交易在LeveDB中并不是单独存储的，而是存储在区块的Body中。

以b + block index + block hash作为关键字就可以唯一确定某个区块的Body所在的位置。

另外，为了能够快速查询某笔交易的数据，在数据库中还存储了每笔交易的索引信息，称为TxLookupEntry。TxLookupEntry中包含了block index和block hash用于定位区块Body，同时还包含了该笔交易在区块Body中的索引位置。

### 2.3 Receipts的存储

![](./picture/ethereum-structure-18.jpg)

交易回执的存储和交易类似，区别是交易回执是单独存储到LevelDB中的，以r为前缀。
另外，由于交易回执和交易是一一对应的，因此也可以通过TxLookupEntry快速定位交易回执所在的位置，加速交易回执的查找。

### 2.4 State DB 存储

![](./picture/ethereum-structure-11.jpg)

以太坊中有两种不同的数据类型：永久数据和暂时数据。永久数据的例子就是转账。一旦转账确认，就会在区块链中记录；然后就再也不可以更改。暂时数据的例子就是特定以太坊账户地址的余额。账户的余额就会存储在状态树中，并且当有特定账户转账的时候，就会改变。永久数据是有意义的，就好像挖矿转账，暂时数据，就例如账户余额，应该被分开存储。以太坊会使用数据树结构来管理数据。

#### 2.4.1 账户状态

账户状态（stateObject）有四个组成部分，不论账户类型是什么，都存在这四个组成部分：

```
type Account struct {
    Nonce    uint64
    Balance  *big.Int
    Root     common.Hash // merkle root of the storage trie
    CodeHash []byte
}
```

1. nonce：如果账户是一个外部拥有账户，nonce代表从此账户地址发送的交易序号。如果账户是一个合约账户，nonce代表此账户创建的合约序号

2. balance： 此地址拥有Wei的数量。1Ether=10^18Wei

3. storageRoot： Merkle Patricia树的根节点Hash值。Merkle树会将此账户存储内容的Hash值进行编码，默认是空值

4. codeHash：此账户EVM代码的hash值。对于合约账户，就是被Hash的代码并作为codeHash保存。对于外部拥有账户，codeHash域是一个空字符串的Hash值

#### 2.4.2 世界状态

StateDB中存储了很多stateObject，而每一个stateObject则代表了一个以太坊账户，包含了账户的地址、余额、nonce、合约代码hash等状态信息。所有账户的当前状态在以太坊中被称为“世界状态”，在每次挖出或者接收到新区块时需要更新世界状态。

为了能够快速检索和更新账户状态，StateDB采用了两级缓存机制，参见下图：

![](./picture/ethereum-structure-15.jpg)

- 第一级缓存以map的形式存储stateObject

- 第二级缓存以MPT的形式存储

- 第三级就是LevelDB上的持久化存储

当上一级缓存中没有所需的数据时，会从下一级缓存或者数据库中进行加载。

以太坊的全局状态就是由账户地址和账户状态的一个映射组成。这个映射被保存在一个叫做Merkle Patricia树的数据结构中。

这棵树要求存在里面的值（value）都有一个对应的key。从树的根节点开始，key会告诉你顺着哪个子节点可以获得对应的值，这个值存在叶子节点。在以太坊中，key/value是地址和与2地址相关联的账户之间状态的映射，包括每个账户的balance, nonce, codeHash和storageRoot（storageRoot自己就是一颗树）。

![](./picture/ethereum-structure-13.jpg)


![](./picture/ethereum-structure-12.png)

状态树————是唯一和独特的。

在以太坊中，只有唯一的状态树。 这个状态树会实时更新。 状态树包含秘钥和每个账户的价值对，这些是在以太坊网络上。 秘钥是单个160字节的认证器（以太坊账户的地址）。

状态树的“数值”是通过对以太坊账户以下账户细节的编译得出的： -nonce -余额 -storageRoot -codeHash。

storageRoot ————智能合约数据的存储（存储的是智能合约执行后修改的变量值）。

![](./picture/ethereum-structure-14.jpg)

- 对于基本数据类型，key = 合约中的变量声明位置（从0开始）

- 对于map类型，key = SHA3(map中的关键字，变量声明位置)，也就是把map中的关键字和变量声明位置拼在一起成为一个64字节的[]byte，然后计算hash值

- value不管在哪种情况下都存储变量的实际值

  ![](./picture/ethereum-structure-16.jpg)
