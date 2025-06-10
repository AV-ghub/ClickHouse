# Common
[Change Data Capture (CDC) with PostgreSQL and ClickHouse - Part 1](https://clickhouse.com/blog/clickhouse-postgresql-change-data-capture-cdc-part-1#final-performance)

[Change Data Capture (CDC) with PostgreSQL and ClickHouse - Part 2](https://clickhouse.com/blog/clickhouse-postgresql-change-data-capture-cdc-part-2)

Debezium’s PostgreSQL connector works with one of Debezium’s supported logical decoding [plug-ins](https://debezium.io/documentation/reference/stable/postgres-plugins.html#:~:text=Debezium%E2%80%99s%20PostgreSQL%20connector%20works%20with%20one%20of%20Debezium%E2%80%99s%20supported%20logical%20decoding%20plug%2Dins)

Debezium is a set of services for capturing changes in a database.  
Debezium aims to produce messages in a format independent of the source DBMS.   

Debezium is capable of producing a stream of insert, update and delete events from Postgres by exploiting the WAL log **via the pgoutput plugin** and a replication slot.

In order to process a stream of update and delete rows while avoiding the above usage patterns, we can use the ClickHouse table engine **ReplacingMergeTree**.

Deleted rows are only removed at merge time if the table level setting clean_deleted_rows is set to Always. By default, this value is set to Never, which means rows will never be deleted. As of ClickHouse **version 23.5**, this feature has a [known issue](https://github.com/ClickHouse/ClickHouse/issues/50346) when the value is set to Always, causing the wrong rows to be potentially removed.

For performance, **primary index is held in memory** with a level of indirection using marks to minimize size.   
The use of the **ORDER BY key for deduplication** in the ReplacingMergeTree can cause this to become long, increasing memory usage.   
If this becomes a concern, users can specify the PRIMARY KEY directly, restricting those columns loaded into memory while preserving the ORDER BY to maximize compression and enforce uniqueness.

Using this approach, the Postgres primary key columns can be omitted from the PRIMARY KEY - saving memory without impacting query performance.

# Practice
## DB creation
Both instances
```sql
create database pdkc
```
Check Kafka
```bash
/usr/bin/kafka/kafka-topics.sh --list   --bootstrap-server localhost:9092
```
Everything is ready.

## Postgres
```sql
\c pdkc

CREATE TABLE uk_price_paid                                                                                                                                                               (
   id serial,
   price INTEGER,
   date Date,
   postcode1 varchar(8),
   postcode2 varchar(3),
   type varchar(13),
   is_new SMALLINT,
   duration varchar(9),
   addr1 varchar(100),
   addr2 varchar(100),
   street varchar(60),
   locality varchar(35),
   town varchar(35),
   district varchar(40),
   county varchar(35),
   primary key(id)
);
```

```bash
wget https://datasets-documentation.s3.eu-west-3.amazonaws.com/uk-house-prices/postgres/uk_prices.sql.tar.gz
```

```bash
tar -xzvf uk_prices.sql.tar.gz
# uk_prices.sql
```

```sql
psql -h localhost -U postgres -p 5433 -d pdkc < /home/backups/uk_prices.sql
Пароль пользователя postgres: 
INSERT 0 10000
INSERT 0 10000
...
```

После загрузки

```bash
bash-5.1$ psql -h localhost -U postgres -p 5433 -d pdkc
```

Смотрим

```sql
pdkc=# \dt+
                                       Список отношений
 Схема  |      Имя      |   Тип   | Владелец |  Хранение  | Метод доступа | Размер  | Описание 
--------+---------------+---------+----------+------------+---------------+---------+----------
 public | uk_price_paid | таблица | postgres | постоянное | heap          | 3608 MB | 
(1 строка)

pdkc=# select count(*) from uk_price_paid;
  count   
----------
 27734966
(1 строка)
```

Публикуем

```sql
pdkc=# CREATE PUBLICATION pdkc_integration FOR TABLE public.uk_price_paid;
CREATE PUBLICATION
pdkc=# SELECT * FROM pg_create_logical_replication_slot('pdkc_integration_slot', 'pgoutput');
       slot_name       |     lsn     
-----------------------+-------------
 pdkc_integration_slot | B0/5DF969C8
(1 строка)
```

## ClickHouse schema
```sql
CREATE TABLE default.uk_price_paid
(
	`id` UInt64,
	`price` UInt32,
	`date` Date,
	`postcode1` LowCardinality(String),
	`postcode2` LowCardinality(String),
	`type` Enum8('other' = 0, 'terraced' = 1, 'semi-detached' = 2, 'detached' = 3, 'flat' = 4),
	`is_new` UInt8,
	`duration` Enum8('unknown' = 0, 'freehold' = 1, 'leasehold' = 2),
	`addr1` String,
	`addr2` String,
	`street` LowCardinality(String),
	`locality` LowCardinality(String),
	`town` LowCardinality(String),
	`district` LowCardinality(String),
	`county` LowCardinality(String),
	`version` UInt64,
	`deleted` UInt8
)
ENGINE = ReplacingMergeTree(version, deleted)
PRIMARY KEY (postcode1, postcode2, addr1, addr2)
ORDER BY (postcode1, postcode2, addr1, addr2, id)
```

## Configuring a CDC pipeline
### Kafka Connect framework

Installation dir
```bash
/usr/bin/kafka
```

---

### Что такое Kafka Connect?

* Kafka Connect — это фреймворк для потоковой интеграции данных с Kafka.
* Он позволяет подключать источники (source connectors) и приемники (sink connectors).
* В нашем случае нужен source connector для PostgreSQL (чтобы читать изменения из WAL и публиковать в Kafka).

---

### Как проверить, установлен ли Kafka Connect?

1. В папке с установкой Kafka — там обычно есть скрипты для запуска Connect, например:

  * `connect-standalone.sh`
  * `connect-distributed.sh`

2. Можно проверить, запущен ли процесс Kafka Connect:

* В Linux/Unix:

  ```bash
  ps aux | grep connect
  ```
* Или проверить, доступен ли REST API Kafka Connect по умолчанию на порту 8083:

  ```bash
  curl http://localhost:8083/
  ```

  Если есть ответ (JSON), Kafka Connect запущен.

---

### Как дополнительно к основным коннекторам установить Postgres Debezium Connector?

Читаем тут:  
[Debezium PostgreSQL Connector depoyment documentation](https://debezium.io/documentation/reference/stable/connectors/postgresql.html#postgresql-deployment)

Качаем отсюда:  
[PostgreSQL connector plug-in archive](https://repo1.maven.org/maven2/io/debezium/debezium-connector-postgres/3.1.2.Final/debezium-connector-postgres-3.1.2.Final-plugin.tar.gz)

```bash
wget https://repo1.maven.org/maven2/io/debezium/debezium-connector-postgres/3.1.2.Final/debezium-connector-postgres-3.1.2.Final-plugin.tar.gz
tar -xzvf debezium-connector-postgres-3.1.2.Final-plugin.tar.gz
```

Далее:  

> Сначала папку plugins нужно создать, например в /opt/kafka/

* Для standalone Kafka Connect достаточно положить jar-файл коннектора (то что распаковали из debezium-connector-postgres-3.1.2.Final-plugin.tar.gz) в папку plugins Kafka Connect (`/opt/kafka/`)
* В /etc/kafka:

```bash
sudo nano connect-standalone.properties 
sudo nano connect-distributed.properties
```
=>

```ini
plugin.path=/opt/kafka/debezium-connector-postgres
```

* `connect-standalone.properties` — основной конфиг для Kafka Connect.
* `connect-postgres-source.properties` — конфиг для конкретного источника (PostgreSQL).

---

### Где взять конфиг для PostgreSQL Source Connector?

Создать руками.

* Пример `connect-postgres-source.properties` для Debezium:

```properties
name=postgres-source
connector.class=io.debezium.connector.postgresql.PostgresConnector
tasks.max=1
database.hostname=localhost
database.port=5433
database.user=<user>
database.password=<pwd>
database.dbname=pdkc
database.server.name=pg_server
table.include.list=public.uk_price_paid
plugin.name=pgoutput
slot.name=pdkc_integration_slot
publication.name=pdkc_integration
topic.prefix=pg_server
```

### Как запустить Kafka Connect?

1. Перейди в каталог установки Kafka.

2. Запусти standalone режим (подойдет для тестов):

```bash
# bin/connect-standalone.sh config/connect-standalone.properties config/connect-postgres-source.properties
/bin/kafka/connect-standalone.sh config/connect-standalone.properties config/connect-postgres-source.properties
```

Проверки

```bash
ps aux | grep connect
```

```bash
curl http://localhost:8083/
```


---





















