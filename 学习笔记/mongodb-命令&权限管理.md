- mongo默认库
    - test：默认登陆时存在的库
    - admin库：系统预留库，mongodb系统管理库
    - local库：本地预留库，存储关键日志
    - config库：mongodb配置信息库


#### 基本命令
  - show databases / show dbs
  - show tables / show collections
  - use admin 
  - db / select database()

- 命令总类
  - db相关
    - db.[tab]+[tab] 查看所有db级命令
    - db.help()     查看db级命令的简单帮助
    - db.test.[tab]+[tab]
    - db.test.help()
  - rs复制相关
    - rs.[tab]+[tab]
    - rs.help()
  - sh分片相关
    - sh.[tab]+[tab]
    - sh.help()

- mongodb的对象操作
  - mongodb中只要你use到一个没有的库，系统就会自动生成一个临时库。如果你在库中建集合的话，这个库就被持久化了。
  - 库的操作
    - 删除：
      - use test
      - db.dropDatabase() 删除当前的test库 
  - 集合的操作：
    - 语法：
      - db.<collection>.insertOne(<json>)
      - db.<collection>.insertMany([<json>,<json 2>,<json 3>])
    - 建表：
      - db.createCollection('t2')
        { "ok" : 1 }
      - 方法2： 当你插入一个不存在的表中一条数据，这个表就建立了
        - db.t4.insert({a:100})
    - 插入：
      - mongo中，每条记录都是一个json数据。
      - db.t4.insert({a:100})
  - 文档操作
    - 数据批量录入：
      - for(i=0;i<10000;i++){  
        db.log.insert(  
            {  
            "uid":i,"name":"mongodb","age":6,"date":new Date()  
            }  
        )  
        }  
    - 统计数据行数：
      - db.t4.count()
        1
    - 全表查询，mongodb的全表查询是分页的。每页只显示20条记录
      - db.t4.find()
        { "_id" : ObjectId("5ee444ea9e410b14cd259b75"), "a" : 100 }
      - 每页显示50条记录：DBQuery.shellBatchSize=50
    - 按条件查询
      - > db.log.find({uid:5})  
        { "_id" : ObjectId("5ee446a49e410b14cd259b7b"), "uid" : 5, "name" : "mongodb", "age" : 6, "date" : ISODate("2020-06-13T03:23:16.455Z") }
    - 多添加查询
      - 多添加and查询
        - db.movies.find({"year":1989,"title":'batman'}) 
        - db.movies.find({$and:[{"title":"batman"},{"category":"action"}]})
      - or
        - db.movies.find({$or:[{"year":1989},{"title":"batman"}]})
      - 按正则表达式
        - db.movies.find({"title":/^b/}) 
    - 查询条件对照：
      - a = 1     {a:1}
      - a <> 1    {a:{$ne: 1}
      - a > 1     {a: {$gt: 1}}
      - a >= 1    {a: {$ge: 1}}
      - a < 1     {a: {$lt: 1}}
      - a <= 1    {a: {$le: 1}}
      - a =1 and b =1 {$and: [{a: 1},{b: 1}]}
      - a = 1 or b =1 {$or: [{a: 1},{b: 1}]}
      - a is null {a: {$exists: false}}
      - a in (1,2,3)  {a: {$in: [1,2,3]}}
    - 是用find搜索子文档
      - db.fruit.insertOne({name: "apple",from: {country: "China",province: "Guangdon"}})
      - db.fruit.find({from: {country: 'china'}})
      - db.fruit.find({from.country: "china"})
    - 以json格式化输出
      - db.log.find({uid:5}).pretty()  
        {
            "_id" : ObjectId("5ee446a49e410b14cd259b7b"),  
            "uid" : 5,  
            "name" : "mongodb",  
            "age" : 6,    
            ("2020-06-13T03:23:16.455Z")  
        }
    - 删除记录
      - db.log.remove({})  #全表删除
        WriteResult({ "nRemoved" : 10000 })
      - db.log.remove({uid:10})  #删除单条记录
        WriteResult({ "nRemoved" : 1 })
    - 查看集合存储信息
      - db.log.totalSize()  #该大小包含集合中索引+压缩数据的总大小
        724992

#### 用户权限管理
- 验证库：
  - 在mongodb建立普通用户时，要先use到的库，在使用该用户登陆时，要加上验证库才能登陆
  - 对于管理员必须要use到admin库建立
- 注意：
  - 建立用户时，use到的库就是他的验证库
  - 登陆时，必须指定验证库才能登陆
  - 管理员的验证库是admin，普通用户的验证库一般是该用户管理的库 
  - 如果直接登陆数据库，不需进行use，默认是test库。
  - 从3.6版本，不在配置文件中添加bindIp参数，不可远程登陆，只能本地登陆
  - 创建用户时，一个用户只能登陆和管理一个库，一个会话不能连接多个库，否则会造成数据耦合太高
- 角色role：
  - 数据库用户角色：read、readWrite;
  - 数据库管理角色：dbAdmin、dbOwner、userAdmin；
  - 集群管理角色：clusterAdmin、clusterManager、clusterMonitor、hostManager；
  - 备份恢复角色：backup、restore；
  - 所有数据库角色：readAnyDatabase、readWriteAnyDatabase、userAdminAnyDatabase、dbAdminAnyDatabase
  - 超级用户角色：root
　 // 这里还有几个角色间接或直接提供了系统超级用户的访问（dbOwner 、userAdmin、userAdminAnyDatabase）
  - 内部角色：__system PS：关于每个角色所拥有的操作权限可以点击上面的内置角色链接查看详情。
- 创建用户语法：
``` bash
use admin 
db.createUser
{
    user: "<name>",
    pwd: "<cleartext password>",
    roles: [
       { role: "<role>",
     db: "<database>" } | "<role>",
    ...
    ]
}
```
- 参数分析：
  - user: 用户名
  - pwd： 密码
  - roles：角色
  - db: 作用对象
  - role：权限 root,readRwrite，read
  - use
  - db.createUser({user:"testuser",pwd:"123",roles:[{role:"readWrite",db:"test"}]})
- 示例
  - use test
  - db.createUser({user:"testuser",pwd:"123",roles:[{role:"readWrite",db:"test"}]}) 
- 验证登陆 
  - mongo -u testuser -p123  192.168.120.128/test
- 创建管理员账户：管理所有数据库，必须use admin再创建
  - use admin
  - db.createUser({user:"admin",pwd:"123",roles:[{role:"root",db:"admin"}]})
- 验证用户：
  - db.auth("admin",'123')
- 修改用户密码：
  - db.changeUserPassword('admin','tUDfqjDWHR4hSIXs')
- 查看用户：
  - db.system.users.find().pretty()
- 删除用户
  - use test  
    switched to db test  
    db.dropUser('testuser')  
    true  
