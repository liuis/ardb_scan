| author                                                       | time      | version |                        |
| ------------------------------------------------------------ | --------- | ------- | ---------------------- |
| [ping.li@conflux-chain.org](mailto:ping.li@conflux-chain.org) | 2019.6.14 | v0.0.1  | mongodb sharing schema |

# MongoDB sharding schema desigin 

[TOC]

## 零.conflux_scan需求：

1. 根据区块hash查找区块
2. 根据交易hash 查找交易
3. 根据区块链hash，查找这个区块包含的所有的交易
4. 查找排好序的区块链列表
5. 根据账号地址查询账号所有的交易
6. 矿工挖到的区块，矿工地址
7. 最新区块难度总值
8. 最新的区块链的交易总数，TPS
9. Privot Chain
10. Epoch info

## 壹.Redis 的schema 

### 一.Block   ->HMSET ：key :"blocks:${blockHash}"

#### 1.Block Detail, for block hash query:

​	key: "blocks:${blockHash}"
​	type: Hashes
​	key-value format: {
​			    "deferredReceiptsRoot": "0x42c7a763c5fb8d2342b1a75b2d214f83212eca6bb9ad432502b40e73b1b2b22d",
​			    "deferredStateRoot": "0x7acf7512bb021303ac4517ea0140c53ab6fd35b923ff55292a58a11dbb61f79e",
​			    "difficulty": "0x77397115",
​			    "epochNumber": "0x15817",
​			    "gasLimit": "0xb2d05e00",
​			    "hash": "0x6f5358834b127e1d0fb7e9055db64fa5fb7276804291f1554d0a5487782ef506",
​			    "height": "0x15817",
​			    "miner": "0x0000000000000000000000000000000000000008",
​			    "nonce": "0x6ccbf7cbe080f3a4",
​			    "parentHash": "0x889a47fd6433af97a9d29a29878cd6e0698ddfef16ea6e1ac736eff5e10e12ba",
​			    "refereeHashes": [
​			      "0x661cd2771f13f3790ffc70eb118c7ad2ecc8de48f8f7fc07ed9b923c160f0da7"
​			    ],
​			    "size": "0x0",
​			    "timestamp": "0x5cb81529",
​			    "transactionsRoot": "0x1dcc4de8dec75d7aab85b567b6ccd41ad312451b948a7413f0a142fd40d49347"
​			    "isPivot": "0x1"
​			}

#### 	2.events:

​		insert: inserted when grow a new epoch
​			1. attach isPivot
​			2. attach transactionCount && newTransactionCount, delete transactions
​			3. serialize referees
​			4. hmset key object
​		modify: should'n be modified before deleted
​		delete: deleted when pop an epoch
​		get: attach missing fields with null

### 二.Transaction  -> HMSET ：key :"blocks:${blockHash}"

#### 1.TrasnactionDetail, for transaction hash query:

 	key: "transactions:${txHash}"
 	type: Hashes
 	key-value format: {
				"blockHash":
				"contractCreated":
				"data": "0x",
				"from": "0x5510949f549c6f251bf9db1aa51674abc5b1f6db",
				"gas": "0x989680",
				"gasPrice": "0x2",
				"hash": "0x19a447a350fd7f35455bf6e29ee922973eb49bea6dd76dbbd1a19fbf86ce2c0f",
				"nonce": "0x7",
				"r": "0xc20e60663863f397c92e009ed9232255c6a2001ddfdcdcb2405cd9c203a41703",
				"s": "0x9d14f7437897d59104a4a06573ecaae39ad943b327631d3d1e72a80b26b1281",
				"v": "0x0",
				"to": null,
				"transactionIndex": "0x0",
			    "timestamp": "0x5cb81529",
				"value": "0x0",
				"firstBlockHash": "0x6f5358834b127e1d0fb7e9055db64fa5fb7276804291f1554d0a5487782ef506"
			}

#### 2.events:

