#### mongodump
- 介绍
  - mongodump能够在Mongodb运行时进行备份，它的工作原理是对运行的Mongodb做查询，然后将所有查到的文档写入磁盘。但是存在的问题时使用mongodump产生的备份不一定是数据库的实时快照，如果我们在备份时对数据库进行了写入操作，则备份出来的文件可能不完全和Mongodb实时数据相等。另外在备份时可能会对其它客户端性能产生不利的影响
- 用法：
  - -h:指明数据库宿主机的IP
  - -u:指明数据库的用户名
  - -p:指明数据库的密码
  - -d:指明数据库的名字
  - -c:指明collection的名字
  - -o:指明到要导出的文件名
  - -q:指明导出数据的过滤条件
  - -j, --numParallelCollections=  number of collections to dump in parallel (4 by default)
  - --oplog  备份的同时备份oplog
  - -authenticationDatabase：指定认证库
- 全库备份
  - mongodump -uadmin -p123 --port 26060 --authenticationDatabase admin -o mongodb/full
- 备份指定库
  -  mongodump -uadmin -p123 --port 26060 --authenticationDatabase admin -d test -o  mongodb/db/
- 备份指定库下的指定表 
  - mongodump -uadmin -p123 --port 26060 --authenticationDatabase admin -d test -c log -o mongodb/collection/ 
- 压缩备份：
  - mongodump -uadmin -p123 --port 26060 --authenticationDatabase admin -d test -o mongodb/db/ --gzip

#### mongorestore
- 恢复
- 恢复库
  - mongorestore -uadmin -p123 --port 26060 --authenticationDatabase  admin -d test  mongodb/full/test/   
- 恢复表：
  - mongorestore -uadmin -p123 --port 26060 --authenticationDatabase admin -d test -c log mongodb/collection/test/log.bson
- 恢复前删除原始数据(***慎用***)
  - mongorestore -uadmin -p123 --port 26060 --authenticationDatabase  admin -d test --drop mongodb/full/test/ 

#### oplog
- oplog简单的说类似于mysql的binlog。其中记录的是整个mongod实例一段时间内数据库的所有变更（插入/更新/删除）操作
  - 默认分配的大小为磁盘空闲的5%，可以通过 mongod --oplogSize设置大小
  - 在local.oplog.rs表中记录
  - 只有在副本集架构中存在
  - 当空间用完时新记录自动覆盖最老的记录。
- op：
  - i： insert
  - u:  update
  - d:  delete
  - c:  db cmd
- rs.printReplicationInfo()
  - ty88:PRIMARY> rs.printReplicationInfo();  
    configured oplog size:   51200MB  #oplog集合大小
    log length start to end: 3542secs (0.98hrs)   语句窗口覆盖时间，多长时间后后面的操作会覆盖最早的操作
    oplog first event time:  Tue Jun 23 2020 16:21:40 GMT+0800 (CST)  
    oplog last event time:   Tue Jun 23 2020 17:20:42 GMT+0800 (CST)  
    now:                     Tue Jun 23 2020 17:20:50 GMT+0800 (CST)  
- mongodump --oplog
  - mongodump -uadmin -p123 --port 26060 --authenticationDatabase admin --oplog -o mongodb/full
  - oplog会记录触发备份开始，到备份结束这段时间所有的数据变化。
  - mongorestore -uadmin -p123 --port 26060 --authenticationDatabase admin --oplogReplay  mongodb/full/

#### 生产备份与恢复案例
 ``` bash
 背景：每天0点全备，oplog恢复窗口为48小时
某天，上午10点world.city 业务表被误删除。
恢复思路：
    0、停应用
    2、找测试库
    3、恢复昨天晚上全备
    4、截取全备之后到world.city误删除时间点的oplog，并恢复到测试库
    5、将误删除表导出，恢复到生产库

恢复步骤：
模拟故障环境：

1、全备数据库
模拟原始数据

mongo --port 28017
use wo
for(var i = 1 ;i < 20; i++) {
    db.ci.insert({a: i});
}

全备:
rm -rf /mongodb/backup/*
mongodump --port 28018 --oplog -o /mongodb/backup

--oplog功能:在备份同时,将备份过程中产生的日志进行备份
文件必须存放在/mongodb/backup下,自动命令为oplog.bson

再次模拟数据
db.ci1.insert({id:1})
db.ci2.insert({id:2})


2、上午10点：删除wo库下的ci表
10:00时刻,误删除
db.ci.drop()
show tables;

3、备份现有的oplog.rs表
mongodump --port 28018 -d local -c oplog.rs  -o /mongodb/backup

4、截取oplog并恢复到drop之前的位置
更合理的方法：登陆到原数据库
[mongod@db03 local]$ mongo --port 28018
my_repl:PRIMARY> use local
db.oplog.rs.find({op:"c"}).pretty();

{
    "ts" : Timestamp(1553659908, 1),
    "t" : NumberLong(2),
    "h" : NumberLong("-7439981700218302504"),
    "v" : 2,
    "op" : "c",
    "ns" : "wo.$cmd",
    "ui" : UUID("db70fa45-edde-4945-ade3-747224745725"),
    "wall" : ISODate("2019-03-27T04:11:48.890Z"),
    "o" : {
        "drop" : "ci"
    }
}

获取到oplog误删除时间点位置:
"ts" : Timestamp(1553659908, 1)

 5、恢复备份+应用oplog
[mongod@db03 backup]$ cd /mongodb/backup/local/
[mongod@db03 local]$ ls
oplog.rs.bson  oplog.rs.metadata.json
[mongod@db03 local]$ cp oplog.rs.bson ../oplog.bson 
rm -rf /mongodb/backup/local/
 
mongorestore --port 38021  --oplogReplay --oplogLimit "1553659908:1"  --drop   /mongodb/backup/
```
