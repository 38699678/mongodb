#### 分片集群
![avatar](img/sharded-cluster-production-architecture.bakedsvg.svg)
- 原理
  - 分布式架构
  - mongos负责接收请求
  - configserver负责记录配置信息，分配mongos请求
  - shard负责存储数据
  - mongos接受客户的写入请求，他是不知道写道哪个shard的。他先去configservers询问，由于configservers存储了所有节点的配置信息。他返回给mongos你把数据写到shard1吧。mongos就把数据写入到shard1了

#### 部署流程 
- 架构说明
  - 物理机： 
    - node1 192.168.120.128
    - node2 192.168.120.130
    - node3 192.168.120.131
  - config
    - 192.168.120.128:38018
    - 192.168.120.130:38019
    - 192.168.120.137:38020
  - shard1
    - 192.168.120.128:48018
    - 192.168.120.130:48019
    - 192.168.120.131:48020
  - shard2
    - 192.168.120.128:48021
    - 192.168.120.130:48022
    - 192.168.120.131:48023
  - router:
    - 192.168.120.127:38017
- 部署node1 shard1
  - 编写配置文件
  ```
  systemLog:
    destination: file
    path: /app/mongodb/48018/log/mongodb.log   
    logAppend: true
  storage:
    journal:
      enabled: true
    dbPath: /data/mongodb/48018/data
    directoryPerDB: true
  #engine: wiredTiger
    wiredTiger:
      engineConfig:
        cacheSizeGB: 1
        directoryForIndexes: true
      collectionConfig:
        blockCompressor: zlib
      indexConfig:
        prefixCompression: true
  net:
    bindIp: 192.168.120.128,127.0.0.1
    port: 48018
  replication:
    oplogSizeMB: 2048
    replSetName: sh1
  sharding:
    clusterRole: shardsvr
  processManagement: 
    fork: true
  ```
  - 创建日志和数据目录
    - mkdir /app/mongodb/48018/log/ /data/mongodb/48018/data -p
  - 启动服务
    - mongod -f sh-48018.conf
- 部署node1 shard2
  - 编写配置文件
  ``` bash
  systemLog:
    destination: file
    path: /app/mongodb/48021/log/mongodb.log   
    logAppend: true
  storage:
    journal:
      enabled: true
    dbPath: /data/mongodb/48021/data
    directoryPerDB: true
  #engine: wiredTiger
    wiredTiger:
      engineConfig:
        cacheSizeGB: 1
        directoryForIndexes: true
      collectionConfig:
        blockCompressor: zlib
      indexConfig:
        prefixCompression: true
  net:
    bindIp: 192.168.120.128,127.0.0.1
    port: 48018
  replication:
    oplogSizeMB: 2048
    replSetName: sh2
  sharding:
    clusterRole: shardsvr
  processManagement: 
    fork: true
  ``` 
  - 创建目录
    - mkdir  /app/mongodb/48021/log/  /data/mongodb/48021/data -p
  - 启动sha2
    - mongod -f /app/mongodb/conf/sh-48021.conf
- 部署node1 configserver
  - 编写configserve配置文件
  ``` bash
  systemLog:
    destination: file
    path: /app/mongodb/38018/log/mongodb.conf
    logAppend: true
  storage:
    journal:
      enabled: true
    dbPath: /data/mongodb/38018/data
    directoryPerDB: true
    #engine: wiredTiger
    wiredTiger:
      engineConfig:
        cacheSizeGB: 1
        directoryForIndexes: true
      collectionConfig:
        blockCompressor: zlib
      indexConfig:
        prefixCompression: true
  net:
    bindIp: 192.168.120.128,127.0.0.1
    port: 38018
  replication:
    oplogSizeMB: 2048
    replSetName: configReplSet
  sharding:
    clusterRole: configsvr
  processManagement: 
    fork: true
  ```
  -创建日志和数据目录
    - mkdir /app/mongodb/38018/log/ /data/mongodb/38018/data -p 
  - 启动服务
    - mongod -f /app/mongodb/conf/cs-38018.conf
- 其他节点也按照以上方式部署。并启动服务
- 配置mongo各副本集关系
  - 配置shard1
  ``` bash 
  #mongo --port 48018 admin
  use admin
  config = {_id: 'sh1', members: [
                          {_id: 0, host: '192.168.120.128:48018'},
                          {_id: 1, host: '192.168.120.130:48019'},
                          {_id: 2, host: '192.168.120.131:48020',"arbiterOnly":true}]
           }

  rs.initiate(config)
  ```
