---
title: Canal1.1.5无法连接到MySQL8的解决方案
tags:
  - Canal
  - MySQL
categories:
  - Java学习
abbrlink: 608baaff
date: 2023-05-05 16:10:22
---

按照[官方文档](https://github.com/alibaba/canal/wiki/QuickStart)的配置和Sample代码，canal启动了，sample代码执行成功，但是当我修改数据库里面的数据之后，canal并未获取到数据库的变动。

看了下日志文件 (\canal\logs\example\example.log)，发现有这样一个报错信息：
```java
2023-05-05 15:40:43.320 [destination = example , address = /127.0.0.1:3306 , EventParser] ERROR com.alibaba.otter.canal.common.alarm.LogAlarmHandler - destination:example[com.alibaba.otter.canal.parse.exception.CanalParseException: java.io.IOException: connect /127.0.0.1:3306 failure
Caused by: java.io.IOException: connect /127.0.0.1:3306 failure
	at com.alibaba.otter.canal.parse.driver.mysql.MysqlConnector.connect(MysqlConnector.java:85)
	at com.alibaba.otter.canal.parse.inbound.mysql.MysqlConnection.connect(MysqlConnection.java:90)
	at com.alibaba.otter.canal.parse.inbound.mysql.MysqlEventParser.preDump(MysqlEventParser.java:86)
	at com.alibaba.otter.canal.parse.inbound.AbstractEventParser$1.run(AbstractEventParser.java:176)
	at java.lang.Thread.run(Unknown Source)
Caused by: java.io.IOException: Error When doing Client Authentication:ErrorPacket [errorNumber=1045, fieldCount=-1, message=Access denied for user 'canal'@'localhost' (using password: YES), sqlState=28000, sqlStateMarker=#]
	at com.alibaba.otter.canal.parse.driver.mysql.MysqlConnector.negotiate(MysqlConnector.java:274)
	at com.alibaba.otter.canal.parse.driver.mysql.MysqlConnector.connect(MysqlConnector.java:82)
	... 4 more
]
```
无法连接到MySQL,在身份认证的时候出现了错误。

**原因：** MySQL8默认的身份认证方式为caching_sha2_password，而canal使用的是mysql_native_password

**解决方案：** 在canal配置文件conf/example/instance.properties 中配置的canal.instance.dbUsername用户，必须要指定为mysql_native_password才行。

修改对应用户的认证方式：
```SQL
ALTER USER 'your_username'@'%' IDENTIFIED WITH mysql_native_password BY 'your_password';
```

或者是创建用户的时候就指定认证方式
```sql
/*创建canal用户，指定mysql_native_password加密方式*/
CREATE USER canal IDENTIFIED WITH mysql_native_password BY 'canal';
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
GRANT ALL PRIVILEGES ON *.* TO 'canal'@'%' ;
FLUSH PRIVILEGES;
```
