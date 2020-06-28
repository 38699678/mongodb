#### mtool 分析mongo日志工具

-  mlogfilter 解析mongo.log日志
  - 分析慢查询并以json格式，可以导入到数据库中
    - /usr/local/python/bin/mlogfilter mongod.log --slow --json |more
    - /usr/local/python/bin/mlogfilter mongod.log --slow --json | mongoimport -d test -c mycoll

  - 查询某个库某个集合的慢查询， --slow指定慢查询的时间为多少毫秒
    - /usr/local/python/bin/mlogfilter mongod.log --namespace admin.\$cmd --slow 1000
    - /usr/local/python/bin/mlogfilter mongod.log --namespace bg_panda.rh_match_event --slow 1000
``` bash
 2020-06-28T00:59:12.873+0800 I WRITE [conn59042814] remove bg_panda.rh_match_event appName: "MongoDB Shell" command: { q: { createTime: { $gte: "2020-06-21 00:00:00", $lt: "2020-06-22 00:00:00" } }, limit: 0 } planSummary: IXSCAN { createTime: -1 } keysExamined:29761 docsExamined:29761 ndeleted:29761 keysDeleted:267849 numYields:233 queryHash:6472E07B planCacheKey:2B96BB62 locks:{ ParallelBatchWriterMode: { acquireCount: { r: 234 } }, ReplicationStateTransition: { acquireCount: { w: 234 } }, Global: { acquireCount: { w: 234 } }, Database: { acquireCount: { w: 234 } }, Collection: { acquireCount: { w: 234 } }, Mutex: { acquireCount: { r: 59523 } } } flowControl:{ acquireCount: 234 } storage:{ data: { bytesRead: 3795567, timeReadingMicros: 3603 } } 1091ms
```
  - 查看某一个操作类型的慢查询，一次只能指定一个操作类型，可以是query,insert,update,delete,command,getmore
    - /usr/local/python/bin/mlogfilter mongod.log --slow  1000  --namespace bg_panda.rh_match_event   --operation query
  - 根据线程id查看慢查询
    - /usr/local/python/bin/mlogfilter mongod.log --slow  1000  --namespace order.bill   --operation query  --thread conn1317475
  - 查看9月份的日志
    - /usr/local/python/bin/mlogfilter mongod.log --from Sep
  - 查看5分钟前的日志
    - /usr/local/python/bin/mlogfilter mongod.log --from "now -5min"
  - 返回当天00点到2点的日志 
    - /usr/local/python/bin/mlogfilter mongod.log --namespace bg_panda.rh_match_event --slow 1000 --from today --to +2hours
  - 返回当天从9：30开始的日志
    - /usr/local/python/bin/mlogfilter mongod.log --from today 9:30

- mloginfo 统计mongod日志信息
  - 显示统计信息
    - /usr/local/python/bin/mloginfo mongod.log --queries
``` bash 
namespace                                         operation    pattern                                            count    min (ms)    max (ms)    mean (ms)    95%-ile (ms)    sum (ms)    allowDiskUse

panda_rcs.rcs_market_category                     find         {"matchId": 1}                                     56156         146         396    n/a               9786172       174.3    None
sr_panda.sr_panda_live_marketOdds                 remove       {"modifyTime": 1}                                      1      139241      139241    n/a                139241    139241.0    None
bc.original_match_info                            find         {"date": 1, "isVisible": 1, "sportId": 1}            556         141         969    n/a                122364       220.1    None
sr_panda.sr_panda_live_eventStatisticsInfo_uof    remove       {"createTime": 1}                                      1       27578       27578    n/a                 27578     27578.0    None
panda_rcs.rcs_market_category                     remove       {"$and": [{"matchStartTime": 1}], "sportId": 1}       98         214         382    n/a                 25017       255.3    None
bg_panda.rh_match_marketodds                      remove       {"createTime": 1}                                      1       17803       17803    n/a                 17803     17803.0    None
panda_rcs.rcs_market_category                     remove       {"crtTime": 1, "matchStartTime": 1}                   74         167         256    n/a                 14557       196.7    None
sr_panda.sr_panda_live_eventInfo_bmk              remove       {"createTime": 1}                                      1       13486       13486    n/a                 13486     13486.0    None
bc_panda.rh_match_market                          remove       {"createTime": 1}                                      1       11844       11844    n/a                 11844     11844.0    None
bc.original_match_market                          remove       {"generationTime": 1}                                  1       10163       10163    n/a                 10163     10163.0    None
sr_panda.sr_panda_fix_matchInfo                   remove       {"modifyTime": 1}                                      1        3700        3700    n/a                  3700      3700.0    None
sr_panda.sr_panda_live_matchStatus_uof            remove       {"modifyTime": 1}                                      1        2824        2824    n/a                  2824      2824.0    None
bc_panda.rh_match_info                            remove       {"createTime": 1}                                      1        2687        2687    n/a                  2687      2687.0    None
sr_panda.sr_panda_live_betSettlement              remove       {"createTime": 1}                                      1        2598        2598    n/a                  2598      2598.0    None
bg_panda.rh_match_status                          remove       {"createTime": 1}                                      1        1952        1952    n/a                  1952      1952.0    None
bc_panda.rh_competition_info                      remove       {"createTime": 1}                                      1        1407        1407    n/a                  1407      1407.0    None
bg_panda.rh_match_event                           remove       {"createTime": 1}                                      1        1091        1091    n/a                  1091      1091.0    None
sr_panda.sr_panda_fix_tournament                  remove       {"modifyTime": 1}                                      1         586         586    n/a                   586       586.0    None
bg_panda.rh_match_result                          remove       {"createTime": 1}                                      1         248         248    n/a                   248       248.0    None
bc_panda.rh_match_result                          remove       {"createTime": 1}                                      1         237         237    n/a                   237       237.0    None
sr_panda.sr_panda_live_betStatus                  remove       {"modifyTime": 1}                                      1         167         167    n/a                   167       167.0    None
```
  - 对统计结果进行排序
    - mloginfo mongod.log --queries --sort count
    - mloginfo mongod.log --queries --sort sum
  - 显示重启信息
    - mloginfo mongod.log --restarts
  - 显示统计日志信息
    - mloginfo mongod.log --distinct
  - 统计连接信息
    - mloginfo mongod.log --connections
  - 统计复制集信息
    - mloginfo mongod.log --rsstate