- 配置shard2
  ``` bash
  #mongo --port 48021 admin
  use admin 
  config = {_id: 'sh2', members: [
                          {_id: 0, host: '192.168.120.128:48021'},
                          {_id: 1, host: '192.168.120.130:48022'},
                          {_id: 2, host: '192.168.120.131:48023',"arbiterOnly":true}]
           }
  rs.initiate(config)
  ```
- 配置configservers
  ``` bash
  #mongo --port 38018
  use  admin
  config = {_id: 'configReplSet', members: [
                          {_id: 0, host: '192.168.120.128:38018'},
                          {_id: 1, host: '192.168.120.130:38019'},
                          {_id: 2, host: '192.168.120.131:38020'}]
           }
  rs.initiate(config)  
  ```
- 部署node1 mongos
  - 编写配置文件
  ``` bash 
  systemLog:
    destination: file
    path: /app/mongodb/38017/log/mongos.log
    logAppend: true
  net:
    bindIp: 10.0.0.51,127.0.0.1
    port: 38017
  sharding:
    #此处设置configserver地址，需要在配置文件中指定
    configDB: configReplSet/192.168.120.128:38018,192.168.120.130:38019,192.168.120.131:38020
  processManagement: 
    fork: true
- 启动mongos服务
  - mongos -f mongos-38017.conf
- 配置mongos，添加shard记录
    - mongo 192.168.120.128:38017/admin
    - use admin
    - db.runCommand({addshard:"sh1/192.168.120.128:48018,192.168.120.130:48019,192.168.120.131:48020"})
    - db.runCommand({addshard:"sh2/192.168.120.128:48021,192.168.120.130:48022,192.168.120.131:48023"})
    - db.runCommand({listshards:1}) #列出所有切片
    - sh.status()
#### 使用分片集群 
- RANGE分片配置
    - 激活mongodb分片功能
        - mongo --port 38017 admin
        - use admin
        - db.runCommand({enablesharding:"test"}) #设置test库启用分片功能
    - 指定分片键对集合分片
        - 创建索引
        - use test
        - db.vast.ensureIndex( { id: 1 } )
        - 开启分片
        - use admin
        - db.runCommand( { shardcollection : "test.vast",key : {id: 1} } )
    - 集合分片验证
        - use test
        - for(i=1;i<1000000;i++){ db.vast.insert({"id":i,"name":"shenzheng","age":70,"date":new Date()}); }
        - db.vast.stats()
- hash分片设置
    - 对于test1开启分片功能
      - mongo --port 38017 admin
      - use admin
      - db.runCommand( { enablesharding : "test1" } )
    - 对于test1库下的vast表建立hash索引
      - use test1
      - db.vast.ensureIndex( { id: "hashed" } )
    - 开启分片 
      - use admin
      - sh.shardCollection( "test1.vast", { id: "hashed" } )
    - 录入10w行数据测试
      - use test1
      - for(i=1;i<100000;i++){ db.vast.insert({"id":i,"name":"shenzheng","age":70,"date":new Date()}); }
    - hash分片结果测试
      - mongo --port 48018
        use test1
        db.vast.count();
        mongo --port 48018
        use test1
        db.vast.count();
- 分片集群的查询及管理
    - 判断是否Shard集群
      - db.runCommand({ isdbgrid : 1})
    - 列出所有分片信息
      - db.runCommand({ listshards : 1})
    - 列出开启分片的数据库
      - use config
      - db.databases.find( { "partitioned": true } )
      - 或者：
      - db.databases.find() //列出所有数据库分片情况
    - 查看分片的片键
      - db.collections.find().pretty()
        {
            "_id" : "test.vast",
            "lastmodEpoch" : ObjectId("58a599f19c898bbfb818b63c"),
            "lastmod" : ISODate("1970-02-19T17:02:47.296Z"),
            "dropped" : false,
            "key" : {
                "id" : 1
            },
            "unique" : false
        }
    - 查看分片的详细信息
      - sh.status()
    - 删除分片节点（谨慎）
      - 确认blance是否在工作
        - sh.getBalancerState()
    - 删除shard2节点(谨慎)
      - db.runCommand( { removeShard: "shard2" } )
    - 注意：删除操作一定会立即触发blancer。
