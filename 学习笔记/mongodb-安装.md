- 逻辑结构
  - Mongodb 逻辑结构                         MySQL逻辑结构
  - 库database                                 库
  - 集合（collection）                          表
  - 文档（document）                            数据行
  
- 下载
  - wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel70-4.0.18.tgz
- 安装前准备：
  - 关闭大页内存机制
    ########################################################################
    root用户下
    在vi /etc/rc.local最后添加如下代码
    if test -f /sys/kernel/mm/transparent_hugepage/enabled; then
    echo never > /sys/kernel/mm/transparent_hugepage/enabled
    fi
    if test -f /sys/kernel/mm/transparent_hugepage/defrag; then
    echo never > /sys/kernel/mm/transparent_hugepage/defrag
    fi
            
    cat  /sys/kernel/mm/transparent_hugepage/enabled        
    cat /sys/kernel/mm/transparent_hugepage/defrag  
    其他系统关闭参照官方文档：   

    https://docs.mongodb.com/manual/tutorial/transparent-huge-pages/
    ---------------
    为什么要关闭？
    Transparent Huge Pages (THP) is a Linux memory management system 
    that reduces the overhead of Translation Lookaside Buffer (TLB) 
    lookups on machines with large amounts of memory by using larger memory pages.
    However, database workloads often perform poorly with THP, 
    because they tend to have sparse rather than contiguous memory access patterns. 
    You should disable THP on Linux machines to ensure best performance with MongoDB.
  - 创建mongodb用户
    - useradd mongodb
    - passwd mongodb
  - 解压缩并软链
    - tar -zxvf mongodb-linux-x86_64-rhel70-4.0.19-rc0.tgz
    - ln -s /usr/local/src/mongodb-linux-x86_64-rhel70-4.0.19-rc0 /app/mongodb
  - 创建各种目录并修改属主
    - mkdir /app/mongodb/{conf,log,data}
    - chown -R mongodb.mongodb /app/mongodb
  - 切换用户并修改用户文件
    - su - mongodb
    - vi .bash_profile
      - export PATH=/app/mongodb/bin:$PATH
    - source .bash_profile
    - 启动mongodb服务
      - mongod --dbpath=/app/mongodb/data --logpath=/app/mongodb/log/mongodb.log --port=27017 --logappend --fork
    - 登录测试
      - mongo
  - 创建systemd启动脚本：
    - [Unit]
      Description=mongodb 
      After=network.target remote-fs.target nss-lookup.target
      [Service]
      User=mongod
      Type=forking
      ExecStart=/app/mongodb/bin/mongod --config /app/mongodb/conf/mongo.conf
      ExecReload=/bin/kill -s HUP 
      ExecStop=/app/mongodb/bin/mongod --config /app/mongodb/conf/mongo.conf --shutdown
      PrivateTmp=true  
      [Install]
      WantedBy=multi-user.target
  - 启动mongo
    - /app/mongodb/bin/mongod --config /app/mongodb/conf/mongo.conf
  - 关闭Mongo
    - /app/mongodb/bin/mongod --config /app/mongodb/conf/mongo.conf --shutdown
  
        
