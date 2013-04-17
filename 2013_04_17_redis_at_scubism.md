# redis - 魅惑のスピードとちょっと変わった柔軟性
## naoki @s-cubism 2013/04/17

---

- **リアルタイム解析**にredisが使えないか検討中．
- **4つの方向**からredisをひも解いてみます．

---

- ちなみにnpm/database界隈でredisは一番人気

https://nodejsmodules.org/tags/database

---

整合性？リレーション？

分散性？可用性？永続性？

機能性？アドホック？

スループット？レイテンシ？

---

整合性？リレーション？

分散性？可用性？永続性？

機能性？アドホック？

***スループット？レイテンシ？***

---

# スループット？レイテンシ？

## **redisは速いのか？**

---

http://mocobeta-backup.tumblr.com/post/36435137765/kvs-memcached-couchbase-mongodb-redis

---

## スループット？レイテンシ？

- 単体サーバ & 単純set/get

> 
- **35000tps**(read/write/update)
- **0.05ms**!!!


---

## スループット？レイテンシ？

- ちなみに(単体サーバでの)簡単な比較(だいたい)

- mysql

>
- ~**3000tps**
- ~**2ms**

- mongo


>
- ~**10000tps**
- ~**2ms**

cassandraとかcouchbaseとかは他の人に聞いてみて下さい．

---

## 結論！！

- redisは速い！

---

その気になったら負荷検証スクリプト上げときます．

range queryとaggregationなども比較してます．
riak, couchbaseとか．redshiftとかdynamodbとか．

(YCSBとか重くね？)

---

整合性？リレーション？

分散性？可用性？永続性？

機能性？アドホック？

スループット？レイテンシ？

---

整合性？リレーション？

分散性？可用性？永続性？

***機能性？アドホック？***

スループット？レイテンシ？

---

# 機能性？アドホック？

## redisは使えるのか？

---

- key-valueだけれど、valueにけっこー**いろんな型**がある．
- **ユニークな操作**もかなりある．

---

参考:
the little redis book
>
https://github.com/craftgear/the-little-redis-book/blob/master/ja/redis.md

documents: commands
>
http://redis.io/commands


---

## 5つのデータ構造

str(bitmap)

hash

list

set

sorted set

---

## データ構造の操作

str(bitmap) : set, get, strlen, **getrange**, append, incr, incrby, **setbit**, **getbit**

hash : hset, hget, hmset, hmget, hgetall, hkeys, hdel

list : lpush, ltrim

set: sadd, **sismember**, **sinter**, sinterstore

sorted set: zadd, **zcount**, **zrevrank**

---

## 注意点

- O(logN)とかO(N)とかけっこーあるよ．
- ってかむしろO(1)が基本とかすごいよね！

---

## piplining

単にまとめてリクエスト送る機能．
ネットワーク負荷軽減

(パイプラインじゃなくね？？)

```ruby
redis.pipelined do
  100.times do
    redis.incr('powerlevel')
  end
end
```

---

## その他

- sort -> list, set, sorted set
- expire

---

## 結論

かなりおもろいね！

---

整合性？リレーション？

分散性？可用性？永続性？

機能性？アドホック？

スループット？レイテンシ？

---

***整合性？リレーション？***

分散性？可用性？永続性？

機能性？アドホック？

スループット？レイテンシ？


---

# 整合性？リレーション？

## いろいろ制約あるんだけど．．

---

## 整合性？リレーション？

x リレーション

o トランザクション

multi, exec, discard

```sql
multi
hincrby groups:1percent balance -9000000000
hincrby groups:99percent balance 9000000000
exec
```

o CAS(楽観ロック)

watch 

---

## トランザクション

>
All the commands in a transaction are **serialized** and executed sequentially

### 
>
a Redis transaction is also **atomic**

---

## トランザクション

>
However if the Redis server crashes or is killed by the system administrator in some hard way it is possible that only a partial number of operations are registered. 

###
>
Using the redis-check-aof tool it is possible to fix the append only file that will remove the partial transaction so that the server can start again

###
**atomic性保つよ．crash時には復帰時にどうにかするよ．**

---

## トランザクション

**warning!!: エラー時の挙動**

シンタックスエラーは先に検知するが、**型による操作の間違い**はそのまま通る．

>
even when a command fails, all the other commands in the queue are processed

