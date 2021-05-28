# K2: Ascendiendo a la cumbre del procesamiento real-time

![](https://upload.wikimedia.org/wikipedia/commons/1/12/K2_2006b.jpg)

## Setup


### Create Weavescope 

```bash
kubectl create clusterrolebinding "cluster-admin-$(whoami)" --clusterrole=cluster-admin --user="$(gcloud config get-value core/account)"
kubectl apply -f "https://cloud.weave.works/k8s/scope.yaml?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

```bash
WEAVE_POD=$(kubectl get pod -n weave | grep app | awk '{printf("%s", $1)}')
kubectl port-forward -n weave $WEAVE_POD 4040
```

### Create zookeeper, kafka y kafka-rest

```bash
kubectl apply -f zookeeper.yml
kubectl apply -f kafka.yml
kubectl apply -f kafka-http.yml
```

### Create ksqlDB (server & cli)

```bash
kubectl apply -f ksql-server.yml
kubectl apply -f ksql-connect.yml
```

### Create Grafana and PostgresSQL

```bash
kubectl apply -f postgresql.yml
kubectl apply -f grafana.yml
```

## Configure Postgres Tables


```sql

-- Table Definition ----------------------------------------------

CREATE TABLE sensors (
    sensor_id text PRIMARY KEY,
    name text,
    campus text,
    zone text
);

CREATE TABLE "data-agg" (
    sensor_id text,
    date timestamp without time zone,
    temperature double precision,
    humidity double precision,
    name text,
    campus text,
    zone text
);

CREATE UNIQUE INDEX sensors_data_pkey ON "data-agg"(sensor_id text_ops);
CREATE UNIQUE INDEX sensors_pkey ON sensors(sensor_id text_ops);
```

## Building data pipeline

### Configure kafka topics

```bash
kubectl exec -it kafka-0 bash
/opt/kafka/bin/kafka-topics.sh --create --partitions 10 --zookeeper zk-cs:2181 --replication-factor 3 --topic data
/opt/kafka/bin/kafka-topics.sh --create --partitions 10 --zookeeper zk-cs:2181 --replication-factor 3 --topic jdbc-sensors
/opt/kafka/bin/kafka-topics.sh --create --partitions 10 --zookeeper zk-cs:2181 --replication-factor 3 --topic data-agg
```

### Sending test data
```bash
docker run -it -e HTTP_SERVER=$HTTP_KAFKA_REST_IP:8082 -e HTTP_TOPIC=data -e HTTP_INTERVAL_MS=10 andresgomezfrr/data-simulator:3.0
```

### KSQL Kafka Connect

```sql
 CREATE SOURCE CONNECTOR "jdbc-connector-source" WITH(
    "connector.class"='io.confluent.connect.jdbc.JdbcSourceConnector',
    "connection.url"='jdbc:postgresql://postgres:5432/postgres?user=postgres&password=postgres',
    "mode"='bulk',
    "topic.prefix"='jdbc-',
    "table.whitelist"='sensors',
    "value.converter"='org.apache.kafka.connect.json.JsonConverter',
    "value.converter.schemas.enable"='false',
    "key"='sensor_id'
);
```

```sql
CREATE SINK CONNECTOR "jdbc-connector-sink" WITH(
    "connector.class"='io.confluent.connect.jdbc.JdbcSinkConnector',
    "connection.url"='jdbc:postgresql://postgres:5432/postgres?user=postgres&password=postgres',
    "table.name.format"='${topic}',
    "topics"='data-agg',
    "value.converter"='io.confluent.connect.avro.AvroConverter',
    "value.converter.schemas.enable"='true',
    "value.converter.schema.registry.url"='http://kafka-schema-registry:8089',
    "auto.create" = 'true'
);
```

### KSQL Queries

```sql

CREATE STREAM sensor_data ( id STRING, metrics MAP<String, INT> ) WITH ( KAFKA_TOPIC='data', VALUE_FORMAT='JSON', PARTITIONS='10');

CREATE TABLE sensor_metadata ( sensor_id STRING PRIMARY KEY, name STRING, campus STRING, zone STRING ) WITH ( KAFKA_TOPIC='jdbc-sensors', VALUE_FORMAT='JSON', PARTITIONS='10' );

CREATE STREAM sensor_enrich WITH ( KAFKA_TOPIC='data-agg', VALUE_FORMAT='AVRO', PARTITIONS='10' ) AS SELECT id, AS_VALUE(FROM_UNIXTIME(SENSOR_DATA.ROWTIME)) AS date, AS_VALUE(id) as sensor_id, METRICS['temperature'] AS temperature, metrics['humidity'] AS humidity, name, zone, campus FROM sensor_data LEFT JOIN sensor_metadata ON sensor_data.id = sensor_metadata.sensor_id;
``

![](https://media-exp1.licdn.com/dms/image/C4D1BAQFMnvw5k083Pg/company-background_10000/0/1571323616993?e=2159024400&v=beta&t=yaKSR3yQbtbj1h5C60It1CgAHkYMyYXlGSEf17EDBFw)
