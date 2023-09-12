# Пример для потоковой передачи данных из MySQL в PostgreSQL в реальном времени

![Схема](https://miro.medium.com/v2/resize:fit:4800/format:webp/1*zC9YrEaC-b-tsfEWpRd_ow.png)

## Запуск

Установить Docker и docker compose
Клонировать репозиторий

В файле docker-compose.yaml для сервисов MySQL, Zookeeper, Kafka и PostgreSQL монтируются папки операционной системы.
Если просто указать тома в файле yaml
```
[service_name]
 volumes:
       - my-datavolume:/var/lib/mysql
volumes:
  my-datavolume:
```
Docker создаст том в папке /var/lib/docker/volumes. Этот том будет существовать до тех пор, пока не остановить группу контейнеров 
```
docker-compose down -v
```
Чтобы правильно создать папки надо обратиться к документации MySQL, Zookeeper, Kafka и PostgreSQL:
для данных mysql
```
mkdir -p ./data/db/mysql
```
для zookeeper
```
mkdir -p ./data/zoo/data
mkdir -p ./data/zoo/log
```
для kafka
```
mkdir -p ./data/kafka/data
mkdir -p ./data/kafka/logs
mkdir -p ./data/kafka/config
```
для postgresql
```
mkdir -p ./data/db/postgresql/data
```

Всем этим папкам я сменил владельца и назначил права
```
sudo chown -R denis:docker data/
sudo chmod -R 775 data/
```
запуска всех контейнеров
```
docker-compose up -d
```
Отлаживаем ошибки в случае возникновения, добиваемся успешного запуска.

## Создание source и receiver(sink) коннекторов
### Source коннектор
Задеплоим Source коннектор с конфигурацией из файла debezium-mysql-source-connector-inventory.json для получения данных из таблицы MySQL в топик Kafka.
```
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d @debezium-mysql-source-connector-inventory.json
```
Корректный ответ
```
HTTP/1.1 201 Created
Date: Sat, 12 Aug 2023 20:18:09 GMT
Location: http://localhost:8083/connectors/inventory-connector
Content-Type: application/json
Content-Length: 722
Server: Jetty(9.4.51.v20230217)
```

Проверить статус
```
curl -H "Accept:application/json" localhost:8083/connectors/inventory-connector/status | jq 
```
```
{
  "name": "inventory-connector",
  "connector": {
    "state": "RUNNING",
    "worker_id": "192.168.192.7:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "192.168.192.7:8083"
    }
  ],
  "type": "source"
}
```
**jq** - предварительно установить 

Если все OK, то начнут создаваться топики в Kafka, если не появятся  - отлаживать.
Проверить какие топики созданы Kafka:
```
docker exec -it kafka bin/kafka-topics.sh --list --bootstrap-server kafka:9092
```
Ответ
```
__consumer_offsets
addresses
customers
dbserver1
geom
my_connect_configs
my_connect_offsets
my_source_connect_statuses
orders
products
products_on_hand
schema-changes.inventory
```
### База данных приемник

**База данных inventory (или любая другая) должна быть создана/существовать в PostgreSQL.**

### Receiver коннектор

Для того чтобы реплицировать изменения в базу PostgreSQL компанией Confluent создан JDBC Connector. По умолчанию он не поставляется с Kafka Connect.
Проверить можно вызвав:
```
curl -H "Accept:application/json" localhost:8083/connector-plugins/
```
В случае, если предполагается использование Kafka Connect от Confluent, потребуется самостоятельно добавить плагины необходимых коннекторов в директорию, указанную в plugin.path или задаваемую через переменную окружения CLASSPATH.

Скачать плагин Sink от Confluent поместить его в папку connector-plugins (имя произвольное) и подмонтировать в контейнер в папку с плагинами
```
[service_name]
   volumes:
      - ./connector-plugins/confluentinc-kafka-connect-jdbc-10.7.4/lib:/kafka/connect/confluentinc-kafka-connect-jdbc-10.7.4 
```
после этого перезапустить группу контейнеров
```
docker-compose down -v
docker-compose up -d
```
Теперь можно создать Reciever (sink) коннектор класса io.confluent.connect.jdbc.JdbcSinkConnector
Задеплоим  sink коннектор для записи данных из таблицы MySQL 'customers' через одноименный топик Kafka в аналогичную таблицу в базе PostgreSQL.
На каждую таблицу свой sink коннектор, но это надо проверять экспериментально. Может можно на всю схему сразу конфиг.
Конфигурация в файле jdbc-sink-postgres-customers.json
Обратить внимание на transforms в источнике и приемнике.
```
        "transforms": "unwrap",
        "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
        "transforms.unwrap.drop.tombstones": "false",
```
Debezium пишет поумолчанию в топик с именем: 
[Topic names](https://debezium.io/documentation/reference/stable/connectors/mysql.html#mysql-topic-names).

Для JDBC коннектора от Confluent надо только имя (таблицы) [Confluent Data Mapping](https://docs.confluent.io/kafka-connectors/jdbc/current/sink-connector/sink_config_options.html#data-mapping)

Задеплоим коннектор
```
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d @jdbc-sink-postgres-customers.json
```
```
HTTP/1.1 201 Created
Date: Sat, 12 Aug 2023 20:53:49 GMT
Location: http://localhost:8083/connectors/jdbc-sink-postgresql-inventory-customers
Content-Type: application/json
Content-Length: 619
Server: Jetty(9.4.51.v20230217)
```
Проверим статус
```
curl -H "Accept:application/json" localhost:8083/connectors/jdbc-sink-postgresql-inventory-customers/status | jq
```
```
{
  "name": "jdbc-sink-postgresql-inventory-customers",
  "connector": {
    "state": "RUNNING",
    "worker_id": "192.168.224.7:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "192.168.224.7:8083"
    }
  ],
  "type": "sink"
}
```

Если нет ошибок, то в PostgreSQL создастся таблица customers и в ней будут оражаться изменения из одноименной таблицы MySQL

## Полезные команды:
```
docker logs -f --tail=4 watcher | jq
```
этой командой можно в онлайн режиме смотреть содержимое топика (предварительно должен быть настроен и развернут watcher)

### Работа к Kafka Connect

Список всех **плагинов** коннекторов в системе
```
curl -H "Accept:application/json" localhost:8083/connector-plugins/
```
Список коннекторов
```
curl -X GET localhost:8083/connectors/
```

Удалить коннектор
```
curl -X DELETE localhost:8083/connectors/<connector name>
```

[Connect REST Interface](https://docs.confluent.io/platform/current/connect/references/restapi.html)



## Ссылки
https://blog.devgenius.io/change-data-capture-from-mysql-to-postgresql-using-kafka-connect-and-debezium-ae8740ef3a1d

https://debezium.io/blog/2017/09/25/streaming-to-another-database/#:~:text=curl%20-i%20-X%20POST%20-H,%22Accept%3Aapplication%2Fjson%22%20-H%20%22Content-Type%3Aapplication%2Fjson%22%20http%3A%2F%2Flocalhost%3A8083%2Fconnectors%2F%20-d%20%40source.json

https://medium.com/swlh/sync-mysql-to-postgresql-using-debezium-and-kafkaconnect-d6612489fd64

https://docs.confluent.io/kafka-connectors/jdbc/current/sink-connector/sink_config_options.html

Скачать плагин Sink от Confluent https://d1i4a15mxbxib1.cloudfront.net/api/plugins/confluentinc/kafka-connect-jdbc/versions/10.7.4/confluentinc-kafka-connect-jdbc-10.7.4.zip

https://docs.confluent.io/platform/current/connect/logging.html
