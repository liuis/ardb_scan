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

### 创建keyspace 

```sql
CREATE KEYSPACE IF NOT EXISTS conflux_scan
WITH replication = {'class':'SimpleStrategy', 'replication_factor' : 3};
```

### 创建 table

##### cassandra 会把大写的字段名变成小写，使用下划线分割

#### Cassandra Collection: Set, List, Map

Cassandra collections are a good way for handling tasks. Multiple elements can be stored in collections. There are limitations in Cassandra collections.

- Cassandra collection cannot store data more than 64KB.
- Keep a collection small to prevent the overhead of querying collection because entire collection needs to be traversed.
- If you store more than 64 KB data in the collection, only 64 KB will be able to query, it will result in loss of data.

在2.2之前的版本collections 会有size大小的限制，为64K，2.2版本之后可以插入2billion的items

if you are using cassandra 2.2 and later you can insert 2billion items into collection. here is the link. [http://docs.datastax.com/en/cql/3.3/cql/cql_using/useCollections.html](http://docs.datastax.com/en/cql/3.3/cql/cql_using/useCollections.html)



#### 1.Block Detail, for block hash query:

table_name: block_detail

```sql

CREATE TABLE IF NOT EXISTS
block_detail 
( block_hash text  PRIMARY KEY,
 detail map<text, text>,
 refereeHashes list<text>
);

insert into block_detail JSON '
{
  "block_hash": "12345",
  "detail": {
    "deferredReceiptsRoot": "0x42c7a763c5fb8d2342b1a75b2d214f83212eca6bb9ad432502b40e73b1b2b22d",
    "deferredStateRoot": "0x7acf7512bb021303ac4517ea0140c53ab6fd35b923ff55292a58a11dbb61f79e",
    "difficulty": "0x77397115",
    "epochNumber": "0x15817",
    "gasLimit": "0xb2d05e00",
    "hash": "0x6f5358834b127e1d0fb7e9055db64fa5fb7276804291f1554d0a5487782ef506",
    "height": "0x15817",
    "miner": "0x0000000000000000000000000000000000000008",
    "nonce": "0x6ccbf7cbe080f3a4",
    "parentHash": "0x889a47fd6433af97a9d29a29878cd6e0698ddfef16ea6e1ac736eff5e10e12ba",
    "size": "0x0",
    "timestamp": "0x5cb81529",
    "transactionsRoot": "0x1dcc4de8dec75d7aab85b567b6ccd41ad312451b948a7413f0a142fd40d49347",
    "isPivot": "0x1",
    "newTransactionCount": "1111"
  },
  "refereeHashes": [
    "0x661cd2771f13f3790ffc70eb118c7ad2ecc8de48f8f7fc07ed9b923c160f0da7"
  ]
}';



detail => map: 

{    'deferredReceiptsRoot': '0x42c7a763c5fb8d2342b1a75b2d214f83212eca6bb9ad432502b40e73b1b2b22d',
  'deferredStateRoot': '0x7acf7512bb021303ac4517ea0140c53ab6fd35b923ff55292a58a11dbb61f79e',
  "difficulty": "0x77397115",
  "epochNumber": "0x15817",
  "gasLimit": "0xb2d05e00",
  "hash": "0x6f5358834b127e1d0fb7e9055db64fa5fb7276804291f1554d0a5487782ef506",
  "height": "0x15817",
  "miner": "0x0000000000000000000000000000000000000008",
  "nonce": "0x6ccbf7cbe080f3a4",
  "parentHash": "0x889a47fd6433af97a9d29a29878cd6e0698ddfef16ea6e1ac736eff5e10e12ba",
  "refereeHashes": 
    "0x661cd2771f13f3790ffc70eb118c7ad2ecc8de48f8f7fc07ed9b923c160f0da7"
  ,
  "size": "0x0",
  "timestamp": "0x5cb81529",
  "transactionsRoot": "0x1dcc4de8dec75d7aab85b567b6ccd41ad312451b948a7413f0a142fd40d49347",
  "isPivot": "0x1",
  "newTransactionCount": "1111"
}
```

#### 2.Transaction Detail, for transaction hash query:

table_name : tx_detail

```sql
CREATE TABLE IF NOT EXISTS
tx_detail 
( tx_hash text, 
 detail map<text, text>,
 PRIMARY KEY (tx_hash)
);
insert into tx_detail JSON '
{
  "tx_hash": "123456",
  "detail": {
    "blockHash": "1234",
    "contractCreated": "12345",
    "data": "0x11",
    "from": "0x5510949f549c6f251bf9db1aa51674abc5b1f6db",
    "gas": "0x989680",
    "gasPrice": "0x2",
    "hash": "0x19a447a350fd7f35455bf6e29ee922973eb49bea6dd76dbbd1a19fbf86ce2c0f",
    "nonce": "0x7",
    "r": "0xc20e60663863f397c92e009ed9232255c6a2001ddfdcdcb2405cd9c203a41703",
    "s": "0x9d14f7437897d59104a4a06573ecaae39ad943b327631d3d1e72a80b26b1281",
    "v": "0x0",
    "to": "null",
    "transactionIndex": "0x0",
    "timestamp": "0x5cb81529",
    "value": "0x0",
    "firstBlockHash": "0x6f5358834b127e1d0fb7e9055db64fa5fb7276804291f1554d0a5487782ef506"
  }
}';

detail => map:
{
  "blockHash": "1234",
  "contractCreated": "12345",
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


 ////# 表注意，from 和 to 有关键字问题，改为alias_from 和 alias_to
 #此处要有分区键(sharding key) hash 作为分区键
```

#### 3.Block Transactions, for block transaction query:

用于查询某个block的所有的tx

table_name：blockhash_txs

```sql
CREATE TABLE IF NOT EXISTS
blockhash_txs 
( id UUID,
 block_hash text ,
 transaction_hash  set<text>,
 PRIMARY KEY (id, block_hash) 
);
#此处要做分区键，transactionHash 作为分区键  set<text> 无法作为分区键
```

#### 4.Blocks Ordered by Execution, for block list query:

所有的排好序的block

table_name : blocks_in_oreder

```sql
/*create type if not exists
blocks_score
(
   block_hash text,
   score  text
);
*/
CREATE TABLE IF NOT EXISTS
blocks_in_order 
( id UUID PRIMARY KEY,
  block_hash set<frozen  <map<text, int>>>,
);

map => 
{
   "block_hash":xxxx,
   "score": int
}
```

#### 5.Transactions Ordered by Execution, for Transactions list query:

所有的排好序的block

table_name : transactions_in_oreder

```sql
CREATE TABLE IF NOT EXISTS
transactions_in_oreder 
( id UUID PRIMARY KEY, 
  transaction_hash set<text>
);
#此处做分区键(sharding key)
```

#### 6.Transactions by Account:

所有的排好序的block

table_name : transactions_by_account

```sql

CREATE TABLE IF NOT EXISTS
transactions_by_account 
( account_address text PRIMARY KEY ,
  tx list<frozen <map<text, text>>>,
);

cqlsh:conflux_scan> insert into transactions_by_account JSON '{
                ...   "account_address": "2xxx",
                ...   "tx": [
                ...     {
                ...       "txHash": "1xxxxx",
                ...       "txType": "sender",
                ...       "timestamp": "13457899"
                ...     },
                ...     {
                ...       "txHash": "13xxxxx",
                ...       "txType": "sender",
                ...       "timestamp": "23457899"
                ...     }
                ...   ]
                ... }';
cqlsh:conflux_scan> select * from transactions_by_account ;

 account_address | tx
-----------------+-----------------------------------------------------------------------------------------------------------------------------------------
           14566 |                                                                                                                                    null
            2xxx | [{'timestamp': '13457899', 'txHash': '1xxxxx', 'txType': 'sender'}, {'timestamp': '23457899', 'txHash': '13xxxxx', 'txType': 'sender'}]
tx  map =>
 { "txHash" :xxxxx,
  "txType":  xxxxx,
  "timestamp": xxxxx
  }
, #此处做分区键(sharding key)account_address
```

#### 7.blocks by Miners

table_name : blocks_by_miner

```sql
CREATE TABLE IF NOT EXISTS
blocks_by_miner 
( 
  miner_address text PRIMARY KEY,
  block_hash list<frozen  <map<text, int>>>
);

cqlsh:conflux_scan> insert into blocks_by_miner JSON '{
                ... "miner_address": "1234",
                ...
                ... "block_hash" :
                ... [
                ...   {"block_hash" : "03331", "score" : 1},
                ...   {"block_hash" : "03330", "score" : 2}
                ...
                ... ]
                ...
                ... }';
cqlsh:conflux_scan> select * from blocks_by_miner ;

 miner_address | block_hash
---------------+----------------------------------------------------------------------
          1234 | [{'block_hash': 3331, 'score': 1}, {'block_hash': 3330, 'score': 2}]
map => 
{
   "block_hash":xxxx,
   "score": int
}

```

#### 8.Latest Block List, capped by X(1000 for now):

1000个最新的区块

table_name : latest_block_list

```sql
CREATE TABLE IF NOT EXISTS
latest_block_list 
( 
  id varchar PRIMARY KEY,
  block_hash list<text>,
);

cqlsh:conflux_scan> insert into latest_block_list(id , block_hash) values ('cfx', ['1xx233', '32432xxx33']);
cqlsh:conflux_scan> select * from latest_block_list ;

 id  | block_hash
-----+--------------------------
 cfx | ['1xx233', '32432xxx33']
```

#### 9.Sum difficulties in latest blocks, for difficulty query:

在插入最新的区块的时候，获取区块中的难度值

table_name : latest_blocks_sum_difficulties

```sql
CREATE TABLE IF NOT EXISTS
latest_blocks_sum_difficulties 
( id varchar PRIMARY KEY,
  difficulty varchar
);

select * from latest_blocks_sum_difficulties
                ... ;

 id | difficulty
----+------------

(0 rows)
cqlsh:conflux_scan> insert into latest_blocks_sum_difficulties(id, difficulty) values('cfx', '0');
cqlsh:conflux_scan> select * from latest_blocks_sum_difficulties  ;

 id  | difficulty
-----+------------
 cfx |          0
```

#### 10.transaction in latest blocks, for TPS query::

在插入最新的区块的时候，获取区块中的交易数相加来获取TPS

table_name : latest_blocks_sum_transactions

```sql
CREATE TABLE IF NOT EXISTS
latest_blocks_sum_transactions 
( id varchar PRIMARY KEY,
  tps int
);

cqlsh:conflux_scan> insert into  latest_blocks_sum_transactions(id,tps) values('cfx', 1000);
cqlsh:conflux_scan> select * from latest_blocks_sum_transactions ;

 id  | tps
-----+------
 cfx | 1000
```

#### 11.Miners 表

table_name: miners

```sql
CREATE TABLE IF NOT EXISTS
miners 
( 
  miner_address text  PRIMARY KEY,
  block_hash  list<text>
);


cqlsh:conflux_scan> CREATE TABLE IF NOT EXISTS
                ... miners
                ... (
                ...   miner_address text  PRIMARY KEY,
                ...   block_hash  list<text>
                ... );
cqlsh:conflux_scan> insert into miners(miner_address, block_hash) values ('0x3333', ['0x123', 0x456677]);
InvalidRequest: Error from server: code=2200 [Invalid query] message="Invalid list literal for block_hash: value 0x456677 is not of type text"
cqlsh:conflux_scan> insert into miners(miner_address, block_hash) values ('0x3333', ['0x123', '0x456677']);
cqlsh:conflux_scan> select * from miners ;

 miner_address | block_hash
---------------+-----------------------
        0x3333 | ['0x123', '0x456677']
```

#### 11.PivotChain 表

table_name: pivot_chain

```sql
CREATE TABLE IF NOT EXISTS
pivot_chain 
( id varchar PRIMARY KEY,
  block_hash  list<text>
);

cqlsh:conflux_scan> CREATE TABLE IF NOT EXISTS
                ... pivot_chain
                ... ( id varchar PRIMARY KEY,
                ...   block_hash  list<text>
                ... );
cqlsh:conflux_scan> insert into pivot_chain(id, block_hash) values ('cfx', ['0x23dfsr32', '0x43tgdt4']);
cqlsh:conflux_scan> select * from pivot_chain ;

 id  | block_hash
-----+-----------------------------
 cfx | ['0x23dfsr32', '0x43tgdt4']

(1 rows)
cqlsh:conflux_scan>
```

#### 12.EpochInfo 表

table_name: epoch_info

```sql
CREATE TABLE IF NOT EXISTS
epoch_info 
( epochNumber int PRIMARY KEY,
  block_hash  list<text>
);
```

#### 13.Sync 表

table_name: sync 

常量表

```sql
CREATE TABLE IF NOT EXISTS
sync_mode
( id varchar PRIMARY KEY,
  sync int 
);

insert into sync_mode(id, sync) values ("cfx", 1);
```

#### 

## 引用

1.官方文档[http://cassandra.apache.org/doc/latest/architecture/index.html](http://cassandra.apache.org/doc/latest/architecture/index.html)

2.Cassandra 安装引导 https://dzone.com/articles/install-cassandra-on-ubuntu-1804

​    https://linuxize.com/post/how-to-install-apache-cassandra-on-ubuntu-18-04/

3. macos install guitar：https://gist.github.com/JasperWoo/f90408544133d01ccba8d77ddfdaf75b
4. cassandra 如何写入一个json  https://stackoverflow.com/questions/25098451/cql3-each-row-to-have-its-own-schema

5.cassandra的collection limit https://stackoverflow.com/questions/40438429/what-are-the-correct-cassandra-collection-limits