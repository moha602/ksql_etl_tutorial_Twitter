# Streaming ETL pipeline

## Acknowledgement 

This material is derived from the [Streaming ETL pipeline](https://www.entechlog.com/blog/kafka/exploring-ksqldb-superset-with-twitter/).

## What is it?

A streaming ETL pipeline, sometimes called a “streaming data pipeline”, is a set of software services that ingests events, transforms them, and loads them into destination storage systems. It’s often the case that you have data in one place and want to move it to another as soon as you receive it, but you need to make some changes to the data as you transfer it.

Maybe you need to do something simple, like transform the events to strip out any personally identifiable information. Sometimes, you may need to do something more complex, like enrich the events by joining them with data from another system. Or perhaps you want to pre-aggregate the events to reduce how much data you send to the downstream systems.

A streaming ETL pipeline enables streaming events between arbitrary sources and sinks, and it helps you make changes to the data while it’s in-flight.


## Why ksqlDB?

Gluing all of the above services together is certainly a challenge. Along with your original databases and target analytical data store, you end up managing clusters for Kafka, connectors, and your stream processors. It's challenging to operate the entire stack as one.

ksqlDB helps streamline how you write and deploy streaming data pipelines by boiling it down to just two things: storage (Kafka) and compute (ksqlDB).

![easy](img/etl-easy.png)

Using ksqlDB, you can run any Kafka Connect  connector by embedding it in ksqlDB's servers. You can transform, join, and aggregate all of your streams together by using a coherent, powerful SQL language. This gives you a slender architecture for managing the end-to-end flow of your data pipeline.


### The connectors

In the folder `./connectors` you find the Twitter connectors downloaded from [Confluent Hub](https://www.confluent.io/hub/jcustenborder/kafka-connect-twitter).


### The docker-compose file

The `docker-compose.yml` file defines the services to launch:

```yaml
---
version: '2'

services:
  
  zookeeper:
    image: confluentinc/cp-zookeeper:6.1.0
    hostname: zookeeper
    container_name: zookeeper
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  broker:
    image: confluentinc/cp-enterprise-kafka:6.1.0
    hostname: broker
    container_name: broker
    depends_on:
      - zookeeper
    ports:
      - "29092:29092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://broker:9092,PLAINTEXT_HOST://localhost:29092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1

  schema-registry:
    image: confluentinc/cp-schema-registry:6.1.0
    hostname: schema-registry
    container_name: schema-registry
    depends_on:
      - zookeeper
      - broker
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_CONNECTION_URL: 'zookeeper:2181'

  ksqldb-server:
    image: confluentinc/ksqldb-server:0.15.0
    hostname: ksqldb-server
    container_name: ksqldb-server
    depends_on:
      - broker
      - schema-registry
    ports:
      - "8088:8088"
    volumes:
      - "./connectors/:/usr/share/kafka/plugins/"
    environment:
      KSQL_LISTENERS: "http://0.0.0.0:8088"
      KSQL_BOOTSTRAP_SERVERS: "broker:9092"
      KSQL_KSQL_SCHEMA_REGISTRY_URL: "http://schema-registry:8081"
      KSQL_KSQL_LOGGING_PROCESSING_STREAM_AUTO_CREATE: "true"
      KSQL_KSQL_LOGGING_PROCESSING_TOPIC_AUTO_CREATE: "true"
      KSQL_CONNECT_GROUP_ID: "ksql-connect-cluster"
      KSQL_CONNECT_BOOTSTRAP_SERVERS: "broker:9092"
      KSQL_CONNECT_KEY_CONVERTER: "org.apache.kafka.connect.storage.StringConverter"
      KSQL_CONNECT_VALUE_CONVERTER: "io.confluent.connect.avro.AvroConverter"
      KSQL_CONNECT_KEY_CONVERTER_SCHEMA_REGISTRY_URL: "http://schema-registry:8081"
      KSQL_CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL: "http://schema-registry:8081"
      KSQL_CONNECT_VALUE_CONVERTER_SCHEMAS_ENABLE: "false"
      KSQL_CONNECT_CONFIG_STORAGE_TOPIC: "ksql-connect-configs"
      KSQL_CONNECT_OFFSET_STORAGE_TOPIC: "ksql-connect-offsets"
      KSQL_CONNECT_STATUS_STORAGE_TOPIC: "ksql-connect-statuses"
      KSQL_CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: 1
      KSQL_CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: 1
      KSQL_CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: 1
      KSQL_CONNECT_PLUGIN_PATH: "/usr/share/kafka/plugins"

  ksqldb-cli:
    image: confluentinc/ksqldb-cli:0.15.0
    container_name: ksqldb-cli
    depends_on:
      - broker
      - ksqldb-server
    entrypoint: /bin/sh
    tty: true
```

Note that the ksqlDB server image mounts the `connectors` directory, too. 

Bring up the entire stack by running:

```
docker-compose up
```

### Start the Twitter source connectors

Connect to ksqlDB's server by using its interactive CLI. Run the following command from your host:

```
docker exec -it ksqldb-server bash
```

After accessing the ksqldb-server you run this command to launch its interactive CLI

```
echo -e "\n\n⏳ Waiting for ksqlDB to be available before launching CLI\n"; while : ; do curl_status=$(curl -s -o /dev/null -w %{http_code} http://ksqldb-server:8088/info) ; echo -e $(date) " ksqlDB server listener HTTP state: " $curl_status " (waiting for 200)" ; if [ $curl_status -eq 200 ] ; then  break ; fi ; sleep 5 ; done ; ksql http://ksqldb-server:8088
```


Invoke the following command in ksqlDB, which creates a Twitter source connector and writes all of its changes to Kafka topics:



```NOTE
You should replace the OAuth parameters with the correct values of your twitter developper account
after you set it up. You can use the following link:
[Get twitter API key applicable for you by following]
(https://developer.twitter.com/en/docs/getting-started).
```

```sql
CREATE SOURCE CONNECTOR SOURCE_TWITTER_01 WITH (
    'connector.class' = 'com.github.jcustenborder.kafka.connect.twitter.TwitterSourceConnector',
    'twitter.oauth.accessToken' = 'TWITTER_ACCESSTOKEN',
    'twitter.oauth.consumerSecret' = 'TWITTER_CONSUMERSECRET',
    'twitter.oauth.consumerKey' = 'TWITTER_CONSUMERKEY',
    'twitter.oauth.accessTokenSecret' = 'TWITTER_ACCESSTOKENSECRET',
    'kafka.status.topic' = 'twitter_01',
    'process.deletes' = false,
    'filter.keywords' = 'oracle,java,mssql,mysql,devrel,apachekafka,confluentinc,ksqldb,kafkasummit,kafka connect,rmoff,tlberglund,gamussa,riferrei,nehanarkhede,jaykreps,junrao,gwenshap'
);
```

### Check status of connector using

```sql
DESCRIBE CONNECTOR SOURCE_TWITTER_01;
```


### Check status of topic using, If we had tweets coming in you should see the topic created

```sql
SHOW TOPICS;
```

### Create the ksqlDB source streams

For ksqlDB to be able to use the topics that Twitter connector created, you must declare streams over it. 

Run the following statement to create a stream over the `twitter` table:

```sql
CREATE STREAM TWEETS WITH (
          KAFKA_TOPIC='twitter_01', 
          VALUE_FORMAT='AVRO'
);
```

### Issue this command to start reading data from the beginning of the topic

```sql
SET 'auto.offset.reset'='earliest';
```

### Selecting/Filtering columns with ksqlDB queries Push Query

```sql
SELECT USER->SCREENNAME, LANG, TEXT 
FROM TWEETS 
WHERE LANG = 'en'
EMIT CHANGES;
```

### Pull Query

```
You can’t run a pull against stream, KSQL currently only supports pull queries on materialized aggregate tables. i.e. those created by a CREATE TABLE AS SELECT , FROM GROUP BY style statement.
```

### Aggregates in ksqlDB

```sql
SELECT LANG,COUNT(*) FROM TWEETS GROUP BY LANG EMIT CHANGES;
```

### Create a stream of hashtags (converted to lowercase)

```sql
CREATE STREAM STM_TWEETS_HASHTAGS AS 
SELECT ID, LCASE(EXPLODE(HASHTAGENTITIES)-> TEXT) AS HASHTAG 
FROM TWEETS;
```

### Create a table of hashtags and count each hashtags

```sql
CREATE TABLE TBL_TWITTER_HASHTAGS AS 
SELECT HASHTAG, COUNT(*) AS CT 
FROM STM_TWEETS_HASHTAGS 
GROUP BY HASHTAG;
```

### Issue Push Query on the table

```sql
SELECT HASHTAG, CT FROM TBL_TWITTER_HASHTAGS EMIT CHANGES;
```


### Issue Pull Query on the table

```sql
SELECT HASHTAG, CT FROM TBL_TWITTER_HASHTAGS WHERE HASHTAG='oracle';
```