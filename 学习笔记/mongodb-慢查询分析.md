#### mongo cpu高处理流程
- mongodb cpu居高不下，很大程度是索引问题。

1. 分析数据库正在处理的请求
  - 执行db.currentOp()
  ``` bash
  {
                        "type" : "op",
                        "host" : "DH0179-15-115:26060",
                        "desc" : "conn61435700",
                        "connectionId" : 61435700,
                        "client" : "10.5.17.112:63495",
                        "clientMetadata" : {
                                "driver" : {
                                        "name" : "mongo-java-driver",
                                        "version" : "3.8.2"
                                },
                                "os" : {
                                        "type" : "Linux",
                                        "name" : "Linux",
                                        "architecture" : "amd64",
                                        "version" : "5.5.1-1.el7.elrepo.x86_64"
                                },
                                "platform" : "Java/Oracle Corporation/1.8.0_232-b09"
                        },
                        "active" : true,
                        "currentOpTime" : "2020-06-28T14:59:51.741+0800",
                        "effectiveUsers" : [
                                {
                                        "user" : "tymongo",
                                        "db" : "panda_rcs"
                                }
                        ],
                        "opid" : -1089111958,
                        "lsid" : {
                                "id" : UUID("0dacbd1b-79a9-4249-b453-6cb0b4534f69"),
                                "uid" : BinData(0,"BIg/82y+RJyo/NDiNoTHUawS+P1kzKNLpK3a1dUZbQU=")
                        },
                        "secs_running" : NumberLong(0),
                        "microsecs_running" : NumberLong(6825),
                        "op" : "query",
                        "ns" : "panda_rcs.match_market_live",
                        "command" : {
                                "find" : "match_market_live",
                                "filter" : {
                                        "matchId" : NumberLong(69690)
                                },
                                "limit" : 1,
                                "singleBatch" : true,
                                "$db" : "panda_rcs",
                                "$clusterTime" : {
                                        "clusterTime" : Timestamp(1593327591, 15),
                                        "signature" : {
                                                "hash" : BinData(0,"0lZlCHbLAnyjbFUa1CKjw0bgyCI="),
                                                "keyId" : NumberLong("6790268785883348995")
                                        }
                                },
                                "lsid" : {
                                        "id" : UUID("0dacbd1b-79a9-4249-b453-6cb0b4534f69")
                                }
                        },
                        "planSummary" : "COLLSCAN",
                        "numYields" : 34,
                        "locks" : {
                                "ReplicationStateTransition" : "w",
                                "Global" : "r",
                                "Database" : "r",
                                "Collection" : "r"
                        },
                        "waitingForLock" : false,
                        "lockStats" : {
                                "ReplicationStateTransition" : {
                                        "acquireCount" : {
                                                "w" : NumberLong(35)
                                        }
                                },
                                "Global" : {
                                        "acquireCount" : {
                                                "r" : NumberLong(35)
                                        }
                                },
                                "Database" : {
                                        "acquireCount" : {
                                                "r" : NumberLong(35)
                                        }
                                },
                                "Collection" : {
                                        "acquireCount" : {
                                                "r" : NumberLong(35)
                                        }
                                },
                                "Mutex" : {
                                        "acquireCount" : {
                                                "r" : NumberLong(1)
                                        }
                                }
                        },
                        "waitingForFlowControl" : false,
                        "flowControlStats" : {

                        }
                },
    ```
    - db.currentOp()输出详解：
      - client：请求是由哪个客户端发起的
      - active： 是否处于活动状态
      - ns: 操作的库和表
      - op：发起的处理请求
      - lockstats： 操作获得以下锁后，消耗的微妙时间
      - opid：操作的opid，有需要的话，可以通过 db.killOp(opid) 直接干掉的操作
      - secs_running：操作运行时间。单位是秒
      - microsecs_running： 操作运行时间，默认是微妙
      - query/ns: 这个能看出是对哪个集合正在执行什么操作
      - lock: 锁类型相关信息
      - waitingForLock：为true表示在等待锁 

    - 查看当前处于活动状态的，执行时间大于2秒的操作
      - db.currentOp({active:"true","secs_running":{"$gt":2}})
    - 可以通过该查询，获取长时间操作的opid。执行db.killOp(opid)

#### 分析数据库慢请求
- mongodb支持profiling功能，将请求的执行情况记录到admin下的system.profile集合里。
- profiling三种模式：
  - 0 关闭profiling
  - 1 记录慢查询模式，200为阈值，默认100ms 
    - db.setProfilingLevel(1,200)
  - 2 记录所有操作
``` bash 
drug:PRIMARY> db.system.profile.find().pretty()
{
    "op" : "query",    #操作类型，有insert、query、update、remove、getmore、command   
    "ns" : "mc.user",  #操作的集合
    "query" : {        #查询语句
        "mp_id" : 5,
        "is_fans" : 1,
        "latestTime" : {
            "$ne" : 0
        },
        "latestMsgId" : {
            "$gt" : 0
        },
        "$where" : "new Date(this.latestNormalTime)>new Date(this.replyTime)"
    },
    "cursorid" : NumberLong("1475423943124458998"),
    "ntoreturn" : 0,   #返回的记录数。例如，profile命令将返回一个文档（一个结果文件），因此ntoreturn值将为1。limit(5)命令将返回五个文件，因此ntoreturn值是5。如果ntoreturn值为0，则该命令没有指定一些文件返回，因为会是这样一个简单的find()命令没有指定的限制。
    "ntoskip" : 0,     #skip()方法指定的跳跃数
    "nscanned" : 304,  #扫描数量
    "keyUpdates" : 0,  #索引更新的数量，改变一个索引键带有一个小的性能开销，因为数据库必须删除旧的key，并插入一个新的key到B-树索引
    "numYield" : 0,    #该查询为其他查询让出锁的次数
    "lockStats" : {    #锁信息，R：全局读锁；W：全局写锁；r：特定数据库的读锁；w：特定数据库的写锁
        "timeLockedMicros" : {     #锁
            "r" : NumberLong(19467),
            "w" : NumberLong(0)
        },
        "timeAcquiringMicros" : {  #锁等待
            "r" : NumberLong(7),
            "w" : NumberLong(9)
        }
    },
    "nreturned" : 101,        #返回的数量
    "responseLength" : 74659, #响应字节长度
    "millis" : 19,            #消耗的时间（毫秒）
    "ts" : ISODate("2014-02-25T02:13:54.899Z"), #语句执行的时间
    "client" : "127.0.0.1",   #链接ip或则主机
    "allUsers" : [ ],     
    "user" : ""               #用户
}
```

#### 分析获得的信息 
- 通过以上两个功能，我们能获得慢查询的详细信息。通过这些信息，我们进行判断
  - 全表扫描（关键字： COLLSCAN、 docsExamined ）
    - 全集合（表）扫描COLLSCAN 。 当一个操作请求（如查询、更新、删除等）需要全表扫描时，将非常占用CPU资源。在查看慢请求日志时发现COLLSCAN关键字，很可能是这些查询占用了CPU资源。
  - 通过查看docsExamined的值，可以查看到一个查询扫描了多少文档。该值越大，请求所占用的CPU开销越大。
  - 不合理的索引（关键字： IXSCAN、keysExamined ）
    - 索引不是越多越好，索引过多会影响写入、更新的性能。
    - 如果您的应用偏向于写操作，索引可能会影响性能。
    - 通过查看keysExamined字段，可以查看到一个使用了索引的查询，扫描了多少条索引。该值越大，CPU开销越大
