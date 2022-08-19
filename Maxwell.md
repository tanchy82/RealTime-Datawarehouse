
# **Maxwell**

## 1, 基本介绍 
  maxwell是一个能实时读取MySQL二进制日志binlog，并生成 JSON 格式的消息。    
  作为生产者发送给 Kafka，Kinesis、RabbitMQ、Redis、Google Cloud Pub/Sub、文件或其它平台的应用程序。  
  它的常见应用场景有ETL、维护缓存、收集表级别的dml指标、增量到搜索引擎、数据分区迁移、切库binlog回滚方案等。   
  
  官网地址:http://maxwells-daemon.io  
  
  GitHub地址:https://github.com/zendesk/maxwell
  
## 2, Mysql 必操作
 - my.cnf操作 
  ```
  # /etc/my.cnf
   [mysqld]
    # maxwell needs binlog_format=row
   binlog_format=row
   server_id=1 
   log-bin=master
  ```
 - 库操作
  ```
   mysql> CREATE USER 'maxwell'@'%' IDENTIFIED BY 'XXXXXX';
   mysql> CREATE USER 'maxwell'@'localhost' IDENTIFIED BY 'XXXXXX';

   mysql> GRANT ALL ON maxwell.* TO 'maxwell'@'%';
   mysql> GRANT ALL ON maxwell.* TO 'maxwell'@'localhost';

   mysql> GRANT SELECT, REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO 'maxwell'@'%';
   mysql> GRANT SELECT, REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO 'maxwell'@'localhost';
  ```
  
## 3, binlog介绍
  binlog是mysql当中的二进制日志，主要用于记录对mysql数据库当中的数据发生或潜在发生更改的SQL语句，并以二进制的形式保存在磁盘中。  
  如果后续我们需要配置主从数据库，如果我们需要从数据库同步主数据库的内容，我们就可以通过binlog来进行同步。   
  说白了binlog可以用于解决实时同步mysql数据库当中的数据 binlog的格式也有三种：STATEMENT、ROW、MIXED。 
  
  - STATMENT模式：基于SQL语句的复制(statement-based replication, SBR)，每一条会修改数据的sql语句会记录到binlog中。  
    - 优点：不需要记录每一条SQL语句与每行的数据变化，这样子binlog的日志也会比较少，减少了磁盘IO，提高性能  
    - 缺点：在某些情况下会导致master-slave中的数据不一致(如sleep()函数,last_insert_id()以及user-defined functions(udf)等会出现问题)  
  
  - ROW模式(row-based replication, RBR)：不记录每一条SQL语句的上下文信息，仅需记录哪条数据被修改了，修改成了什么样子了。
    - 优点：不会出现某些特定情况下的存储过程或function、或trigger的调用和触发无法被正确复制的问题
    - 缺点：会产生大量的日志，尤其是alter table的时候会让日志暴涨
  
  - 混合模式复制(mixed-based replication, MBR)：以上两种模式的混合使用  
    一般的复制使用STATEMENT模式保存binlog，对于STATEMENT模式无法复制的操作使用ROW模式保存binlog,MySQL会根据执行的SQL语句选择日志保存方式。
 
  > **因为statement只有sql，没有数据，无法获取原始的变更日志，所以一般建议为ROW模式mysql数据实时同步**

## 4, binlog删除
  当开启MySQL数据库主从时，会产生大量如mysql-bin.00000* log的文件，这会大量耗费您的硬盘空间.可以有三种删除方法:
  - 关闭mysql主从，关闭binlog  
  ```
   # vim /etc/my.cnf  //注释掉log-bin,binlog_format
   # log-bin=mysql-bin
   # binlog_format=row
   然后重启数据库
  ```
  - 开启mysql主从，设置expire_logs_days
    - 重启数据库方式  
      修改expire_logs_days,x是自动删除的天数，一般将x设置为短点; 默认值为0,表示“没有自动删除”
      ```
       # vim /etc/my.cnf
       expire_logs_days = x  
      ```
    - 不重启数据库方式  
      开启mysql主从，直接在mysql里设置expire_logs_days 
      ```
      # /usr/local/mysql/bin/mysql -u root -p
      > show binary logs;
      > show variables like '%log%';
      > set global expire_logs_days = x;
      ```
  - 手动清除binlog文件，> PURGE MASTER LOGS TO ‘MySQL-bin.010′
    - 定制删除 
     ```
      # /usr/local/mysql/bin/mysql -u root -p
      > PURGE MASTER LOGS BEFORE DATE_SUB(CURRENT_DATE, INTERVAL 10 DAY);  //删除10天前的MySQL binlog日志
      > PURGE MASTER LOGS TO 'MySQL-bin.010';  //清除MySQL-bin.010日志
      > PURGE MASTER LOGS BEFORE '2018-11-22 13:00:00';   //清除2018-11-22 13:00:00前binlog日志
      > PURGE MASTER LOGS BEFORE DATE_SUB( NOW( ), INTERVAL 3 DAY);  //清除3天前binlog日志
     ```
    - 重置日志 
     ```
      # /usr/local/mysql/bin/mysql -u root -p
      > reset master;  //清除binlog
     ```
     
 > **清除binlog时，对从mysql的影响**  
 > 如果您有一个活性的从属服务器，该服务器当前正在读取您正在试图删除的日志之一，则本语句不会起作用而是会失败并伴随一个错误。  
 > 不过，如果从属服务器是停止的，并且您碰巧清理了其想要读取的日志之一，则从属服务器启动后不能复制。当从属服务器正在复制时，本语句可以安全运行。您不需要停止它们。 
  

 ## 5, bootstrap 数据重放
   maxwell数据重放分两种模式
   - 命令行操作
     官网提供命令行模式未操作成功,会出现以下异常
     ```
     [root@******** bin]# ./maxwell-bootstrap --config ./config.properties --database **** --table ***
     connecting to jdbc:mysql://127.0.0.1:3306/maxwell?allowPublicKeyRetrieval=true&connectTimeout=5000&zeroDateTimeBehavior=convertToNull
     10:54:49,073 ERROR MaxwellBootstrapUtility - failed to connect to mysql server @ jdbc:mysql://127.0.0.1:3306/maxwell?     allowPublicKeyRetrieval=true&connectTimeout=5000&zeroDateTimeBehavior=convertToNull
     10:54:49,078 ERROR MaxwellBootstrapUtility - Connections could not be acquired from the underlying database!java.sql.SQLException: Connections could not be acquired from the underlying database! at com.mchange.v2.sql.SqlUtils.toSQLException(SqlUtils.java:118)
     ```
       
   - 数据库操作
  
