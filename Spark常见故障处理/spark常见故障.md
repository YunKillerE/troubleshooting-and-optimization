工作中遇到的一些错误记录总结

# 一.内存问题导致，也就是常说的OOM

1. driver OOM

*报错：* Job aborted due to stage failure: Total size of serialized results of 334502 tasks (1024.0 MB) is bigger than spark.driver.maxResultSize (1024.0 MB) 

*处理：* 对于collect和一些操作，driver会接收各task执行后的数据，spark.driver.maxResultSize参数控制接收数据大小，建议先检查代码，避免或减少take，collect操作，如果不成功再考虑增大该参数。尽量不要使用collect操作即可。

2. executor OOM

*报错：* OOM

*处理：*

* 可以按内存优化的方法增加code使用内存空间
* 增加executor内存总量,也就是说增加spark.executor.memory的值
* 增加任务并行度(大任务就被分成小任务了)
        
参考spark性能调优基础篇里面的内存条有这一块

3. GC开销超过限制

*报错：* java.lang.OutOfMemoryError: GC overhead limit exceeded at scala.collection.immutable.HashMap.scala.collection.immutable.HashMap.makeHashTrieMap(HashMap.scala:175) 

*处理：* 分为两个角度，一是是检查代码，减少不必要的冗余，重用的RDD要序列化缓存，减少shuffle数据，加大并行度；二从参数配置看，加大executor内存，增加shuffle buffer缓存，但有时候也因为job写的太低效而出现无效。

4. 空指针异常

*报错：* java.lang.NullPointerException at com.immomo.recommend.recommend_molive$$anonfun$1.apply(recommend_molive.scala:83)、 
*处理：* 该问题一般是代码中的，检查数组，对象内容是否可能为空；尤其是表数据，能有字段的值为null，但没有处理null，出现这个错误。

5. kyro 缓存溢出

*报错：* java.lang.OutOfMemoryError: Java heap space at com.esotericsoftware.kryo.io.Output.require(Output.java:168) 
*处理：* 该报错堆栈可以看到是kyro请求空间，结果不够出现溢出，因为kyro序列化器能序列化的单个对象最大限制为spark.kryoserializer.buffer.max定义，这个值最大为2g。所以建议优先检查代码中的大对象，想办法裁剪对象大小，如果不行再考虑增大spark.kryoserializer.buffer.max数值。

6. Executor & Task Lost

*报错：* 

