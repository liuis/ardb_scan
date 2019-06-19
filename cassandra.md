## Cassandra 的数据存储结构

Cassandra 的数据模型是基于列族（Column Family）的四维或五维模型。它借鉴了 Amazon 的 Dynamo 和 Google's BigTable 的数据结构和功能特点，采用 Memtable 和 SSTable 的方式进行存储。在 Cassandra 写入数据之前，需要先记录日志 ( CommitLog )，然后数据开始写入到 Column Family 对应的 Memtable 中，Memtable 是一种按照 key 排序数据的内存结构，在满足一定条件时，再把 Memtable 的数据批量的刷新到磁盘上，存储为 SSTable 。

  下图为Cassandra的数据模型图

![img](https://www.ibm.com/developerworks/cn/opensource/os-cn-cassandra/image001.gif)

## Cassandra 的数据模型的基本概念：

1. Cluster : Cassandra 的节点实例，它可以包含多个 Keyspace

2. Keyspace : 用于存放 ColumnFamily 的容器，相当于关系数据库中的 Schema 或 database

3. ColumnFamily : 用于存放 Column 的容器，类似关系数据库中的 table 的概念 

4. SuperColumn ：它是一个特列殊的 Column, 它的 Value 值可以包函多个 Column

5. Columns：Cassandra 的最基本单位。由 name , value , timestamp 组成


## 数据模型实例

![数据模型实例](https://www.ibm.com/developerworks/cn/opensource/os-cn-cassandra/image002.jpg)







## conflux_scan_code_env



2. ```
   go的Http server
   go get -u github.com/gin-gonic/gin    
   go的web3 接口
   go get -u github.com/regcostajr/go-web3
   go的cassandra的driver
   go get github.com/gocql/gocql
   ```



## 引用

1.官方文档[http://cassandra.apache.org/doc/latest/architecture/index.html](http://cassandra.apache.org/doc/latest/architecture/index.html)

2.Cassandra 安装引导 https://dzone.com/articles/install-cassandra-on-ubuntu-1804

​    https://linuxize.com/post/how-to-install-apache-cassandra-on-ubuntu-18-04/

3. macos install guitar：https://gist.github.com/JasperWoo/f90408544133d01ccba8d77ddfdaf75b
4. cassandra 如何写入一个json  https://stackoverflow.com/questions/25098451/cql3-each-row-to-have-its-own-schema





## ISSUES

1. 先通过go web3 可以连通 conflux fullnode
2. cassandra 不是schema less 的 需要针对不同的需求建立表,然后设定设定好分区键，可以sharding的做sharding
3. 操作从conflux来的数据，新增一个epoch 然后操作 增删改查 cassandra
4. Http server的客户端的router的请求和返回API接口。

## New ISSUES

不在使用golang 来作为编程语言，继续使用nodejs

1.现在的问题就是保留所有的db.js 和sync.js 的接口，对上层逻辑是无感知的。只需要改动db.js 和sync.js的逻辑



## cassandra sharding

1.block detail

2.total ordered block

3.pivot chain

4.block epoch

5.block by miner

==6.latest block==

------

上述1-6的功能因为是0.25s/block  所以可以不用作sharding

下述7-10的功能因为每秒3000TPS的写入量要做sharding

------

7.tx detail  —> sharding key   Tx hash

8.tx  by  account —> sharding key  account_address

9.total ordered Tx ———>   

- 直接放到内存里  
- 累加一定数量的tx  ，然后压缩一次写入数据库

10. tx by block  ——>

- "blockhash:1"  作为sharding key， 1.2.3 ~ len(tx) 
- 累加一定数量的tx  ，然后压缩一次写入数据库

## cassandra schema 

#### 1.Block Detail, for block hash query:

table_name: block_detail

```sql
CREATE TABLE 
.block_detail 
( id UUID  PRIMARY KEY,
  hash text, 
 lastnamedeferredReceiptsRoot text, 
 deferredStateRoot text, 
 difficulty text, 
 epochNumber text, 
 gasLimit text,
 height  text,
 miner  text,
 nonce  text,
 parentHash  text,
 refereeHashes text,
 size text,
 timestamp text,
 transactionsRoot text,
 isPivot text
);
 		    
```

#### 2.Transaction Detail, for transaction hash query:

table_name : tx_detail

```sql
CREATE TABLE 
.tx_detail 
( id UUID,
  hash text, 
 blockHash text, 
 contractCreated text, 
 data text, 
 from text, 
 gas text,
 gasPrice  text,
 hash  text,
 nonce  text,
 r  text,
 s text,
 v text,
 to text,
 transactionIndex text,
 timestamp text,
 value  text,
 firstBlockHash text,
 PRIMARY KEY (id, hash) #此处要有分区键(sharding key) hash 作为分区键
);
```

#### 3.Block Transactions, for block transaction query:

用于查询某个block的所有的tx

table_name：blockhash_txs

```sql
CREATE TABLE 
.blockhash_txs 
( id UUID，
  transactionHash text ,
  blockhash set<text>，
 PRIMARY KEY (id, transactionHash) #此处要做分区键，transactionHash 作为分区键
);
```

#### 4.Blocks Ordered by Execution, for block list query:

所有的排好序的block

table_name : blocks_in_oreder

```sql
CREATE TABLE 
.blocks_in_order 
( id UUID PRIMARY KEY,
  blockHash set<text>
);
```

#### 5.Transactions Ordered by Execution, for Transactions list query:

所有的排好序的block

table_name : transactions_in_oreder

```sql
CREATE TABLE 
.blocks_in_order 
( id UUID PRIMARY KEY, #此处做分区键(sharding key)
  transactionHash set<text>
);
```

#### 6.Transactions by Account:

所有的排好序的block

table_name : transactions_by_account

```sql
CREATE TABLE 
.transactions_by_account 
( id UUID ,
  account_address text,
  transactionHash list<text>,

  PRIMARY KEY (id, account_address), #此处做分区键(sharding key)account_address
);
```

#### 7.blocks by Miners:

所有的排好序的block

table_name : blocks_by_miner

```sql
CREATE TABLE 
.blocks_by_miner 
( id UUID PRIMARY KEY,
  miner_address text,
  blockHash set<text>
);
```

#### 8.Latest Block List, capped by X(1000 for now):

1000个最新的区块

table_name : latest_block_list

```sql
CREATE TABLE 
.latest_block_list 
( id UUID PRIMARY KEY,
  blockHash set<text>
);
```

#### 9.Sum difficulties in latest blocks, for difficulty query:

在插入最新的区块的时候，获取区块中的难度值

table_name : latest_blocks_sum_difficulties

```sql
CREATE TABLE 
.latest_blocks_sum_difficulties 
( id UUID PRIMARY KEY,
  difficulty string
);
```

#### 10.transaction in latest blocks, for TPS query::

在插入最新的区块的时候，获取区块中的交易数相加来获取TPS

table_name : latest_blocks_sum_transactions

```sql
CREATE TABLE 
.latest_blocks_sum_transactions 
( id UUID PRIMARY KEY,
  tps integer
);
```

#### 11.Miners 表

table_name: miners

```sql
CREATE TABLE 
.miners 
( id UUID PRIMARY KEY,
  miner_address text,
  blockHash  text
);
```

#### 11.PivotChain 表

table_name: pivot_chain

```sql
CREATE TABLE 
.pivot_chain 
( id UUID PRIMARY KEY,
  blockHash  list<text>
);
```

#### 12.EpochInfo 表

table_name: epoch_info

```sql
CREATE TABLE 
.epoch_info 
( id UUID PRIMARY KEY,
  epochNumber, integer,
  blockHash  list<text>
);
```

#### 