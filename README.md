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
kubectl apply -f schema-registry.yml
kubectl apply -f ksql-server.yml
kubectl apply -f kafka-connect.yml
```

### Create Grafana and PostgresSQL

```bash
kubectl apply -f postgresql.yml
kubectl apply -f grafana.yml
```

## Configure Postgres Tables

* Connect to postgresql using psql and execute the SQL queries:

    ```bash
    kubectl exec -it $(kubectl get pod -l app=postgres -o json | jq .items[].metadata.name -r) -- psql -U postgres   
    ```

    ```sql
    CREATE TABLE sensors (
        sensor_id text PRIMARY KEY,
        name text,
        campus text,
        zone text
    );

    INSERT INTO sensors (sensor_id, name, campus, zone) VALUES('1111', 'Sensor A', 'Plaza Mayor', 'Sur');
    INSERT INTO sensors (sensor_id, name, campus, zone) VALUES('2222', 'Sensor B', 'Plaza Mayor', 'Norte');
    INSERT INTO sensors (sensor_id, name, campus, zone) VALUES('3333', 'Sensor C', 'Parque', 'Central');
    ```

## Building data pipeline

### Export SVC IP address

```bash
export HTTP_KAFKA_REST_IP=$(kubectl get svc kafka-http -o json | jq .status.loadBalancer.ingress[].ip -r)
export GRAFANA_IP=$(kubectl get svc grafana -o json | jq .status.loadBalancer.ingress[].ip -r)

echo "HTTP_KAFKA_REST_IP: $HTTP_KAFKA_REST_IP"
echo "GRAFANA_IP: http://$GRAFANA_IP:3000"
```

### Configure kafka topics

* Connect to kafka broker and execute:

```bash
kubectl exec -it kafka-0 bash
```

```bash
/opt/kafka/bin/kafka-topics.sh --create --partitions 10 --zookeeper zk-cs:2181 --replication-factor 3 --topic data
/opt/kafka/bin/kafka-topics.sh --create --partitions 10 --zookeeper zk-cs:2181 --replication-factor 3 --topic jdbc-sensors
/opt/kafka/bin/kafka-topics.sh --create --partitions 10 --zookeeper zk-cs:2181 --replication-factor 3 --topic data-agg
```

### Sending test data
```bash
docker run -it -e HTTP_SERVER=$HTTP_KAFKA_REST_IP:8082 -e HTTP_TOPIC=data -e HTTP_INTERVAL_MS=10 andresgomezfrr/data-simulator:3.0
```

### KSQL Queries

* Connect to kafkasql and execute:

```bash
kubectl exec -it $(kubectl get pod -l app=ksql-cli -o json | jq .items[].metadata.name -r) -- ksql http://ksql-server:8088
```

```sql

CREATE STREAM sensor_data ( id STRING, metrics MAP<String, INT> ) WITH ( KAFKA_TOPIC='data', VALUE_FORMAT='JSON', PARTITIONS='10');

CREATE TABLE sensor_metadata ( sensor_id STRING PRIMARY KEY, name STRING, campus STRING, zone STRING ) WITH ( KAFKA_TOPIC='jdbc-sensors', VALUE_FORMAT='JSON', PARTITIONS='10' );

CREATE STREAM sensor_enrich WITH ( KAFKA_TOPIC='data-agg', VALUE_FORMAT='AVRO', PARTITIONS='10' ) AS SELECT id, AS_VALUE(FROM_UNIXTIME(SENSOR_DATA.ROWTIME)) AS date, AS_VALUE(id) as sensor_id, METRICS['temperature'] AS temperature, metrics['humidity'] AS humidity, name, zone, campus FROM sensor_data LEFT JOIN sensor_metadata ON sensor_data.id = sensor_metadata.sensor_id;
```

### KSQL Kafka Connect

* Connect to kafkasql and execute:

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

## Import grafana dashboard
![](https://media-exp1.licdn.com/dms/image/C4D1BAQFMnvw5k083Pg/company-background_10000/0/1571323616993?e=2159024400&v=beta&t=yaKSR3yQbtbj1h5C60It1CgAHkYMyYXlGSEf17EDBFw)
