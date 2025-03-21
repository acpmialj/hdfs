# hdfs
Playing with HDFS

docker network create mynet
docker run --rm -itd -v ./workspace:/workspace --name hadoop -p 8888:8888 -p 50070:50070 -p 8088:8088 --network mynet --net-alias namenode jdvelasq/hadoop:2.10.1

desde un shell

hadoop fs -mkdir /user/hive
hadoop fs -chown hive /user/hive

hadoop fs -mkdir /user/impala
hadoop fs -chown impala /user/impala

hadoop fs -mkdir /user/druid
hadoop fs -chown hive /user/druid


docker run --rm -d -v ./workspace:/workspace -p 10000:10000 -p 10002:10002 --env SERVICE_NAME=hiveserver2 --network mynet --name hive4 apache/hive:3.1.3

docker exec -it hive4 bash 

hadoop fs -mkdir hdfs://namenode:9000/user/hive/vuelos/
hadoop fs -put /workspace/flights-1m.parquet hdfs://namenode:9000/user/hive/vuelos/flights-1m.parquet

beeline -u 'jdbc:hive2://localhost:10000/'

!sh hadoop fs -ls hdfs://namenode:9000/user/hive