# Integraciones con HDFS
Operaciones con ficheros Parquet, HDFS, Hive, Impala, Druid y Superset. 

Todos los contenedores tienen acceso al directorio ./workspace en /workspace, y operan en la red mynet. 
```
docker network create mynet
```

## Clúster HDFS
Lanzamiento de un contenedor con hadoop core integrado. HDFS estará disponible en hdfs://namenode:9000. La webUI de HDFS está en http://localhost:50070

```
docker network create mynet
docker run --rm -itd -v ./workspace:/workspace --name hadoop -p 8888:8888 -p 50070:50070 -p 8088:8088 --network mynet --net-alias namenode jdvelasq/hadoop:2.10.1
```

Desde un shell preparamos unos directorios en HDFS. No hace falta especificar namenode:puerto, porque este contenedor los toma por omisión. 

```
docker exec -it hadoop bash

hadoop fs -mkdir /user/hive
hadoop fs -chown hive /user/hive

hadoop fs -mkdir /user/impala
hadoop fs -chown impala /user/impala

hadoop fs -mkdir /user/druid
hadoop fs -chown druid /user/druid
```

## Impala
Lanzamos un contenedor con Impala. Accederá a HDFS como se ha indicado antes.

```
docker run --rm -d --name impala --network mynet \
  -p 21000:21000 -p 21050:21050 -p 25000:25000 -p 25010:25010 -p 25020:25020 \
  --memory=4096m -e JAVA_HOME="/usr" -v ./workspace:/workspace apache/kudu:impala-latest impala
```

Desde un shell preparamos los ficheros (los movemos de /workspace a HDFS). Desde el impala-shell, operamos sobre una tabla externa en HDFS, formato Parquet. El esquema lo infiere del fichero ("like PARQUET...").

```
docker exec -it impala bash

hadoop fs -mkdir hdfs://namenode:9000/user/impala/vuelos/
hadoop fs -put /workspace/flights-1m.parquet hdfs://namenode:9000/user/impala/vuelos/flights-1m.parquet

impala-shell

CREATE external TABLE Flights like PARQUET 'hdfs://namenode:9000/user/impala/vuelos/flights-1m.parquet' STORED AS PARQUET LOCATION 'hdfs://namenode:9000/user/impala/vuelos';
select * from flights limit 10;
```

## Hive
Repetimos con Hive. En este caso el esquema de la tabla hay que especificarlo. 

```
docker run --rm -d -v ./workspace:/workspace -p 10000:10000 -p 10002:10002 --env SERVICE_NAME=hiveserver2 --network mynet --name hive apache/hive:3.1.3

docker exec -it hive bash 

hadoop fs -mkdir hdfs://namenode:9000/user/hive/vuelos/
hadoop fs -put /workspace/flights-1m.parquet hdfs://namenode:9000/user/hive/vuelos/flights-1m.parquet

beeline -u 'jdbc:hive2://localhost:10000/'

CREATE external TABLE Flights (fl_date date, dep_delay  smallint, arr_delay  smallint, air_time   smallint, distance  smallint, dep_time   float,  arr_time float) STORED AS PARQUET LOCATION 'hdfs://namenode:9000/user/hive/vuelos';

CREATE TABLE parquet_test ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.avro.AvroSerDe' STORED AS AVRO TBLPROPERTIES ('avro.schema.url'='myHost/myAvroSchema.avsc'); 

CREATE EXTERNAL TABLE parquet_test LIKE avro_test STORED AS PARQUET LOCATION 'hdfs://myParquetFilesPath';

CREATE EXTERNAL TABLE parquet_test LIKE avro_test STORED AS PARQUET LOCATION 'hdfs://myParquetFilesPath';
```

## Druid
Lanzamos un contenedor Druid, reemplazando el fichero "common.runtime.properties", de tal forma que se cargue la extensión de lectura de ficheros Parquet. 

```
docker run --rm -itd \
    --name druid \
    --network mynet \
    -p 9999:9999 \
    -v ./workspace:/workspace \
    -v ./druid.properties:/opt/druid/conf/druid/single-server/nano-quickstart/_common/common.runtime.properties \
    jdvelasq/druid:0.22.1
```
El proceso de ingesta es el habitual. De forma local el fichero está en /workspace. Si se desea, se puede copiar a HDFS tal como se ha hecho con HIVE e Impala. 


## superset
```
docker run --rm -d -p 8080:8088 --name superset --network mynet acpmialj/ipmd:ssuperset
```

webUI en http://localhost:8080 (admin/admin)
Conexión a Impala como "Other", "impala://impala:21050/default"

Conexión a Hive como "Hive", "hive://hive@hive:10000/default"

Conexión a Druid como "Druid", "druid://druid/druid/v2/sql"