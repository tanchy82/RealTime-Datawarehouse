
## 1, 基本介绍 
  maxwell是一个能实时读取MySQL二进制日志binlog，并生成 JSON 格式的消息。    
  作为生产者发送给 Kafka，Kinesis、RabbitMQ、Redis、Google Cloud Pub/Sub、文件或其它平台的应用程序。  
  它的常见应用场景有ETL、维护缓存、收集表级别的dml指标、增量到搜索引擎、数据分区迁移、切库binlog回滚方案等。   
  
  官网地址:http://maxwells-daemon.io  
  
  GitHub地址:https://github.com/zendesk/maxwell
  
  
## 2, binlog介绍
  binlog是mysql当中的二进制日志，主要用于记录对mysql数据库当中的数据发生或潜在发生更改的SQL语句，并以二进制的形式保存在磁盘中。  
  如果后续我们需要配置主从数据库，如果我们需要从数据库同步主数据库的内容，我们就可以通过binlog来进行同步。   
  说白了binlog可以用于解决实时同步mysql数据库当中的数据 binlog的格式也有三种：STATEMENT、ROW、MIXED。 
  
  -STATMENT模式：基于SQL语句的复制(statement-based replication, SBR)，每一条会修改数据的sql语句会记录到binlog中。  
     -优点：不需要记录每一条SQL语句与每行的数据变化，这样子binlog的日志也会比较少，减少了磁盘IO，提高性能。  
     -缺点：在某些情况下会导致master-slave中的数据不一致(如sleep()函数， last_insert_id()，以及 user-defined functions(udf)等会出现问题)  
 
  