* executor lost : WARN TaskSetManager: Lost task 1.0 in stage 0.0 (TID 1, aa.local): ExecutorLostFailure (executor lost) 
* task lost : WARN TaskSetManager: Lost task 69.2 in stage 7.0 (TID 1145, 192.168.47.217): java.io.IOException: Connection from /192.168.47.217:55483 closed 
* 各种timeout : java.util.concurrent.TimeoutException: Futures timed out after [120 second 
* 各种timeout : ERROR TransportChannelHandler: Connection to /192.168.47.212:35409 has been quiet for 120000 ms while there are outstanding requests. Assuming connection is dead; please adjust spark.network.timeout if this is wrong 

*处理：* 

提高 spark.network.timeout 的值，根据情况改成300(5min)或更高。

默认为 120(120s),配置所有网络传输的延时，如果没有主动设置以下参数，默认覆盖其属性
```
spark.core.connection.ack.wait.timeout
spark.akka.timeout
spark.storage.blockManagerSlaveTimeoutMs
spark.shuffle.io.connectionTimeout
spark.rpc.askTimeout or spark.rpc.lookupTimeout
```

7. org.apache.spark.shuffle.FetchFailedException

这种问题一般发生在有大量shuffle操作的时候,task不断的failed,然后又重执行，一直循环下去，非常的耗时

*报错：* 

* missing output location : org.apache.spark.shuffle.MetadataFetchFailedException: Missing an output location for shuffle 0 
* shuffle fetch faild : org.apache.spark.shuffle.FetchFailedException: Failed to connect to spark047215/192.168.47.215:50268 

当前的配置为每个executor使用1cpu,5GRAM,启动了20个executor

*处理：* 一般遇到这种问题提高executor内存即可,同时增加每个executor的cpu,这样不会减少task并行度。

    spark.executor.memory 15G
    spark.executor.cores 3
    spark.cores.max 21

启动的execuote数量为:7个

    execuoteNum = spark.cores.max/spark.executor.cores 

每个executor的配置：
    
    3core,15G RAM 

消耗的内存资源为:105G RAM

    15G*7=105G 

可以发现使用的资源并没有提升，但是同样的任务原来的配置跑几个小时还在卡着，改了配置后几分钟就结束了。

# 二. 数据倾斜

1. 大多数任务都完成了，还有那么一两个任务怎么都跑不完或者跑的很慢。分为数据倾斜和task倾斜两种。

*报错：*  数据倾斜

<img src="http://s1.51cto.com/wyfs02/M02/8F/9B/wKioL1jm5vjDQfWcAAC_LXFYzAk842.jpg" width = "600" height = "400" alt="来源：http://www.jianshu.com/p/2b23a3fb479d" align=center />

*处理：* 数据倾斜大多数情况是由于大量null值或者""引起，在计算前过滤掉这些数据既可。

`sqlContext.sql("...where col is not null and col != ''") `

*报错：* 任务倾斜
*处理：* 

task倾斜原因比较多，网络io,cpu,mem都有可能造成这个节点上的任务执行缓慢，可以去看该节点的性能监控来分析原因。以前遇到过同事在spark的一台worker上跑R的任务导致该节点spark task运行缓慢。
或者可以开启spark的推测机制，开启推测机制后如果某一台机器的几个task特别慢，推测机制会将任务分配到其他机器执行，最后Spark会选取最快的作为最终结果。

    spark.speculation true
    spark.speculation.interval 100 - 检测周期，单位毫秒;
    spark.speculation.quantile 0.75 - 完成task的百分比时启动推测
    spark.speculation.multiplier 1.5 - 比其他的慢多少倍时启动推测。


# 三. 版本问题导致

1. java版本不一致

*报错：* java.lang.UnsupportedClassVersionError:  Unsupported major.minor version 52.0
*处理：* 该问题一般是spark的java版本与作业编译的java版本不一致，建议将本地java版本改为与spark一致的版本。

2. Scala版本不一致 

*报错：* Java.lang.NoSuchMethodError: scala.reflect.api.JavaUniverse.runtimeMirror(Ljava/lang/ClassLoader;)Lscala/reflect/api/JavaMirrors$JavaMirror; 
*处理：* 该报错就是本地使用的scala版本与集群的不一致，建议把本地scala版本替换为集群版本

3. 本地jar包跟hdfs远程的不一致 

*报错：* local class incompatible: stream classdesc serialVersionID = -6965587383804958479374, local class serialVersionID = -2231952633394736947 
*处理：* 因为spark使用spark.yarn.archive参数上传本地jar包到hdfs来加速job执行过程，所以新编译的spark要运行的话，或者删除这一参数，或者将新编译spark的jar包覆盖集群目录下的。 


# 四. 其他问题

*报错：* 
*处理：* 

1. 磁盘临时文件空间不足 

*报错：* java.io.IOException: No space left on device 
*处理：* 在shuffle过程中，中间文件都放在/tmp目录，当shuffle文件达到磁盘空间上限，就报错。解决方法可以增大executor个数，分担压力，如果仍不可以的话就联系平台同学配置spark-default.conf中设置spark.local.dir（默认是/tmp）为磁盘空间足够的目录即可解决。在yarn模式则配置LOCAL_DIRS。


2. 文件没有访问权限 

*报错：* Caused by: org.apache.hadoop.ipc.RemoteException(org.apache.hadoop.security.AccessControlException): Permission denied: user=dm, access=EXECUTE, inode=”/user/hadoop/.sparkStaging/application_1480755301936_1884”:hadoop:supergroup:drwx—— 
*处理：* 查看这个job是什么用户执行，要确定任务执行的权限，一般是使用其他组件调用，导致执行用户变化，导致没有文件权限。

3. LZO相关问题

*报错：* 

*  ```Caused by: java.lang.IllegalArgumentException: Compression codec com.hadoop.compression.lzo.LzoCodec not found.
     
      at org.apache.hadoop.io.compress.CompressionCodecFactory.getCodecClasses(CompressionCodecFactory.java:134)
     
      at org.apache.hadoop.io.compress.CompressionCodecFactory.<init>(CompressionCodecFactory.java:174)
     
      at org.apache.hadoop.mapred.TextInputFormat.configure(TextInputFormat.java:45)
     
      ... 66 more
     
     Caused by: java.lang.ClassNotFoundException: Class com.hadoop.compression.lzo.LzoCodec not found
     
      at org.apache.hadoop.conf.Configuration.getClassByName(Configuration.java:1680)
     
      at org.apache.hadoop.io.compress.CompressionCodecFactory.getCodecClasses(CompressionCodecFactory.java:127)
     
      ... 68 more
      ```
* ``` [ERROR] [2014-08-05 10:34:41 933] com.hadoop.compression.lzo.GPLNativeCodeLoader [main] (GPLNativeCodeLoader.java:36) Could not load native gpl library
  
  java.lang.UnsatisfiedLinkError: no gplcompression in java.library.path```


*处理：* 解决办法就是得安装好LZO，并且在HDFS、SPARK中配置好相关的包、文件等，具体步骤见

    http://find.searchhub.org/document/a128707a98fe4ec6
    https://github.com/twitter/hadoop-lzo/blob/master/README.md
    http://hsiamin.com/posts/2014/05/03/enable-lzo-compression-on-hadoop-pig-and-spark/

CDH参考：https://www.cloudera.com/documentation/enterprise/latest/topics/cm_ig_install_gpl_extras.html

参考：

    http://blog.csdn.net/xwc35047/article/details/53933265
    http://bigdata.51cto.com/art/201704/536499.htm
    http://www.cnblogs.com/Scott007/p/3889959.html