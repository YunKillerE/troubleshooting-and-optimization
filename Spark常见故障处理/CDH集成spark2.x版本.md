# 下载最新的spark

http://spark.apache.org/downloads.html

# 集成

解压，然后进入目录

    cp -R /etc/spark/conf/* conf/
    cp /etc/hive/conf/hive-site.xml conf/
    sed -i "s#\(.*SPARK_HOME\)=.*#\1=$(pwd)#" conf/spark-env.sh
    sed -i 's/spark.master=yarn-client/spark.master=yarn/' conf/spark-defaults.conf
    sed -i '/spark.yarn.jar/d' conf/spark-defaults.conf
    echo "log4j.logger.org.spark_project.jetty=ERROR" >> conf/log4j.properties

# 报错

    Caused by: java.lang.ClassNotFoundException: com.cloudera.spark.lineage.ClouderaNavigatorListener
    
解决办法，删除conf/spark-default.conf中的如下行，可能有两行

    spark.extraListeners=com.cloudera.spark.lineage.ClouderaNavigatorListener
    
然后就可以了

用spark sql时，也会出现问题，同样的方法，删除conf/spark-default.conf中的如下行

    spark.sql.queryExecutionListeners=com.cloudera.spark.lineage.ClouderaNavigatorListener

    