# CDH集群中安装了spark2.x，提交应用是找不到kafka的包
```
Exception in thread "main" java.lang.NoClassDefFoundError: org/apache/spark/streaming/kafka/KafkaUtils$
```

# 独立安装的spark2.x确实没有把kafka的包加进来，只加了hive的相关包

```
cp -R /etc/spark/conf/* conf/
cp /etc/hive/conf/hive-site.xml conf/
sed -i "s#\(.*SPARK_HOME\)=.*#\1=$(pwd)#" conf/spark-env.sh
sed -i 's/spark.master=yarn-client/spark.master=yarn/' conf/spark-defaults.conf
sed -i '/spark.yarn.jar/d' conf/spark-defaults.conf
echo "log4j.logger.org.spark_project.jetty=ERROR" >> conf/log4j.properties
```

# 解决办法

export KAFKA_CLASSPATH=$(find /opt/cloudera/parcels/KAFKA/lib/kafka/libs/kafka* -name 'kafka*.jar' -print0 | sed 's/\x0/,/g')

bin/spark-submit --jars $KAFKA_CLASSPATH --master yarn-client --num-executors 8 --executor-memory 4G --executor-cores 2 /home/develop/shaoxing/crime.jar









