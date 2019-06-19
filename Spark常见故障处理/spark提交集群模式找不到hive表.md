# spark-submit应用到集群模式

/opt/spark2/bin/spark-submit --jars kudu-spark2_2.11-1.3.0.jar --master yarn --deploy-mode cluster --num-executors 30 --executor-memory 8G --executor-cores 2 --conf spark.rpc.askTimeout=6000s --conf spark.network.timeout=6000s --class import.hiveToKudu kudu.jar call_center tpcds_bin_partitioned_parquet_1000 kudu_spark_tpcds_1000

# 报错

User class threw exception: org.apache.spark.sql.catalyst.analysis.NoSuchTableException: Table or view 'call_center' not found in database 'tpcds_bin_partitioned_parquet_1000';

表明明存在为什么找不到呢？

# 解决办法


/opt/spark2/bin/spark-submit --jars kudu-spark2_2.11-1.3.0.jar --files /etc/hive/conf/hive-site.xml --master yarn --deploy-mode cluster --num-executors 30 --executor-memory 8G --executor-cores 2 --conf spark.rpc.askTimeout=6000s --conf spark.network.timeout=6000s --class import.hiveToKudu kudu.jar call_center tpcds_bin_partitioned_parquet_1000 kudu_spark_tpcds_1000


参考：https://community.hortonworks.com/questions/5798/spark-hive-tables-not-found-when-running-in-yarn-c.html


如果缺少jar包也可以采用类似的方法解决，比如

> 报错：java.lang.NoClassDefFoundError: org/apache/hadoop/hive/conf/HiveVariableSource

也可以采取上面的方法解决，或者也有可能是真的找不到jar包，采用下面办法解决：

export HIVE_CLASSPATH=$(find /opt/cloudera/parcels/CDH/lib/hive/lib/ -name 'hive*.jar' -print0 | sed 's/\x0/,/g')

spark-submit --jars $HIVE_CLASSPATH --master yarn-client --num-executors 8 --executor-memory 4G --executor-cores 2 /home/develop/shaoxing/crime.jar


# 找不到hive表情况可能是spark-defaults.conf中缺失配置

```
spark.sql.hive.metastore.jars=${env:HADOOP_COMMON_HOME}/../hive/lib/*:${env:HADOOP_COMMON_HOME}/client/*
spark.sql.hive.metastore.version=1.1.0
spark.sql.catalogImplementation=hive
```
