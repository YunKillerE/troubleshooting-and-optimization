# How should one solve "Bad connect ack with firstBadLink" DataNode problem in Hadoop

https://www.quora.com/How-should-one-solve-Bad-connect-ack-with-firstBadLink-DataNode-problem-in-Hadoop

http://www.larsgeorge.com/2012/03/hadoop-hbase-and-xceivers.html

1、Firstly, check [Deprecated Properties](https://hadoop.apache.org/docs/r2.4.1/hadoop-project-dist/hadoop-common/DeprecatedProperties.html)

make sure your Hadoop’s version which supported “dfs.datanode.max.xcievers” or “dfs.datanode.max.transfer.threads” in hdfs-site.xml;

2、Secondly, on my own, config below which according to [recommendation](https://community.hortonworks.com/questions/30140/is-there-any-recommendation-for-dfsdatanodemaxtran.html):

```
<property>
 <name>dfs.datanode.max.transfer.threads</name>
 <value>16384</value>
</property>
```

3、Finally ,restart your datanode services.