​		insert: first time inserted along with a new block
​			1. delete all fields with null
​			2. attach timestamp field
​			3. attach firstMinedByBlock field
​			4. hmset key object
​		modify: modified when get receipts from latest_state epoch
​			1. delete all fields with null
​			2. if it is already exists, delete all fields except receipts
​			3. hmset key object
​		delete: deleted when pop an epoch
​			1. if epochNumber matches, delete all fields
​		get:
​			1. attach missing fields with null
​			2. delele firstBlockHash field
​	         comment: When pivot chain pops and changes, the receipts field of a transaction may not consistent with fullnode in a short time. But they will be consistent after a while (append <=4 epochs)

### 三.Block Transactions -> ZADD  key: "blocks:\${blockHash}:transactions"  score: transaction index

#### 1.Block Transactions for block transaction query:

​	key: "blocks:\${blockHash}:transactions"
​	type: Sorted set
​	score: transaction index
​	format: "\${transactionHash}"

#### 2.events:

​		insert: inserted when grow a new epoch
​		delete: deleted when pop an epoch

### 四.Blocks Ordered  ->ZADD key "blocks.in.order"  score\${ZCARD} "\${blockHash}"

#### 1.Blocks Ordered by Execution, for block list query:

​	key: "blocks.in.order"
​	type: Sorted set
​	score: #blocks inserted before me
​	format: "${blockHash}"

#### 2.events:

​	insert: inserted when grow a new epoch
​		 

​		ZADD "blocks.in.order" \${ZCARD} "\${blockHash}"
​		

​	modify: should'n be modified before deleted
​		

​	delete: deleted when pop an epoch
​    

​		ZREM "blocks.in.order" "${blockHash}"

### 五.Transactions Ordered -> ZADD "transactions.in.order" ${ZCARD} "\${transactionHash}"

#### 1.Transactions Ordered by Execution, for transaction list query:

​	key: "transactions.in.order"
​	type: Sorted set
​	score: #transactions inserted before me
​	format: "${transactionHash}"

#### 2.events:

insert: inserted when grow a new epoch
			1. if the transaction is inserted first time:
				ZADD "transactions.in.order" ${ZCARD} "​\${transactionHash}"
				
   2. modify: should'n be modified before deleted

   3. delete: deleted when pop an epoch

        if the transaction is totally deleted:
        ZREM "transactions.in.order" "${transactionHash}"

### 六.Transactions by Account: -> LPUSH key "${transactionHash}"

#### 	1.key: "transactions.by.account:${accountAddress}"

​	type: List
​	format: "${txHash, txType, timestamp}"

#### 	2.events:

​		1.insert: inserted when grow a new epoch

​		if the transaction is inserted first time:
​			LPUSH key "${transactionHash}"
​		2.modify: should'n be modified before deleted
​	    3.delete: deleted when pop an epoch

​		4.if the transaction is totally deleted:
​			LPOP key

### 七.Blocks by Miners: ->ZADD key \${score} "\${blockHash}"

#### 	1.key: "blocks.by.miner:${minerAddress}"

​	type: Sorted set
​	score: block score in "blocks.in.order"
​	format: "\${blockHash}"

#### 	2.events:

​		insert: inserted when grow a new epoch

​				ZADD key \${score} "\${blockHash}"
​       modify: should'n be modified before deleted

​	   delete: deleted when pop an epoch

​				ZREM key "\${blockHash}"

### 八.Latest Block List, capped by X(1000 for now): ->ZADD key \${score} "\${blockHash}"

#### 	1.key: "latest.blocks"

​	type: Sorted set
​	score: block score in "blocks.in.order"
​	format: "${blockHash}"

#### 	2.events:

​		insert: inserted when grow a new epoch

​			  ZADD key \${score} "\${blockHash}"
​		delete: deleted when pop an epoch

​			  ZREM key "${blockHash}".

### 九.Sum difficulties in latest blocks, for difficulty query: -> SET key: "latest.blocks.sum.difficulties"

#### 	1.key: "latest.blocks.sum.difficulties"

