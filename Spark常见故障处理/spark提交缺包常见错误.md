# sparksql 提交到yarn出现 java.lang.NoClassDefFoundError: Lorg/apache/hadoop/hive/ql/plan/TableDesc

环境：CDH5.8.4 正常请应该是不会出现这个问题，可能安装的时候没有选择hive依赖关系

解决办法：加入 --conf "spark.executor.extraClassPath=/opt/cloudera/parcels/CDH/lib/hive/lib/*"

比如：
```
spark-submit --master yarn-client --num-executors 10 --executor-memory 4G --executor-cores 2 --conf spark.rpc.askTimeout=600s --conf spark.network.timeout=600s --conf "spark.executor.extraClassPath=/opt/cloudera/parcels/CDH/lib/hive/lib/*" /home/develop/shaoxing/shaoxing_police.jar 004 20170605
```









