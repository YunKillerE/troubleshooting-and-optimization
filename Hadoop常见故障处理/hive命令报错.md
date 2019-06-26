# Caused by: java.lang.ClassNotFoundException: org.apache.hadoop.mapreduce.Mapper

CDH更新配置后独立部署的hive无法运行，在hadoop-env.sh中

    export HADOOP_MAPRED_HOME=$( ([[ ! '{{CDH_MR2_HOME}}' =~ CDH_MR2_HOME ]] && echo {{CDH_MR2_HOME}} ) || echo ${CDH_MR2_HOME:-/usr/lib/hadoop-mapreduce/}  )

这里会去找CDH_MR2_HOME这个变量

根本原因是找不到MapReduce的相关包，把CDH_MR2_HOME这个环境变量加上就可以了

解决办法：

    export HADOOP_CONF_DIR=/alidata4/hadoopclient/etc/hive-conf
    export YARN_CONF_DIR=/alidata4/hadoopclient/etc/hive-conf
    export HIVE_CONF_DIR=/alidata4/hadoopclient/etc/hive-conf
    export HADOOP_HOME=/alidata4/hadoopclient/CDH-5.14.2-1.cdh5.14.2.p0.3
    export HIVE_HOME=/alidata4/hadoopclient/CDH-5.14.2-1.cdh5.14.2.p0.3/lib/hive
    export YARN_CONF_DIR=/alidata4/hadoopclient/etc/hive-conf
    export CDH_MR2_HOME=$HADOOP_HOME/lib/hadoop-mapreduce
    export SPARK_HOME=/alidata4/hadoopclient/SPARK2-2.3.0.cloudera2-1.cdh5.13.3.p0.316101/lib/spark2
    export SPARK_CONF_DIR=/etc/spark2/conf
