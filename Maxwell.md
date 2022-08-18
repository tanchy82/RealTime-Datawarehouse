
## 1, 基本介绍 
  maxwell是一个能实时读取MySQL二进制日志binlog，并生成 JSON 格式的消息.
  作为生产者发送给 Kafka，Kinesis、RabbitMQ、Redis、Google Cloud Pub/Sub、文件或其它平台的应用程序.
  它的常见应用场景有ETL、维护缓存、收集表级别的dml指标、增量到搜索引擎、数据分区迁移、切库binlog回滚方案等.
  
  官网地址:http://maxwells-daemon.io
  GitHub地址:https://github.com/zendesk/maxwell
  
## 2, binlog介绍
 
  
