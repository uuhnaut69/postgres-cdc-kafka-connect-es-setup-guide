# postgres-cdc-kafka-connect-es-setup-guide
A tutorial setup ETL Postgres CDC -> Kafka Connect -> ES 

# Topology
```

                   +-------------+
                   |             |
                   |   Postgres  |
                   |             |
                   +------+------+
                          |
                          |
                          |
          +---------------v------------------+
          |                                  |
          |           Kafka Connect          |
          |    (Debezium, ES connectors)     |
          |                                  |
          +---------------+------------------+
                          |
                          |
                          |
                          |
                  +-------v--------+
                  |                |
                  | Elasticsearch  |
                  |                |
                  +----------------+
```
# Installation Guide

**Installing the wal2json plugin**

[wal2json](https://github.com/eulerto/wal2json/blob/master/README.md): To encode changes in JSON format

- Go to lib folder of Postgres app: ``` /usr/pgsql-9.6/lib/ ``` (The url maybe difference depend on your installation by homebrew or using Postgres App)
- Set the export path as shown below: ``` export PATH="$PATH:/usr/pgsql-9.6/bin" ```
- Enter the wal2json installation commands.

  ```
  git clone https://github.com/eulerto/wal2json -b master --single-branch \
  && cd wal2json \
  && git checkout d2b7fef021c46e0d429f2c1768de361069e58696 \
  && make && make install \
  && cd .. \
  && rm -rf wal2json
  ```

**Or Using Docker Images:** [Here](https://hub.docker.com/r/debezium/postgres)  
  
**Enable Replication on the PostgreSQL server**

- Open ``` postgresql.conf ``` file. Find and edit as below configuration.
  ```
  # LOGGING
  log_min_error_statement = fatal
  # CONNECTION
  listen_addresses = '*'
  # MODULES
  shared_preload_libraries = 'wal2json'
  # REPLICATION
  wal_level = logical             # minimal, archive, hot_standby, or logical (change requires restart)
  max_wal_senders = 1             # max number of walsender processes (change requires restart)
  #wal_keep_segments = 4          # in logfile segments, 16MB each; 0 disables
  #wal_sender_timeout = 60s       # in milliseconds; 0 disables
  max_replication_slots = 1       # max number of replication slots (change requires restart)
  ```
  
 **Install Kafka Connect (Confluent Platform)**
 
 - Installation Confluent tutorial can be found [here](https://docs.confluent.io/current/installation/index.html#installation-overview)
 
 **Install the connector**
 
 - Debezium Connector Postgresql: 
 ``` 
 confluent-hub install debezium/debezium-connector-postgresql:latest 
 ```
 
 - Elasticsearch Sink Connector:
 ``` 
 confluent-hub install confluentinc/kafka-connect-elasticsearch:latest 
 ``` 
 ( Must use ES version >= 4.0 unless when delete operation on postgres, record being deleted won't be deleted on ES)
 
  
# Connect & Run

- Register Postgres CDC connector 

```
[POST] http://localhost:8083/connectors/

Body request:

{
    "name": "jpwork-connector",
    "config": {
        "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
        "tasks.max": "1",
        "plugin.name": "pgoutput",
        "database.hostname": "127.0.0.1",
        "database.port": "5432",
        "database.user": "postgres",
        "database.password": "postgres",
        "database.dbname" : "postgres",
        "database.server.name": "postgres",
        "schema.whitelist": "public"
    }
}
```

- Register Elasticsearch Sink Connector: [Issue troubleshoot about body request ES](https://github.com/confluentinc/kafka-connect-elasticsearch/issues/230) 
```
[POST] http://localhost:8083/connectors/

Body request:

{
    "name": "elastic-sink",
    "config": {
        "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
        "tasks.max": "1",
        "topics": "postgres.public.category",
        "connection.url": "http://localhost:9200",
        "transforms": "unwrap,key",
        "transforms.unwrap.type": "io.debezium.transforms.UnwrapFromEnvelope",
        "transforms.unwrap.drop.tombstones": "false",
        "transforms.unwrap.drop.deletes": "false",
        "transforms.key.type": "org.apache.kafka.connect.transforms.ExtractField$Key",
        "transforms.key.field": "id",
        "key.ignore": "false",
        "type.name": "category",
        "behavior.on.null.values": "delete"
    }
}
```
