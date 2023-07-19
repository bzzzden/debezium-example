for mysql

```
sudo docker exec -it mysqlterm sh -c /bin/bash
```

```
mysql -hmysql -P3306  -uroot -pdebezium 
```

один раз, чтобы всё удалялось без вопросов
```
SET FOREIGN_KEY_CHECKS=0;
```

for kafka connect

```
curl -H "Accept:application/json" localhost:8083/
```

```
curl -H "Accept:application/json" localhost:8083/connectors/
```


for register MySQL connector

'''

curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d '{ "name": "inventory-connector", "config": { "connector.class": "io.debezium.connector.mysql.MySqlConnector", "tasks.max": "1", "database.hostname": "mysql", "database.port": "3306", "database.user": "debezium", "database.password": "dbz", "database.server.id": "184054", "topic.prefix": "dbserver1", "database.include.list": "inventory", "schema.history.internal.kafka.bootstrap.servers": "kafka:9092", "schema.history.internal.kafka.topic": "schemahistory.inventory" } }'

'''