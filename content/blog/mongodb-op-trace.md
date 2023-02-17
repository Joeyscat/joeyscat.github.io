+++
title = "MongoDB 利用 comment 跟踪操作"
description = ""
date = 2022-08-12
+++


**利用comment跟踪find,update,delete等包含查找行为的操作**

## comment

> comment()[^1] associates a comment string with the find operation. This can make it easier to track a particular query in the following diagnostic outputs:
>
> - The system.profile[^2]
> - The QUERY[^3] log[^4] component
> - db.currentOp()[^5]
>
> https://www.mongodb.com/docs/manual/reference/method/cursor.comment/#behavior

`comment()`可以在查询中附加一段注释，该注释会随着操作被记录到 profile日志，QUERY日志及currentOp中。

comment使用方式(mongo shell)

```js
db.collection.find( { <query> } )._addSpecial( "$comment", <comment> )
db.collection.find( { <query> } ).comment( <comment> )
db.collection.find( { $query: { <query> }, $comment: <comment> } )
```

### systemLog

```js
rds_dQYstkOj [direct: primary] mcm> db.getLogComponents()
{
  verbosity: 0,
  accessControl: { verbosity: -1 },
  command: { verbosity: 0 },
  control: { verbosity: -1 },
  executor: { verbosity: -1 },
  geo: { verbosity: -1 },
  index: { verbosity: -1 },
  network: {
    verbosity: -1,
    asio: { verbosity: -1 },
    bridge: { verbosity: -1 },
    connectionPool: { verbosity: -1 }
  },
  query: { verbosity: -1 },
  replication: {
    verbosity: -1,
    election: { verbosity: -1 },
    heartbeats: { verbosity: -1 },
    initialSync: { verbosity: -1 },
    rollback: { verbosity: -1 }
  },
  sharding: { verbosity: -1, shardingCatalogRefresh: { verbosity: -1 } },
  storage: {
    verbosity: -1,
    recovery: { verbosity: -1 },
    journal: { verbosity: -1 }
  },
  write: { verbosity: -1 },
  ftdc: { verbosity: -1 },
  tracking: { verbosity: -1 },
  transaction: { verbosity: -1 }
}
```

query、write 的 `verbosity` 都是-1，表示继承parent配置，也就是 `verbosity: 0`。

这里需要query log和write log级别设置为2，才会记录到query、update、delete等操作。

## profiler

### profiler配置：

```js
rds_dQYstkOj [direct: primary] mcm> db.getProfilingStatus()
{ was: 1, slowms: 100, sampleRate: 1, ok: 1 }
rds_dQYstkOj [direct: primary] mcm> db.setProfilingLevel(2, { slowms: 100, sampleRate: 1 })
{ was: 1, slowms: 100, sampleRate: 1, ok: 1 }
```

* was 表示profiler级别。
  * 0 表示关闭profiler
  * 1 表示仅记录耗时超过`slowms`的操作
  * 2 表示记录所有操作
* slowms 表示慢操作时间阈值，即耗时超过该值得操作才算慢操作
* sampleRate 表示应记录的慢操作比例，默认为1。

### 测试

#### 修改配置

修改systemLog配置：

```
rds_dQYstkOj [direct: primary] admin> db.setLogLevel(2, "query")
rds_dQYstkOj [direct: primary] admin> db.setLogLevel(2, "write")
```

修改profiler配置：

```js
rds_dQYstkOj [direct: primary] mcm> db.setProfilingLevel(2, { slowms: 100, sampleRate: 1 })
{ was: 1, slowms: 100, sampleRate: 1, ok: 1 }
```

#### 执行操作附加comment

update、remove、query操作filter中附加comment：

```go
func findWithCommentOperator(ctx context.Context, coll *mongo.Collection) (*Order, error) {
	result := &Order{}
	err := coll.FindOne(ctx, bson.M{"$comment": "trace-id:xxxxxxxxxxxxxxxxxxxxxxxxxxxxx"}).Decode(result)
	return result, err
}

func deleteWithCommentOperator(ctx context.Context, coll *mongo.Collection) error {
	_, err := coll.DeleteOne(ctx, bson.M{"$comment": "trace-id:xxxxxxxxxxxxxxxxxxxxxxxxxxxxx"})
	return err
}

func updateWithCommentOperator(ctx context.Context, coll *mongo.Collection) error {
	_, err := coll.UpdateOne(ctx, bson.M{"$comment": "trace-id:xxxxxxxxxxxxxxxxxxxxxxxxxxxxx"},
		bson.M{"$set": bson.M{"name": "jojo"}})
	return err
}
```

#### 查看日志

系统日志

