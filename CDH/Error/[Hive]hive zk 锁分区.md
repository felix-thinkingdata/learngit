title: hive zk 锁分区
categories: [CDH,Error]
date: 2017-06-30
---
## 错误日志:
```
Got user-level KeeperException when processing sessionid:0x15bfba733be2c2b type:create cxid:0x8621e1c zxid:0x170a4b8e0d txntype:-1 reqpath:n/a Error Path:/hive_zookeeper_namespace_hive/rptdata/fact_kesheng_sdk_new_device_hourly/src_file_day=20170414/src_file_hour=07 Error:KeeperErrorCode = NodeExists for /hive_zookeeper_namespace_hive/rptdata/fact_kesheng_sdk_new_device_hourly/src_file_day=20170414/src_file_hour=07
```
```
2013-11-04 23:52:40,485 ERROR ZooKeeperHiveLockManager (ZooKeeperHiveLockManager.java:unlockPrimitive(447)) - Failed to release ZooKeeper lock:
org.apache.zookeeper.KeeperException$NoNodeException: KeeperErrorCode = NoNode for /hive_zookeeper_namespace/<hiveDBName>/<Table>/<PARTITION>/LOCK-SHARED-0000000000
```

### 在ZK中可以看到对应的分区锁住了
`/hive_zookeeper_namespace_hive/<hiveDBName>/<Table>/LOCK-SHARED-`

### 在beeline中查看被锁的表和分区
```
show locks <hiveDBName>.<Table>

show locks <hiveDBName>.<Table> partition (src_file_day='20170218' , src_file_our='17'); 
show locks app.mgba_client_event_detail partition(src_file_day='20171114');
unlock table 
```
可以看到 
```
| <hiveDBName>@ugc_90103_bossmonthorderlog_test@<PARTITION>/<PARTITION> | SHARED     |
```

## 解锁table:
```
unlock table <hiveDBName>.<Table>;
```

## 或者在beeline中,强行把配置删除对应table(不建议)
`set hive.support.concurrency=false;`

之后又被锁了一次,详细过程如下:
```
show locks quancheng.ugc_secondline_order_v;

+-----------------------------------+---------+--+
|             tab_name              |  mode   |
+-----------------------------------+---------+--+
| quancheng@ugc_secondline_order_v  | SHARED  |
| quancheng@ugc_secondline_order_v  | SHARED  |
| quancheng@ugc_secondline_order_v  | SHARED  |
+-----------------------------------+---------+--+
set hive.support.concurrency=false;
0: jdbc:hive2://10.200.60.106:10000/default> show locks quancheng.ugc_secondline_order_v;
INFO  : Compiling command(queryId=hive_20170629105454_e0e96d5e-9602-40e5-b100-250ebf6e5bf9): show locks quancheng.ugc_secondline_order_v
INFO  : Semantic Analysis Completed
INFO  : Returning Hive schema: Schema(fieldSchemas:[FieldSchema(name:tab_name, type:string, comment:from deserializer), FieldSchema(name:mode, type:string, comment:from deserializer)], properties:null)
INFO  : Completed compiling command(queryId=hive_20170629105454_e0e96d5e-9602-40e5-b100-250ebf6e5bf9); Time taken: 0.07 seconds
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Executing command(queryId=hive_20170629105454_e0e96d5e-9602-40e5-b100-250ebf6e5bf9): show locks quancheng.ugc_secondline_order_v
INFO  : Starting task [Stage-0:DDL] in serial mode
INFO  : Completed executing command(queryId=hive_20170629105454_e0e96d5e-9602-40e5-b100-250ebf6e5bf9); Time taken: 0.034 seconds
INFO  : OK
+-----------------------------------+---------+--+
|             tab_name              |  mode   |
+-----------------------------------+---------+--+
| quancheng@ugc_secondline_order_v  | SHARED  |
| quancheng@ugc_secondline_order_v  | SHARED  |
| quancheng@ugc_secondline_order_v  | SHARED  |
+-----------------------------------+---------+--+
3 rows selected (0.144 seconds)
0: jdbc:hive2://10.200.60.106:10000/default> drop quancheng.ugc_secondline_order_v;
Error: Error while compiling statement: FAILED: ParseException line 1:5 cannot recognize input near 'drop' 'quancheng' '.' in ddl statement (state=42000,code=40000)
0: jdbc:hive2://10.200.60.106:10000/default> drop view quancheng.ugc_secondline_order_v;
INFO  : Compiling command(queryId=hive_20170629105656_e957e27e-ee27-45ba-853e-ac39660ddf7e): drop view quancheng.ugc_secondline_order_v
INFO  : Semantic Analysis Completed
INFO  : Returning Hive schema: Schema(fieldSchemas:null, properties:null)
INFO  : Completed compiling command(queryId=hive_20170629105656_e957e27e-ee27-45ba-853e-ac39660ddf7e); Time taken: 0.168 seconds
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Executing command(queryId=hive_20170629105656_e957e27e-ee27-45ba-853e-ac39660ddf7e): drop view quancheng.ugc_secondline_order_v
INFO  : Starting task [Stage-0:DDL] in serial mode
INFO  : Completed executing command(queryId=hive_20170629105656_e957e27e-ee27-45ba-853e-ac39660ddf7e); Time taken: 0.061 seconds
INFO  : OK
No rows affected (0.238 seconds)
0: jdbc:hive2://10.200.60.106:10000/default> show tables;
INFO  : Compiling command(queryId=hive_20170629105656_63fcacb8-ffae-4e00-b810-25317cec8058): show tables
INFO  : Semantic Analysis Completed
INFO  : Returning Hive schema: Schema(fieldSchemas:[FieldSchema(name:tab_name, type:string, comment:from deserializer)], properties:null)
INFO  : Completed compiling command(queryId=hive_20170629105656_63fcacb8-ffae-4e00-b810-25317cec8058); Time taken: 0.068 seconds
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Executing command(queryId=hive_20170629105656_63fcacb8-ffae-4e00-b810-25317cec8058): show tables
INFO  : Starting task [Stage-0:DDL] in serial mode
INFO  : Completed executing command(queryId=hive_20170629105656_63fcacb8-ffae-4e00-b810-25317cec8058); Time taken: 0.1 seconds
INFO  : OK
+-----------+--+
| tab_name  |
+-----------+--+
+-----------+--+
```

