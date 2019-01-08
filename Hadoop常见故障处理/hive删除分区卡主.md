hive -hiveconf hive.root.logger=DEBUG,console -e "alter table xx.xx drop IF EXISTS partition(pt=201811240030);"

开启debug日志，发现一直在等待锁或者说是在获取锁

```
19/01/08 12:33:51 [main]: DEBUG ZooKeeperHiveLockManager: Acquiring lock for xx/xx/pt=201811240050 with mode EXCLUSIVE IMPLICIT
....
19/01/08 12:33:51 [main-SendThread(xxx:2181)]: DEBUG zookeeper.ClientCnxn: Reading reply sessionid:0x267a1df9c294a03, 
packet:: clientPath:null serverPath:null ###finished:false### header:: 10,12  reply
....
```

一直在等待锁，好奇怪

```
SHOW LOCKS xx
```

发现表没有锁住，然后看分区

```
SHOW LOCKS xx PARTITION(pt=201811240030);

OK
xx@xx@pt=201811240020	SHARED
Time taken: 0.152 seconds, Fetched: 1 row(s)

```
发现是分区锁了，多看了几个，发现所有分区都锁住了，原因应该是开发那边合并小文件的程序导致的

逐个分区解锁后，就可以删除分区了

```
unlock table xx PARTITION(pt=201811240030);
```