​	type: String

#### 	2.events:

​	a) insert a block into latest.blocks

​		1.GET key and parse into bignumber

​		2.added by block.difficulty

​		3.parse to string and SET
​    b) delete a block from latest.blocks

1. GET key and parse into bignumber
2. substracted by block.diffculty
3. parse to string and SET

###  十.transaction in latest blocks, for TPS query:-> INCRBY key: "latest.blocks.sum.transactions"

#### 1.key: "latest.blocks.sum.transactions"

type: Integer

#### 2.events:

​	a) insert a block into latest.blocks
​		1. INCRBY key block.transactionSize
​	b) delete a block from latest.blocks
​		1. INCRBY by -block.transactionSize

### 十一.Miners set: -> HSET key "${minerAccount}" \${blockHash}

#### 	1.key: "miners"

​	type: Hashes
​	key-value format: {
​                    "0x000000000000000000000": 0x1273891289
​                    "0xaaaaaaaaaaaaaaaaaaaaa": 0xa8192038fb
​    			}

#### 	2.events:

​		insert: first time inserted along with a block

​				HSET key "${minerAccount}" \${blockHash}
​         modify: should'n be modified before deleted

​	     delete: deleted when pop an block matched block

​				HDEL key "\${minerAccount}"

### 十二.Pivot chain: -> RPUSH key "\${blockHash}"

####     1.key: "pivot.chain"

​    type: List
​	format: "\${blockHash}"

#### 	2.events:

​		insert: grow a new epoch

​				RPUSH key "\${blockHash}"
​		delete: pop an epoch

​				RPOP key

### 十三. Epoch info: ->RPUSH key: "epoch:${epochNumber}"

####     1.key: "epoch:${epochNumber}"

​    type: List
​	format: "${blockHash}"

#### 	2.events:

​    	new: inserted when grow a new epoch
​    	delete: deleted when pop an epoch

## 贰.mongodb sharding 

![](<https://www.ibm.com/developerworks/cn/opensource/os-cn-apache-cassandra3x3/image003.png>)

mongodb ->MongoDB作为非关系型*数据库*，其主要的优势在于schema-less

如果要设计sharding的话：

好的 shard key 应该拥有如下特性：

- key 分布足够离散 （sufficient cardinality）
- 写请求均匀分布 （evenly distributed write）
- 尽量避免 scatter-gather 查询 （targeted read）

我们主要的目的就是在写操作可扩展，分散写的操作，尽可能的将写操作分散到多个Chunk中。

我们的数据格式为epoch->block->TXs   而且是随着timestamp 来增长的，流数据，

很多sharding算法都是如下：

```
shard = hash(routing) % number_of_primary_shards
```

~~Elasticsearch ，可以提供给外部Elasticsearch 的REST API 给社区，第三方也可以开放他们的web端，PC端，移动端~~

Cassandra 优先考虑

redis , mongodb， cassandra 都采用 一致性hash算法，我们的sharding 也可以采用一致性hash

![](https://user-gold-cdn.xitu.io/2018/4/26/162ffff01dab936a?imageslim)

#### 1.需要拆分的collections，

### 一.Block   ->HMSET ：key :"blocks:${blockHash}"

#### 1.Block Detail, for block hash query:    

​	key: "blocks:\${blockHash}"
​	type: Hashes
   ==sharding key："blocks:\${blockHash}"== 

### 二.Transaction  -> HMSET ：key :"transactions:${txHash}"

#### 1.TrasnactionDetail, for transaction hash query:

 	key: "transactions:${txHash}"
 	type: Hashes
 	 ==sharding key："transactions:\${txHash}"== 



### 三.Block Transactions -> ZADD  key: "blocks:\${blockHash}:transactions"  score: transaction index

#### 1.Block Transactions for block transaction query:

​	key: "blocks:\${blockHash}:transactions"
​	type: Sorted set
​	score: transaction index
​	format: "\${transactionHash}"

​    ==sharding key："transactions:\${txHash}"==