```
>show locks ods.migulive_10108_func_use_log_ex;

+-------------------------------------+---------+--+
|              tab_name               |  mode   |
+-------------------------------------+---------+--+
| ods@migulive_10108_func_use_log_ex  | SHARED  |
| ods@migulive_10108_func_use_log_ex  | SHARED  |
| ods@migulive_10108_func_use_log_ex  | SHARED  |
+-------------------------------------+---------+--+
3 rows selected (0.27 seconds)
>show locks ods.migulive_10108_func_use_log_ex partition(src_file_day='20171114', src_file_hour='00');
+----------------------------------------------------+---------+--+
|                      tab_name                      |  mode   |
+----------------------------------------------------+---------+--+
| ods@migulive_10108_func_use_log_ex@src_file_day=20171114/src_file_hour=00 | SHARED  |
| ods@migulive_10108_func_use_log_ex@src_file_day=20171114/src_file_hour=00 | SHARED  |
| ods@migulive_10108_func_use_log_ex@src_file_day=20171114/src_file_hour=00 | SHARED  |
+----------------------------------------------------+---------+--+

```
## 解锁分区
`unlock table ods.migulive_10108_func_use_log_ex partition(src_file_day='20171114', src_file_hour='00');`

```
> unlock table ods.migulive_10108_func_use_log_ex partition(src_file_day='20171114', src_file_hour='00');
INFO  : Compiling command(queryId=hive_20171115221717_540d811a-473a-443a-91c3-e5aa545a2bc1): unlock table ods.migulive_10108_func_use_log_ex partition(src_file_day='20171114', src_file_hour='00')
INFO  : Semantic Analysis Completed
INFO  : Returning Hive schema: Schema(fieldSchemas:null, properties:null)
INFO  : Completed compiling command(queryId=hive_20171115221717_540d811a-473a-443a-91c3-e5aa545a2bc1); Time taken: 0.083 seconds
INFO  : Executing command(queryId=hive_20171115221717_540d811a-473a-443a-91c3-e5aa545a2bc1): unlock table ods.migulive_10108_func_use_log_ex partition(src_file_day='20171114', src_file_hour='00')
INFO  : Starting task [Stage-0:DDL] in serial mode
INFO  : Completed executing command(queryId=hive_20171115221717_540d811a-473a-443a-91c3-e5aa545a2bc1); Time taken: 0.025 seconds
INFO  : OK
No rows affected (0.117 seconds)
> unlock table ods.migulive_10108_func_use_log_ex partition(src_file_day='20171114', src_file_hour='01');
INFO  : Compiling command(queryId=hive_20171115221717_a3869646-831b-4d37-9344-3c8a598307c2): unlock table ods.migulive_10108_func_use_log_ex partition(src_file_day='20171114', src_file_hour='01')
INFO  : Semantic Analysis Completed
INFO  : Returning Hive schema: Schema(fieldSchemas:null, properties:null)
INFO  : Completed compiling command(queryId=hive_20171115221717_a3869646-831b-4d37-9344-3c8a598307c2); Time taken: 0.084 seconds
INFO  : Executing command(queryId=hive_20171115221717_a3869646-831b-4d37-9344-3c8a598307c2): unlock table ods.migulive_10108_func_use_log_ex partition(src_file_day='20171114', src_file_hour='01')
INFO  : Starting task [Stage-0:DDL] in serial mode
INFO  : Completed executing command(queryId=hive_20171115221717_a3869646-831b-4d37-9344-3c8a598307c2); Time taken: 0.025 seconds
INFO  : OK
No rows affected (0.117 seconds)
```