```shell
2022-08-12T16:30:06.828+0800 D2 QUERY    [conn209394] Only one plan is available; it will be run but will not be cached. ns: mcm.order query: { $comment: "trace-id:xxxxxxxxxxxxxxxxxxxxxxxxxxxxx" } sort: {} projection: {} limit: 1, planSummary: COLLSCAN
2022-08-12T16:30:06.829+0800 D2 QUERY    [conn209394] Using idhack: ns: config.transactions query: { _id: { id: UUID("9094dbc7-791e-46c3-8ce1-47c7787533ce"), uid: BinData(0, 20FC9AD9551DFCA3D3A36734D1019EC52177EEA0214881F18E2FDE052D078E5F) } } sort: {} projection: {} ntoreturn=1
2022-08-12T16:30:06.829+0800 D2 QUERY    [conn209394] Only one plan is available; it will be run but will not be cached. ns: mcm.order query: { $comment: "trace-id:xxxxxxxxxxxxxxxxxxxxxxxxxxxxx" } sort: {} projection: {}, planSummary: COLLSCAN
2022-08-12T16:30:06.829+0800 I  WRITE    [conn209394] remove mcm.order command: { q: { $comment: "trace-id:xxxxxxxxxxxxxxxxxxxxxxxxxxxxx" }, limit: 1 } planSummary: COLLSCAN keysExamined:0 docsExamined:1 ndeleted:1 keysDeleted:1 numYields:0 locks:{ ParallelBatchWriterMode: { acquireCount: { r: 2 } }, ReplicationStateTransition: { acquireCount: { w: 2 } }, Global: { acquireCount: { w: 2 } }, Database: { acquireCount: { w: 2 } }, Collection: { acquireCount: { w: 2 } }, Mutex: { acquireCount: { r: 4 } } } flowControl:{ acquireCount: 1 } storage:{} 0ms
2022-08-12T16:30:06.829+0800 D2 QUERY    [conn209394] Only one plan is available; it will be run but will not be cached. ns: mcm.order query: { $comment: "trace-id:xxxxxxxxxxxxxxxxxxxxxxxxxxxxx" } sort: {} projection: {}, planSummary: COLLSCAN
2022-08-12T16:30:06.830+0800 I  WRITE    [conn209394] update mcm.order command: { q: { $comment: "trace-id:xxxxxxxxxxxxxxxxxxxxxxxxxxxxx" }, u: { $set: { name: "jojo" } }, multi: false, upsert: false } planSummary: COLLSCAN keysExamined:0 docsExamined:1 nMatched:1 nModified:1 numYields:0 locks:{ ParallelBatchWriterMode: { acquireCount: { r: 2 } }, ReplicationStateTransition: { acquireCount: { w: 2 } }, Global: { acquireCount: { w: 2 } }, Database: { acquireCount: { w: 2 } }, Collection: { acquireCount: { w: 2 } }, Mutex: { acquireCount: { r: 3 } } } flowControl:{ acquireCount: 1 } storage:{} 0ms
```

profile日志：

```js
rds_dQYstkOj [direct: primary] mcm> db.system.profile.find({},{op:1,ns:1,command:1,ts:1}).sort({ts:-1}).limit(4)
[
  {
    op: 'update',
    ns: 'mcm.order',
    command: {
      q: { '$comment': 'trace-id:xxxxxxxxxxxxxxxxxxxxxxxxxxxxx' },
      u: { '$set': { name: 'jojo' } },
      multi: false,
      upsert: false
    },
    ts: ISODate("2022-08-12T08:30:06.830Z")
  },
  {
    op: 'remove',
    ns: 'mcm.order',
    command: {
      q: { '$comment': 'trace-id:xxxxxxxxxxxxxxxxxxxxxxxxxxxxx' },
      limit: 1
    },
    ts: ISODate("2022-08-12T08:30:06.829Z")
  },
  {
    op: 'query',
    ns: 'mcm.order',
    command: {
      find: 'order',
      filter: { '$comment': 'trace-id:xxxxxxxxxxxxxxxxxxxxxxxxxxxxx' },
      limit: Long("1"),
      singleBatch: true,
      lsid: { id: UUID("9094dbc7-791e-46c3-8ce1-47c7787533ce") },
      '$db': 'mcm',
      '$readPreference': { mode: 'primary' }
    },
    ts: ISODate("2022-08-12T08:30:06.828Z")
  },
...
]
```

update、remove、query操作都有记录附加的comment信息


[^1]: https://www.mongodb.com/docs/manual/reference/method/cursor.comment/

[^2]: https://www.mongodb.com/docs/manual/reference/system-collections/#mongodb-data--database-.system.profile

[^3]: https://www.mongodb.com/docs/manual/reference/log-messages/#mongodb-data-QUERY

[^4]: https://www.mongodb.com/docs/manual/reference/log-messages/

[^5]: https://www.mongodb.com/docs/manual/reference/method/db.currentOp/#mongodb-method-db.currentOp

