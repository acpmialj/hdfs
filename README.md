# hdfs
Playing with HDFS

```
docker network create mynet
docker run --rm -itd -v ./workspace:/workspace --name hadoop -p 8888:8888 -p 50070:50070 -p 8088:8088 --network mynet --net-alias namenode jdvelasq/hadoop:2.10.1
```

desde un shell

```
docker exec -it hadoop bash

hadoop fs -mkdir /user/hive
hadoop fs -chown hive /user/hive

hadoop fs -mkdir /user/impala
hadoop fs -chown impala /user/impala

hadoop fs -mkdir /user/druid
hadoop fs -chown hive /user/druid
```

## Impala
```
docker run --rm -d --name impala --network mynet \
  -p 21000:21000 -p 21050:21050 -p 25000:25000 -p 25010:25010 -p 25020:25020 \
  --memory=4096m -e JAVA_HOME="/usr" -v ./workspace:/workspace apache/kudu:impala-latest impala
```

Desde un shell

```
docker exec -it impala bash

hadoop fs -mkdir hdfs://namenode:9000/user/impala/vuelos/
hadoop fs -put /workspace/flights-1m.parquet hdfs://namenode:9000/user/impala/vuelos/flights-1m.parquet

impala-shell

CREATE external TABLE Flights like PARQUET 'hdfs://namenode:9000/user/impala/vuelos/flights-1m.parquet' STORED AS PARQUET LOCATION 'hdfs://namenode:9000/user/impala/vuelos';
select * from flights limit 10;
```

## Hive

docker run --rm -d -v ./workspace:/workspace -p 10000:10000 -p 10002:10002 --env SERVICE_NAME=hiveserver2 --network mynet --name hive apache/hive:3.1.3

docker exec -it hive bash 

hadoop fs -mkdir hdfs://namenode:9000/user/hive/vuelos/
hadoop fs -put /workspace/flights-1m.parquet hdfs://namenode:9000/user/hive/vuelos/flights-1m.parquet

beeline -u 'jdbc:hive2://localhost:10000/'

CREATE external TABLE Flights (fl_date date, dep_delay  smallint, arr_delay  smallint, air_time   smallint, distance  smallint, dep_time   float,  arr_time float) STORED AS PARQUET LOCATION 'hdfs://namenode:9000/user/hive/vuelos';


## Druid

docker run --rm -itd \
    --name druid \
    --network mynet \
    -p 9999:9999 \
    -v ./workspace:/workspace \
    jdvelasq/druid:0.22.1

No acepta Parquet, hace falta una extensión. Propuesta: en Hive o Impala, hacer una copia en CSV (o JSON)


## superset
docker run --rm -d -p 8080:8088 --name superset --network mynet acpmialj/ipmd:ssuperset

webUI en http://localhost:8080 (admin/admin)
Conexión a Impala como "Other", "impala://impala:21050/default"

Conexión a Hive como "Hive", "hive://hive@hive:10000/default"