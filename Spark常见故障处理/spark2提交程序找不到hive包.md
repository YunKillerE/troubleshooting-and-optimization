# 现象

spark读取Hive中的外部表报java.lang.NoSuchMethodError: org.apache.hadoop.hive.serde2.lazy.LazySim


# 原因

hive表中使用了多分隔符的类org.apache.hadoop.hive.contrib.serde2.MultiDelimitSerDe类

# 解决办法

所有机器上面将/opt/cloudera/parcels/CDH/jars/hive-contrib-1.1.0-cdh5.12.1.jar文件复制到/opt/cloudera/parcels/SPARK2/lib/spark2/jars

gm.sh allhost "cp /opt/cloudera/parcels/CDH/jars/hive-contrib-1.1.0-cdh5.12.1.jar /opt/cloudera/parcels/SPARK2/lib/spark2/jars"

