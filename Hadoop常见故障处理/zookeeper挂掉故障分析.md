# 故障现象：
	
	zk一直在循环的leader选举，但一直选举失败，导致zk不可用
	日志报错：java.io.IOException: ZooKeeperServer not running

# 故障原因：

	最终分析发现zk数据同步未完成及一直在进行leader选举导致zk一个多小时不可用。
	共三台zk，zk01和zk03的数据目录大小是4.3G，但是zk02确是2.6G，首先重启了三四次，结果还是一样，一直在leader选举，并且看了一下gc情况，FullGC挺频繁的，然后就想试了试能不能先恢复1台或者2台，但是也启动不起来，不管是启动单独一个节点还是启动2个数据同步貌似正常的节点，结果都是一样的报错。网上很多人说脏数据导致无法启动，删除数据就可以恢复了，但是zk里面的元数据丢失的话，所有依赖于zk的服务都会无法恢复
	
	所以还是继续找为什么zk02同步不到最新的数据。zk在启动时会进行leader选举，同时follower节点需要获取leader节点的snapshot，并且要在initLimit时间内同步完成，如果没同步完成就无法启动，会出现前面看到的现象每过3分钟会进行一次leader选举，一直无法提供服务。我看了一下一个snapshot大概1.6G，默认的initLimit是10s，而我们的带宽是1Gbps，最后看三台zk服务器的带宽情况，近两个小时带宽都是跑满的

# 解决办法：

	首先重启了三四次，不行，然后观察到频繁FullGC，就加大了zk的jvm内存，由原来的2G加到了6G，重启后GC没那么频繁了，但还是会有FullGC。依然一直在选举不提供服务。
	
	最后试试加大数据同步initLimit时间，由默认的10加到50，同时停止CDH集群，单独启动zk，启动完后观察了一下zk02的数据目录，过来一会儿就同步到了4.3G数据了。等了一会儿leader也选举出来了，再启动集群就正常了
	
# 疑问：

	这里有个疑问是为什么snapshot会突然增长那么大？以前的snapshot一般只有几十M，恢复正常后snapshot也只有几十M
	所以试着通过java -cp /alidata1/cloudera/parcels/CDH/lib/zookeeper/zookeeper.jar:/alidata1/cloudera/parcels/CDH/lib/zookeeper/lib/* org.apache.zookeeper.server.SnapshotFormatter /var/lib/zookeeper/version-2/snapshot.eb00026fab来分析里面有什么数据，但是貌似看不出来

# 真实原因

通过上面命令格式化导出了所有zk数据，有大量如下数据，这是hive共享锁，不知道为什么hive共享锁会占用这么大的zk存储

```
/hive_zookeeper_namespace_hive/banma/LOCK-SHARED-0003702236
  cZxid = 0x0000eb00064c31
  ctime = Wed Nov 27 17:45:15 CST 2019
  mZxid = 0x0000eb00064c31
  mtime = Wed Nov 27 17:45:15 CST 2019
  pZxid = 0x0000eb00064c31
  cversion = 0
  dataVersion = 0
  aclVersion = 0
  ephemeralOwner = 0x26eabb6e5880068
  dataLength = 1000090
  ```
