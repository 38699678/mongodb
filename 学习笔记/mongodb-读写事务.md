## 事务开发 

#### 写事务操作writeConcern
- writeConcern决定一个写操作落到多少个节点上才算成功。writeConcern的取值包括：
  - 0：发起写操作，不关心是否成功
  - 1-n：集群最大数据节点数，写操作需要被复制到界定节点数才算成功
  - majority：写操作需要被复制到大多数节点才算成功
  - 发起写操作的程序将阻塞到写操作到达指定的节点数位置

- writeConcern实验
  - 在复制集测试writeConcern参数
  - db.test.insert( {count: 1},  {writeConcern: {w: "majority"}})
  - db.test.insert( {count: 1},  {writeConcern: {w:3}})
  - db.test.insert( {count: 1},  {writeConcern: {w:4}})
  - 配置延迟节点，模拟网络延迟（复制延迟）
  - conf=rs.conf()
  - conf.members[2].slaveDelay= 5
  - conf.members[2].priority = 0
  - rs.reconfig(conf)
  - 观察复制延迟下的写入，以及timeout参数
  - db.test.insert( {count: 1},  {writeConcern: {w: 3}})
  - db.test.insert( {count: 1},  {writeConcern: {w:3, wtimeout:3000}}

- 注意事项
  - 虽然多于半数的writeConcern都是安全的，但通常只会设置majority，因为这是等待写入延迟时间最短的选择；
  - 不要设置writeConcern等于总节点数，因为一旦有一个节点故障，所有写操作都将失败；
  - writeConcern虽然会增加写操作延迟时间，但并不会显著增加集群压力，因此无论是否等待，写操作最终都会复制到所有节点上。设置writeConcern只是让写操作等待复制后再返回而已；
  - 应对重要数据应用{w: “majority”}，普通数据可以应用{w: 1}以确保最佳性能 
  
#### 读事务操作readPreference
- readPreference：决定使用哪一个节点来满足正在发起的读请求。
  - primary：只选择主节点
  - primaryPreferred：优先选择主节点，如果不可用选择从节点 
  - secondary：只选择从节点
  - secondaryPreferred：优先选择从节点，如果从节点不可用选择主节点。
  - nearest：选择最近的节点，选择网络延时最小的节点。
- readPreference与tag合作，可以将节点选择控制到一个或几个节点
- readPreference 配置
  - 通过MongoDB的连接串参数：
    - mongodb://host1:27107,host2:27107,host3:27017/?replicaSet=rs&readPreference=secondary
  - 通过MongoDB驱动程序API：
    - MongoCollection.withReadPreference(ReadPreference readPref)
  - MongoShell：
    - db.collection.find({}).readPref("secondary")

### readConcern
- readConcern: 选择了指定的节点后，readConcern决定这个节点上的数据哪些是可读的，类似于关系数据库的隔离级别。可选值包括：
  - available: 读取所有可用的数据
  - local：读取所有可用且属于当前分片的数据。（默认）
  - majority：读取在大多数节点上提交完成的数据
  - linearizable: 可线性化读取文档
  - snapshot：读取最近快照中的数据

### readConcern：majority于脏读
#### mongodb中的回滚：
  - 写操作到达大多数节点之前都是不安全的，一旦主节点崩溃，而从节点还没复制到该次操作，刚才的写操作就丢失了。
  - 把一次写操作视为一个事务，从事务的角度，可以认为事务被回滚了。

#### 在可能发生回滚的前提下考虑脏读问题：
  - 如果在一次写操作到达大多数节点前读取了这个写操作，然后因为系统故障操作回滚了，则发生脏读。
  - 使用readConcern：majority可以避免脏读

#### readConcern实现读写分离
  - 向主节点写入一条数据；
  - 提交订单订单确认
  - db.orders.insert({ oid: 101, sku: "kiteboar", q: 1}, {writeConcern:{w: "majority”}})
  - db.orders.find({oid:101}).readPref(“secondary”).readConcern("majority")