---

## トランザクション

**warning!!: エラー時の挙動**

```
MULTI
SET a 3
LPOP a
EXEC
```

```
+ OK
- ERR Operation against a key holding the wrong kind of value
```

>
a kind of error that is very likely to be detected during development, and not in production

##
>
simplified and faster

---

## CAS: 楽観ロック

>
If at least one **watched key is modified** before the EXEC command, the **whole transaction aborts**

###

>
watch zset  
element = zrange zset 0 0  
multi  
zrem zset element  
exec  

---

## 結論

まぁまぁだよね．型エラーには気をつけて．

ってか、整合性ってけっきょく永続性とか分断耐性とかと組みだよね！


---

整合性？リレーション？

分散性？可用性？永続性？

機能性？アドホック？

スループット？レイテンシ？

---

整合性？リレーション？

***分散性？可用性？永続性？***

機能性？アドホック？

スループット？レイテンシ？

---

# 分散性？可用性？永続性？
## データ守れる？サービス守れる？

---

## 永続性

1. RDB: snapshoting
2. AOF: append only file

---

## RDB

pros:
>
RDB files are perfect for backups  
very good for disaster recovery, being a single compact file can be transfered to far data centers  
maximizes Redis performances  
faster restarts


##
cons:
>
data loss!  
fork() cost  

---

## AOF

pros:
>
durable  
different fsync policies: no fsync at all, fsync every second(default), fsync at every query  
rewrite the AOF in background when it gets too big
##
cons:
>
bigger  
slower

---

## warning

>
fsync is performed using a background thread and the main thread will try hard to perform writes when no fsync is in progress.

##
>
fsync every time a new command is appended to the AOF. **Very very slow, very safe**


**守りたいところは逐一transaction挟めばいーじゃん．**


---

## 永続性補足: master-slave replication

>
Replications can be used both **for scalability**, in order to have multiple slaves **for read-only queries**

##

failoverはredis sentinel

---

## シャーディング

メモリ、cpu, スループット限界な時．

proxy: redis cluster, Twemproxy

client: http://redis.io/clients

warning!
>
Redis Cluster is currently not production ready

---

- range partitioning
- hash partitionin

(consistent hashing)

---

**redis sentinel**
monitoring, notification, automatic failover

また今度で．．

---

## 結論

基本そこは目をつぶるよね．

---

# その他トピック

## Fast, easy, realtime metrics using Redis bitmaps
http://blog.getspool.com/2011/11/29/fast-easy-realtime-metrics-using-redis-bitmaps/

## Redis as the primary data store? WTF?!
http://moot.it/blog/technology/redis-as-primary-datastore-wtf.html

---


## Fast, easy, realtime metrics using Redis bitmaps

- facebookに買われたspoolでのアクセス解析方法

---


## Fast, easy, realtime metrics using Redis bitmaps

```
redis.setbit(play:yyyy-mm-dd, user_id, 1)
```
![img](unique_users.png)
![img](union.png)

population countはcpuの命令セットに含まれてるから速いよ（全てではない）  
http://en.wikipedia.org/wiki/Hamming_weight

---


## Fast, easy, realtime metrics using Redis bitmaps

- 128 million users
- 50 ms
- 16 mb
 
Daily:	50.2ms  
Weekly:	392.0ms  
Monthly:	1624.8ms  

---


## Fast, easy, realtime metrics using Redis bitmaps

- あまり大量は使えないけど．．
- とりあえず一桁下げれそう．
- npmだと現状popcount速いのない．．
- pythonだとgmpyってのでpopcountすると良さそう．

http://stackoverflow.com/questions/9829578/fast-way-of-counting-bits-in-python

---

## Redis as the primary data store? WTF?!

- Mootという会社
>
We had an initial goal that all API calls would be processed in **less than 10ms** even under heavy load

単純なアプリケーション側シャーディングでデータサイズはカバー  
自前のmaster(write)/slave(read)/persistence(backup to S3)のroleを定義  
failoverはなし(sentinelまだ出てなかった)

---

# 補足
>
16 core machine will only use one core with Redis

---

## 時間なし！

**pub/sub**
けっこー面白いけど．．

**redis sentinel**
monitoring, notification, automatic failover

**redis cluster**

mass insertion

security

signals/crash

scripting















