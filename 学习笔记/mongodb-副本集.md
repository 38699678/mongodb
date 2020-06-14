#### 普通的复制集
- 原理
  - 基本构成是1主2从的结构，自带互相监控投票机制（Raft（MongoDB）  Paxos（mysql MGR 用的是变种）。如果发生主库宕机，复制集内部会进行投票选举，选择一个新的主库替代原有主库对外提供服务。同时复制集会自动通知。客户端程序，主库已经发生切换了。应用就会连接到新的主库
![avater](img/replica-set-primary-with-two-secondaries.bakedsvg.svg)
- 规划
  - 三台centos7 
    - node1： 192.168.120.128 port： 28017
    - node2： 192.168.120.130 port： 28018
    - node3： 192.168.120.131 port： 28019
- 部署
  - 在三个节点部署mongodb
  - 编辑配置文件rs-28017.conf
    ``` bash   
    systemLog:
      destination: file
      path: /app/mongodb/28017/log/mongodb.log
      logAppend: true
    storage:
      journal:
        enabled: true
      dbPath: /data/mongodb/28017/data
      directoryPerDB: true
    #engine: wiredTiger mongodb的默认存储引擎
    wiredTiger:
        engineConfig:
          cacheSizeGB: 1
          directoryForIndexes: true
        collectionConfig:
          blockCompressor: zlib
        indexConfig:
          prefixCompression: true
    processManagement:
      fork: true
    net:
      bindIp: 192.168.120.128,127.0.0.1
      port: 28017
    replication:
      #此处类似与mysql的binlog，目前指定2G大小
      oplogSizeMB: 2048
      #指定主从复制的名称，这块自定义，不过要和后面的命令相同
      replSetName: my_repl
    ```
    - 创建数据和日志目录
      - mkdir -p /app/mongodb/28017/log/ /data/mongodb/28017/data
    - 启动node1服务
      - mongod -f /app/mongodb/conf/rs-28017.conf 
    - 按照以上部署在其他两个节点中部署mongodb
      - 修改bindIp，port，还有配置文件涉及端口的选项
  - 所有节点都部署并成功启动后，在node1中初始化复制集群。
    - mongo --port 28018 admin
    - use admin 
    - config = {_id: 'my_repl', members: [
                          {_id: 0, host: '192.168.120.128:28017'},
                          {_id: 1, host: '192.168.120.130:28018'},
                          {_id: 2, host: '192.168.120.131:28019'}]
          }     
    - rs.initiate(config) #现在集群已经部署完成了，其他节点会自动加入该集群中
    - rs.status() #查看集群状态
  - 此时slave节点是没有权限读取master传过来的信息
    - 在slave节点 rs.slaveOk() 可以允许slave节点在当前连接会话中有读取master的数据权限

#### 部署只能投票不能复制的节点 
- 在上面的架构中，我们有两个从复制节点，接受主节点的更新复制。由于两个节点复制负责，可能造成io的浪费。官方建议我们只部署一个副本集，另一个节点只做投票。
![avater](img/replica-set-primary-with-secondary-and-arbiter.bakedsvg.svg)

- 部署arbiter节点
  - 如果原来没有部署过副本集,在生成primary的时候，直接指定arbiter节点信息
    - use admin 
    - config = {_id: 'my_repl', members: [  
                          {_id: 0, host: '192.168.120.128:28017'},  
                          {_id: 1, host: '192.168.120.130:28018'},  
                          {_id: 2, host: '192.168.120.131:28019',"arbiterOnly":true}]  }                  
    - rs.initiate(config) 
  - 如果原来已有3节点的副本集，想将其中一个节点改为arbiter。
    - use admin
    - rs.remove("ip:port")
    - 然后清空该节点的数据并重启
    - rs.addArb("192.168.120.131:28019")
    - rs.status() 查看副本集状态信息
    ``` bash
    	"_id" : 2,
			"name" : "192.168.120.131:28019",
			"health" : 1,
			"state" : 7,
			"stateStr" : "ARBITER",
    ```

#### 特殊从节点
- 特殊从节点：
  - arbiter：主要负责选主过程中的投票，但是不存储任何数据，也不提供任何服务
  - hidden：隐藏节点，不参与选主，也不对外提供服务
  - delay：延时节点，数据落后于主库一段时间，因为数据是延时的，也不应该提供服务或参与选主，所以通常会配合hidden（隐藏）
- 部署延时节点，其中的members号码为副本集中第几个副本
  - cfg=rs.conf()                   #定义配置为cfg变量
  - cfg.members[2].priority=0       #定义优先级权重为0，不参与选主
  - cfg.members[2].hidden=true      #设为隐藏节点，不参与业务
  - cfg.members[2].slaveDelay=120   #定义延时的秒杀
  - rs.reconfig(cfg)                #重新加载配置

- 取消以上配置
  - cfg=rs.conf()   
  - cfg.members[2].priority=1   
  - cfg.members[2].hidden=false 
  - cfg.members[2].slaveDelay=0
  - rs.reconfig(cfg)    
- 配置成功后，通过以下命令查询配置后的属性
  - rs.conf(); 

#### 复制集管理
- 查看复制集状态
  - rs.status() #查看完整replication状态i西南西
  - rs.isMaster() #判断节点是否为master节点
  - rs.conf #查看配置信息
- 添加删除节点
  - rs.add("ip:port") #添加普通副本节点
  - rs.addArb("ip:port") #添加arbiter节点
  - rs.remove("ip:port")
- 副本集角色切换（不要人为随便操作）
  - rs.stepDown()
  - rs.freeze(300) //锁定从，使其不会转变成主库.freeze()和stepDown单位都是秒。
- 设置副本节点可读：在副本节点执行
  - rs.slaveOk()
- 查看副本节点（监控主从延时）
  - rs.printSlaveReplicationInfo()  
    source: 192.168.120.130:28018  
	syncedTo: Sun Jun 14 2020 13:54:39 GMT+0800 (CST)  
	0 secs (0 hrs) behind the primary   
