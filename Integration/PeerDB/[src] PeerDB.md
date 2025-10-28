Да, технология ClickPipes от ClickHouse предназначена для интеграции данных из PostgreSQL в ClickHouse, но на данный момент она доступна только в облачной версии ClickHouse Cloud и находится на стадии публичной бета-версии. Это означает, что ClickPipes не поддерживает локальные инстансы ClickHouse или PostgreSQL. ([clickhouse.com][1])

**Альтернативы для локальной интеграции:**

Если вы хотите настроить интеграцию между локальными инстансами PostgreSQL и ClickHouse, вы можете воспользоваться следующими методами:

1. **Использование таблицы ClickHouse с движком PostgreSQL:**

   ClickHouse предоставляет возможность создания таблиц с движком `PostgreSQL`, которые позволяют читать данные непосредственно из таблиц PostgreSQL. Пример создания такой таблицы:

   ```sql
   CREATE TABLE clickhouse_db.table_name
   ENGINE = PostgreSQL('localhost:5432', 'postgres_db', 'table_name', 'username', 'password');
   ```



Этот метод позволяет выполнять запросы к данным PostgreSQL непосредственно из ClickHouse.&#x20;

2. **Использование MaterializedPostgreSQL (экспериментальная функция):**

   ClickHouse также предоставляет экспериментальный движок `MaterializedPostgreSQL`, который позволяет синхронизировать данные между PostgreSQL и ClickHouse. Однако эта функция является экспериментальной и может быть нестабильной.&#x20;

3. **Использование PeerDB (открытый исходный код):**

   PeerDB — это инструмент с открытым исходным кодом, предназначенный для репликации данных из PostgreSQL в ClickHouse. Он может быть использован в локальных окружениях и предоставляет функциональность, аналогичную ClickPipes.&#x20;

**Пример настройки с использованием Docker Compose:**

Для быстрого развертывания локальной среды с PostgreSQL и ClickHouse вы можете использовать `docker-compose`. Пример конфигурации:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:latest
    environment:
      POSTGRES_PASSWORD: your_password
    ports:
      - "5432:5432"

  clickhouse:
    image: yandex/clickhouse-server
    ports:
      - "8123:8123"
      - "9000:9000"
```



После запуска контейнеров вы можете подключиться к PostgreSQL и создать таблицу:

```sql
CREATE DATABASE test_db;
CREATE TABLE test_table (id SERIAL PRIMARY KEY, name VARCHAR(100));
INSERT INTO test_table (name) VALUES ('Alice'), ('Bob');
```



Затем в ClickHouse создайте таблицу с движком PostgreSQL:([clickhouse.com][2])

```sql
CREATE TABLE clickhouse_db.test_table
ENGINE = PostgreSQL('localhost:5432', 'test_db', 'test_table', 'postgres', 'your_password');
```



Теперь вы можете выполнять запросы к данным PostgreSQL из ClickHouse.

**Рекомендации:**

* Для стабильной и поддерживаемой интеграции рекомендуется использовать таблицы ClickHouse с движком PostgreSQL.
* Если вы хотите настроить репликацию данных, рассмотрите использование PeerDB.
* Будьте осторожны при использовании экспериментальных функций, таких как `MaterializedPostgreSQL`, в продакшн-средах.([clickhouse.com][1], [github.com][3])

[1]: https://clickhouse.com/docs/ru/integrations/clickpipes/postgres?utm_source=chatgpt.com "Погружение данных из Postgres в ClickHouse (с использованием CDC) | ClickHouse Docs"
[2]: https://clickhouse.com/docs/ru/integrations/clickpipes/postgres/faq?utm_source=chatgpt.com "Часто задаваемые вопросы о ClickPipes для Postgres | ClickHouse Docs"
[3]: https://github.com/ClickHouse/clickhouse-docs/blob/main/docs/en/integrations/data-ingestion/dbms/postgresql/connecting-to-postgresql.md?utm_source=chatgpt.com "clickhouse-docs/docs/en/integrations/data-ingestion/dbms/postgresql/connecting-to-postgresql.md at main · ClickHouse/clickhouse-docs · GitHub"

PeerDB — это инструмент для репликации данных из PostgreSQL в ClickHouse, ориентированный на высокую производительность и простоту настройки. Он поддерживает два основных режима интеграции: CDC (Change Data Capture) и Query Replication.([threequants.io][1])

---

### 🔄 Как работает PeerDB?

PeerDB использует PostgreSQL Logical Replication Slots для отслеживания изменений в базе данных и их передачи в ClickHouse. Процесс репликации включает два этапа:([clickhouse.com][2])

1. **Начальная загрузка (Initial Load):** PeerDB выполняет параллельное считывание данных из PostgreSQL и их загрузку в ClickHouse, что значительно ускоряет процесс по сравнению с традиционными методами. ([blog.peerdb.io][3])

2. **Репликация изменений (CDC):** После начальной загрузки PeerDB продолжает отслеживать изменения в PostgreSQL через репликационный слот и передавать их в ClickHouse с минимальной задержкой. ([blog.peerdb.io][3])

PeerDB автоматически создает таблицы в ClickHouse с использованием движка `ReplacingMergeTree`, что позволяет эффективно обрабатывать обновления и удаления данных. ([blog.peerdb.io][4])

---

### ⚡ Сравнение с Debezium + Kafka

| Характеристика               | PeerDB                                    | Debezium + Kafka                         |                                                 |
| ---------------------------- | ----------------------------------------- | ---------------------------------------- | ----------------------------------------------- |
| **Реальное время (latency)** | \~10 секунд                               | \~10–30 секунд                           |                                                 |
| **Сложность настройки**      | Высокая (одна точка настройки)            | Высокая (множество компонентов)          |                                                 |
| **Масштабируемость**         | Высокая                                   | Высокая (при правильной настройке)       |                                                 |
| **Надежность**               | Высокая (с механизмами повторных попыток) | Зависит от конфигурации и инфраструктуры |                                                 |
| **Подходит для продакшн**    | Да                                        | Да                                       | ([blog.peerdb.io][3], [benjaminwootton.com][5]) |

---

### 🏗️ Настройка PeerDB для локальной интеграции

Для настройки PeerDB в локальной среде с PostgreSQL и ClickHouse выполните следующие шаги:

1. **Установите PeerDB:** Следуйте инструкциям на официальном сайте PeerDB для установки и настройки.

2. **Настройте PostgreSQL:**

   * Создайте публикацию для логической репликации:

     ```sql
     CREATE PUBLICATION peerdb_publication FOR ALL TABLES;
     ```
   * Создайте репликационный слот:

     ```sql
     SELECT pg_create_logical_replication_slot('peerdb_slot', 'pgoutput');
     ```

3. **Настройте ClickHouse:**

   * Создайте базу данных и таблицы, соответствующие структуре данных в PostgreSQL.

4. **Настройте PeerDB:**

   * Создайте зеркала (mirrors) для синхронизации данных между PostgreSQL и ClickHouse, указав параметры подключения и настройки репликации. ([docs.peerdb.io][6])

---

### ✅ Рекомендации для продакшн-среды

* **Используйте PeerDB:** Для интеграции PostgreSQL и ClickHouse в реальном времени PeerDB предлагает простоту настройки и высокую производительность.

* **Настройте мониторинг:** Регулярно отслеживайте состояние репликации и производительность системы.

* **Тестируйте нагрузку:** Проведите тестирование под нагрузкой, чтобы убедиться в способности системы обрабатывать требуемые объемы данных.

* **Резервное копирование:** Настройте регулярное резервное копирование данных для предотвращения потери информации.

[1]: https://threequants.io/insights/clickhouse-peerdb-cdc/?utm_source=chatgpt.com "Integrating Postgres And ClickHouse Using PeerDB Open Source | 3Quants"
[2]: https://clickhouse.com/blog/enhancing-postgres-to-clickhouse-replication-using-peerdb?utm_source=chatgpt.com "Enhancing Postgres to ClickHouse replication using PeerDB"
[3]: https://blog.peerdb.io/postgres-to-clickhouse-real-time-replication-using-peerdb?utm_source=chatgpt.com "Postgres to ClickHouse Real time Replication using PeerDB"
[4]: https://blog.peerdb.io/postgres-to-clickhouse-data-modeling-tips?utm_source=chatgpt.com "Postgres to ClickHouse: Data Modeling Tips"
[5]: https://benjaminwootton.com/insights/clickhouse-peerdb-demo/?utm_source=chatgpt.com "Reliably Replicating Data Between PostgreSQL and ClickHouse Part 3 - Demo | BenjaminWootton.com"
[6]: https://docs.peerdb.io/mirror/cdc-pg-clickhouse?utm_source=chatgpt.com "CDC Setup from Postgres to ClickHouse - PeerDB Docs: Setup your ETL in minutes with SQL."

На данный момент PeerDB официально поддерживает развертывание только через Docker Compose, и в документации нет описания установки без использования Docker. ([docs.peerdb.io][1])

Однако, если вы хотите установить PeerDB на AlmaLinux без использования Docker, вам потребуется выполнить следующие шаги:

---

### ✅ Шаги для установки PeerDB без Docker на AlmaLinux

1. **Установите необходимые зависимости:**

   PeerDB требует наличия Node.js и npm. Установите их с помощью следующих команд:

   ```bash
   sudo dnf install -y nodejs npm
   ```

2. **Склонируйте репозиторий PeerDB:**

   Скачайте исходный код PeerDB с GitHub:

   ```bash
   git clone --recursive https://github.com/PeerDB-io/peerdb.git
   cd peerdb
   ```

3. **Установите зависимости:**

   В каталоге проекта выполните команду для установки всех необходимых пакетов:

   ```bash
   npm install
   ```

4. **Настройте базу данных:**

   PeerDB использует PostgreSQL для хранения данных. Убедитесь, что у вас установлен PostgreSQL, и создайте базу данных для PeerDB:

   ```bash
   sudo dnf install -y postgresql-server
   sudo postgresql-setup --initdb
   sudo systemctl enable --now postgresql
   sudo -u postgres psql
   CREATE DATABASE peerdb;
   CREATE USER peerdb_user WITH PASSWORD 'your_password';
   GRANT ALL PRIVILEGES ON DATABASE peerdb TO peerdb_user;
   \q
   ```

5. **Настройте конфигурацию PeerDB:**

   Создайте файл конфигурации для подключения к вашей базе данных PostgreSQL. Например, создайте файл `config.json` со следующим содержимым:

   ```json
   {
     "db": {
       "host": "localhost",
       "port": 5432,
       "user": "peerdb_user",
       "password": "your_password",
       "database": "peerdb"
     },
     "port": 3000
   }
   ```

6. **Запустите PeerDB:**

   В каталоге проекта выполните команду для запуска PeerDB:

   ```bash
   npm start
   ```

   Теперь PeerDB будет доступен по адресу [http://localhost:3000](http://localhost:3000).

---

### ⚠️ Важные замечания

* Этот процесс является неофициальным и может потребовать дополнительных настроек в зависимости от вашей среды.
* PeerDB активно развивается, и интерфейс или функциональность могут изменяться. Рекомендуется регулярно проверять [официальную документацию](https://docs.peerdb.io/) для получения актуальной информации.

[1]: https://docs.peerdb.io/quickstart/quickstart?utm_source=chatgpt.com "Quickstart Guide - PeerDB Docs: Setup your ETL in minutes with SQL."

Да, Docker доступен для установки на AlmaLinux 9, включая инструменты Docker Compose. Процесс установки достаточно прямолинеен и не требует использования Docker для самого себя.

---

### ✅ Установка Docker и Docker Compose на AlmaLinux 9

1. **Обновите систему:**

   ```bash
   sudo dnf update -y
   ```



2. **Добавьте официальный репозиторий Docker:**

   ```bash
   sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
   ```



3. **Установите Docker и Docker Compose:**

   ```bash
   sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
   ```



4. **Запустите и включите сервис Docker:**

   ```bash
   sudo systemctl start docker
   sudo systemctl enable docker
   ```



5. **Добавьте вашего пользователя в группу Docker:**

   ```bash
   sudo usermod -aG docker $USER
   newgrp docker
   ```



6. **Проверьте установку:**

   ```bash
   docker --version
   docker compose version
   ```



7. **Проверьте работу Docker:**

   ```bash
   docker run hello-world
   ```



---

### 🌐 Доступность Docker в России

Docker доступен для скачивания и установки в России. Официальный репозиторий Docker для CentOS, который используется для AlmaLinux, доступен по адресу:

```
https://download.docker.com/linux/centos/docker-ce.repo
```

Чтобы использовать **логическую репликацию** в PostgreSQL (например, для CDC с PeerDB, Debezium и пр.), нужно внести изменения в конфигурацию PostgreSQL и перезапустить сервер. Вот пошаговая инструкция:

---

## ✅ 1. Включите логическую репликацию

### Откройте файл конфигурации PostgreSQL

Файл обычно находится по пути:

```
/var/lib/pgsql/data/postgresql.conf
```

или

```
/etc/postgresql/14/main/postgresql.conf  # для Debian/Ubuntu
```

### Измените или добавьте следующие параметры:

```conf
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
```

Также желательно установить:

```conf
wal_keep_size = 64MB
```

> 🔎 `wal_level = logical` — ключевой параметр для логической репликации.
> 🔎 `max_replication_slots` — сколько репликационных слотов можно создать одновременно.
> 🔎 `max_wal_senders` — сколько процессов могут отправлять WAL-логи.

---

## ✅ 2. Разрешите доступ в `pg_hba.conf`

Откройте файл:

```
/var/lib/pgsql/data/pg_hba.conf
```

Добавьте строку (в зависимости от вашего IP и пользователя):

```conf
host    replication     your_user       192.168.0.0/24         md5
```

Или, если вы работаете локально:

```conf
host    replication     all             127.0.0.1/32           md5
```

---

## ✅ 3. Перезапустите PostgreSQL

```bash
sudo systemctl restart postgresql
```

---

## ✅ 4. Создайте пользователя с правами репликации

Выполните в `psql` под пользователем `postgres`:

```sql
CREATE ROLE replication_user WITH REPLICATION LOGIN PASSWORD 'your_password';
```

---

## ✅ 5. Проверьте настройки

Подключитесь к PostgreSQL и проверьте:

```sql
SHOW wal_level;
SHOW max_replication_slots;
```

Также можно проверить текущие слоты:

```sql
SELECT * FROM pg_replication_slots;
```

---

Если вы планируете использовать это с **PeerDB**, то следующий шаг — создание **публикации**:

```sql
CREATE PUBLICATION peerdb_publication FOR ALL TABLES;
```

И PeerDB сможет подключаться, используя созданного пользователя и репликационный слот.

---

Нет, **`wal_level = replica` недостаточен для работы PeerDB или любой другой системы логической репликации** (например, Debezium, pgoutput, pglogical и т. п.).

---

## ❌ `replica` ≠ `logical`

* `replica` — используется для **физической репликации** (например, streaming replication).
* `logical` — необходим для **логической репликации**, которую используют CDC-инструменты (PeerDB, Debezium, etc).

---

## ✅ Для PeerDB обязательно:

```sql
SHOW wal_level;
```

должен возвращать:

```
logical
```

---

## 🔧 Как исправить

1. Открой файл `postgresql.conf`:

```bash
sudo nano /var/lib/pgsql/data/postgresql.conf
```

2. Найди строку:

```conf
wal_level = replica
```

и замени на:

```conf
wal_level = logical
```

3. Перезапусти PostgreSQL:

```bash
sudo systemctl restart postgresql
```

4. Проверь снова:

```sql
SHOW wal_level;
```

---

После этого можно будет создавать **репликационные слоты**, **публикации** и подключать PeerDB без ошибок.


Отличный вопрос — и да, логи PeerDB Mirror действительно не всегда очевидны при интерпретации. Давайте разберёмся по шагам, что именно они могут означать в контексте **однократной репликации** таблицы.

---

### 🔍 Что видно из логов:

```
08:38:44 - Replication slot and publication setup complete
08:38:46 - obtained partitions for table cloud_storage.folderfile
08:39:16 - replicated all rows to destination for table cloud_storage.folderfile
08:39:57 - replicated all rows to destination for table cloud_storage.folderfile
08:39:57 - replicated all rows to destination for table cloud_storage.folderfile
08:39:57 - replicated all rows to destination for table cloud_storage.folderfile
```

---

## 📌 Ключевые события:

1. **08:38:44** — началась настройка репликации, созданы слоты и публикации.
2. **08:38:46** — получены партиции (разделы) таблицы. Это обычно происходит быстро.
3. **08:39:16** — завершена **первая полная передача всех строк** таблицы.
4. **08:39:57** — повторяются сообщения `replicated all rows...` — это **не повторная синхронизация**, а повторный лог о том, что "все строки на месте". Это может быть связано:

   * с проверками целостности/валидности (например, checksum, контроль завершённости);
   * с работой нескольких потоков или повторной проверкой состояния (например, если были перезапросы или таймауты подтверждений).

---

## 🕓 Сколько длилась фактическая синхронизация?

**Фактическая полная синхронизация таблицы** завершилась на момент:

> `08:39:16 - replicated all rows to destination`

Значит, **время репликации составляет:**
📊 **08:38:44 — 08:39:16** → **32 секунды**

---

## ❗️Почему повторяется сообщение в 08:39:57?

1. **PeerDB Mirror** может логировать это повторно в рамках внутренних проверок, например, при авто-коммитах или дополнительных проходах в idle-состоянии.
2. Это **не означает повторную загрузку данных**, иначе вы бы увидели `replicating new rows` или `change event`/`WAL applied` и т.д.
3. Повторные строки — скорее всего, от **разных worker'ов**, которые проверяют состояние или дают консистентную отметку об окончании.

---

## ✅ Вывод:

🟢 **Ориентироваться стоит на интервал: `08:38:44` — `08:39:16`**

🔁 Сообщения в `08:39:57` — это, скорее всего, **вторичные или избыточные логи** об уже завершенной репликации.

---

Если ты хочешь точно подтвердить это:

* Проверь наличие новых `change events` в логах после `08:39:16`. Их отсутствие подтвердит, что синхронизация завершена ранее.
* Посмотри, включен ли у тебя `log level: verbose` — иногда он дублирует сообщения.

---


Отличный вопрос — ты столкнулся с довольно типичной ошибкой при работе с PeerDB + ClickHouse через S3 в Docker-среде. Давайте разберем по шагам, что это за ошибка и куда копать.

---

## 🔥 Ошибка:

```
failed to sync records:
QRepSync Error: code: 499,
message: Could not list objects in bucket 'peerdbbucket' with prefix ...
S3 exception: 
message: 'Code: 198. DB::NetException: Not found address of host: host.docker.internal. (DNS_ERROR)'
```

---

## 💡 Что это значит?

Вот расшифровка:

### 🔍 Основной смысл:

* PeerDB (или ClickHouse, через PeerDB) пытается **прочитать файл из S3-хранилища**, в частности, по префиксу `clone_bptoch...`.
* Однако при этом он обращается к адресу `host.docker.internal`, чтобы разрешить DNS-имя (возможно, используется minio или локальный S3 endpoint).
* DNS-имя `host.docker.internal` **не разрешается в этой среде**, в которой работает ClickHouse.

---

## 🎯 Почему это происходит?

### Возможные причины:

1. **ClickHouse запущен в контейнере Docker** и **пытается обратиться к хост-машине** по имени `host.docker.internal` — но это работает **только в Docker Desktop (на Windows/Mac)** и не работает:

   * на Linux по умолчанию;
   * в некоторых Kubernetes-кластерах;
   * если ClickHouse не имеет доступа к сети хоста.

2. **PeerDB использует S3-совместимый backend**, например MinIO, и URL `host.docker.internal` был прописан как endpoint (или `external_url`) для S3.

   * Это имя нужно разрешить, но оно недоступно ClickHouse.

---

## 🧭 Куда копать:

### ✅ Проверь конфигурацию S3 в ClickHouse

Открой конфигурацию ClickHouse (например, `config.xml` или передаваемые параметры подключения в PeerDB) и проверь:

```xml
<s3>
  <endpoint>http://host.docker.internal:9000</endpoint>
</s3>
```

Если такое есть — ⚠️ **замени `host.docker.internal` на доступный адрес**, например:

* IP-адрес хост-машины (`172.17.0.1` — часто дефолт для Linux Docker);
* `minio:9000`, если MinIO бежит в том же `docker-compose`;
* публичный URL, если используешь внешнее S3-хранилище.

---

## 🛠 Что можно сделать

### 🔧 Решения:

| Сценарий                                    | Решение                                                                                      |
| ------------------------------------------- | -------------------------------------------------------------------------------------------- |
| Docker на **Linux**                         | Используй IP `172.17.0.1` или создай `host.docker.internal` в `/etc/hosts` внутри контейнера |
| ClickHouse и MinIO в одном `docker-compose` | Используй имена сервисов (`minio:9000`)                                                      |
| PeerDB настроен на `host.docker.internal`   | Измени конфиг PeerDB / S3 на IP или hostname, который **виден из ClickHouse**                |
| Kubernetes                                  | Настраивай S3 endpoint как **доступный внешний адрес**, не `host.docker.internal`            |

---

## ✅ Как проверить

Из контейнера ClickHouse (или среды, где он бежит) попробуй:

```bash
curl http://host.docker.internal:9000
```

Если не работает — значит адрес не резолвится, и его надо заменить.

---

## 🔚 Вывод

Ошибка говорит о том, что **ClickHouse не может достучаться до S3 endpoint, указанный как `host.docker.internal`**. Это DNS-проблема, которую нужно решать либо через:

* замену адреса на IP;
* использование правильного hostname в docker-сети;
* настройку `/etc/hosts`.

---


Спасибо за уточнение, теперь картина яснее.
Если ClickHouse у тебя **установлен на хост-машине напрямую**, а PeerDB работает из Docker, и при этом S3 ты не используешь — тогда ошибка с `host.docker.internal` несколько странная, но объяснимая. Давай разберёмся.

---

### Что происходит в твоём случае

* PeerDB в контейнере хочет получить доступ к ClickHouse, который слушает порт 9000 на хосте.
* `host.docker.internal` — это стандартное DNS имя, которое в Docker Desktop (Windows/Mac) резолвится в IP хоста. На Linux оно обычно не работает, но ты, возможно, его где-то в конфиге указал, и PeerDB пытается к нему обратиться.
* В логах PeerDB, видимо, эта строка говорит, что PeerDB **пытается достучаться до S3-совместимого хранилища (или так ему кажется)**, хотя у тебя S3 нет — возможно, PeerDB или ClickHouse пытается читать промежуточные данные из хранилища.

---

### Несколько важных моментов:

1. **Порт 9000 в ClickHouse — это TCP-порт для клиентских соединений (native protocol), а не HTTP.**
   PeerDB может использовать HTTP или S3 API, если это настроено, и тогда порт 9000 — это не то, что надо. Если у тебя нет S3, то, возможно, PeerDB пытается по ошибке подключиться как к S3.

2. **Пользователь по умолчанию и отсутствие пароля** — обычно ClickHouse разрешает подключения без пароля на localhost, но если PeerDB в Docker, то с точки зрения ClickHouse это уже не localhost. Может потребоваться аутентификация, или явное разрешение в конфиге ClickHouse.

3. **Ошибка DNS "host.docker.internal"** — значит где-то в конфиге PeerDB или в параметрах подключения к ClickHouse прописан именно этот хост. Если ClickHouse на хосте, то из контейнера доступ лучше настроить через IP хоста, например:

   * В Linux Docker хост обычно доступен через `172.17.0.1` (проверить `ip addr` для docker0 интерфейса).
   * Или прокинуть сеть `--network host` в контейнере PeerDB (если возможно).

---

### Что проверить и сделать:

1. **Проверь конфиг PeerDB по подключению к ClickHouse.**
   Там не должно быть ссылки на `host.docker.internal` в качестве адреса.

2. **Попробуй в PeerDB прописать IP хоста вместо `host.docker.internal`.**
   Например, `172.17.0.1` или IP интерфейса хоста.

3. **Проверь, что порт 9000 открыт и доступен из контейнера PeerDB.**
   В контейнере можно попробовать:

   ```bash
   telnet 172.17.0.1 9000
   ```

   или

   ```bash
   nc -zv 172.17.0.1 9000
   ```

4. **Убедись, что в ClickHouse в файле `users.xml` или конфиге разрешён доступ без пароля с адреса Docker-сети.**
   Либо создай пользователя с паролем и используй его в PeerDB.

5. **Если PeerDB пытается использовать S3 для промежуточных файлов (как буфер), хотя ты не настраивал S3, — уточни в документации PeerDB, не активирован ли у тебя режим работы с S3 по умолчанию.**
   Возможно, при репликации в ClickHouse PeerDB использует S3 как staging area, и ему надо правильно указать endpoint (если локального S3 нет — нужно отключить эту опцию).

---

### Итог:

* Ошибка с DNS — это то, что контейнер PeerDB пытается достучаться по `host.docker.internal`, который не резолвится.
* Нужно либо изменить адрес в конфиге PeerDB на IP хоста, либо правильно настроить DNS.
* Проверь конфигурацию подключения PeerDB к ClickHouse — порт, пользователя, пароль.
* Проверь, не активирована ли опция S3 в PeerDB для ClickHouse.

---



Хороший вопрос! Где искать конфиг PeerDB — зависит от того, как именно ты его запускаешь и устанавливал.

---

## Где обычно находится конфигурация PeerDB?

### 1. **Если PeerDB запускается из Docker**

* Конфигурационные файлы обычно **монтируются как volume** снаружи контейнера.
* Проверь команду запуска или `docker-compose.yml` — там может быть что-то вроде:

```yaml
volumes:
  - ./peerdb-config:/app/config
```

Или:

```bash
docker run -v /path/to/config:/app/config peerdb-image
```

* В таком случае конфиг лежит в директории `/path/to/config` на хосте.

### 2. **Если PeerDB установлен как бинарь или пакет**

* Обычно конфиг находится в одном из стандартных путей:

  * `/etc/peerdb/peerdb.yaml`
  * `~/.peerdb/config.yaml`
  * В директории, откуда запускается приложение (например, `./config.yaml`)

### 3. **Если PeerDB запускается через CLI**

* Часто конфиг передается параметром запуска, например:

```bash
peerdb --config /path/to/config.yaml
```

* Проверь скрипт или systemd unit, который запускает PeerDB.

---

## Что искать в конфиге?

* Блок с настройками подключения к ClickHouse — там будет host, port, user, password.
* Возможно, настройки S3 (endpoint, bucket), если используются.
* Там может быть что-то вроде:

```yaml
clickhouse:
  host: host.docker.internal
  port: 9000
  user: default
  password: ""
```

---

## Что делать, если не можешь найти конфиг?

* Попробуй поискать в репозитории или документации PeerDB, где конфиг лежит по умолчанию.
* Если запускаешь через Docker — посмотри в `docker-compose.yml` или команду запуска `docker run`.
* Если PeerDB — это сервис в системе, посмотри через:

```bash
ps aux | grep peerdb
```

Чтобы увидеть, с какими параметрами он запускается.

---


Отличный вопрос, и ты близко к разгадке — дело тут не только в правах на файл, а в том, **как ClickHouse управляет пользователями и откуда именно он их загружает**.

---

## 🧠 Суть ошибки:

```text
DB::Exception: Cannot update user `ch` in users_xml because this storage is readonly. (ACCESS_STORAGE_READONLY)
```

Это означает:

> ClickHouse использует **users.xml** (или другой XML-файл), как **readonly-источник пользователей**. А ты пытаешься изменить пользователя (или его права) **через SQL**, что невозможно при таком способе хранения.

---

## 🔍 Почему так происходит

ClickHouse поддерживает разные "источники доступа" (`access storages`), такие как:

| Источник       | Хранит пользователей и права где? | Можно менять через SQL? |
| -------------- | --------------------------------- | ----------------------- |
| `users.xml`    | В XML-файле                       | ❌ Нет (readonly)        |
| `users.d`      | Отдельные XML-файлы в директории  | ❌ Нет                   |
| `users` (RBAC) | В internal-хранилище ClickHouse   | ✅ Да                    |

Ты сейчас используешь **users.xml**, поэтому любая попытка изменить права через SQL вызывает ошибку `ACCESS_STORAGE_READONLY`.

---

## ✅ Как это исправить

### 🔧 Вариант 1: Перейти на RBAC (рекомендуется)

1. **Открой основной конфиг ClickHouse** (обычно `/etc/clickhouse-server/config.xml`).
2. Найди или добавь в блок `<access_control>`:

```xml
<access_control>
    <path>access/</path>
</access_control>
```

3. Удали/отключи `users.xml`, чтобы ClickHouse не считывал пользователей из XML.
   Обычно это в `users.xml`, `users.d`, или `<users>` блоке.

4. Перезапусти ClickHouse:

```bash
sudo systemctl restart clickhouse-server
```

Теперь можно будет делать:

```sql
CREATE USER ch IDENTIFIED BY 'password';
GRANT INSERT, SELECT ON test.* TO ch;
```

---

### 🔧 Вариант 2: Изменить права вручную в `users.xml`

1. Открой файл `users.xml` (обычно `/etc/clickhouse-server/users.xml`).
2. Найди пользователя `ch` и добавь ему нужные права вручную:

```xml
<users>
    <ch>
        <password></password>
        <networks>
            <ip>::/0</ip>
        </networks>
        <profile>default</profile>
        <quota>default</quota>
        <allow_databases>
            <database>test</database>
        </allow_databases>
    </ch>
</users>
```

Но ⚠️ **права типа GRANT/REVOKE всё равно не заработают**, так как XML — это статичная конфигурация.

---

### ℹ️ Проверка источников доступа

Выполни в ClickHouse:

```sql
SHOW ACCESS STORAGES;
```

Если увидишь:

```text
users_xml
```

— значит, это readonly.

---

## ✅ Рекомендуемый путь

Переключись на **RBAC (роль-базированное управление)** — оно позволяет:

* использовать `CREATE USER`, `GRANT`, `REVOKE`, и прочие SQL-команды;
* работать удобно с правами и ролями;
* гибко управлять доступом — особенно если PeerDB будет подключаться и выполнять DDL.

---


Отличный и точный вопрос! Краткий ответ: **нет, всё это напрямую в `users.xml` полноценно сделать нельзя.**

---

## 📌 Почему?

`users.xml` — это **статическая конфигурация**, и она **не поддерживает полную SQL-гранулярность** (как `GRANT ... ON ...`) или продвинутые права типа `ALTER ADD COLUMN`, `s3`, `CREATE TEMPORARY TABLE`, и т.п.

Она поддерживает только очень базовые параметры:

* имя пользователя
* пароль
* доступ по IP
* профиль (`<profile>`)
* квазиправа в виде разрешённых БД (`<allow_databases>`), но **не SQL-уровня GRANT'ы**.

---

## 🔍 Что можно в `users.xml`

Ты можешь в `users.xml` прописать:

```xml
<users>
  <peerdb_user>
    <password>yourpassword</password>
    <networks>
      <ip>::/0</ip> <!-- или ограничение по IP -->
    </networks>
    <profile>default</profile>
    <quota>default</quota>
    <allow_databases>
      <database>peerdb</database>
    </allow_databases>
  </peerdb_user>
</users>
```

Но этого недостаточно для PeerDB, потому что:

* **нет управления на уровне отдельных привилегий** (`ALTER`, `CREATE TEMPORARY TABLE`, `DROP`)
* **нет GRANT'ов на объекты, только на базу целиком**
* **невозможно разрешить `s3` доступ**, потому что он контролируется через SQL-уровень

---

## ✅ Что делать

### Рекомендуется: **включить RBAC (ролевую модель)**

Вот что тебе нужно:

1. В `config.xml` включи RBAC (если ещё не включено):

```xml
<access_control>
  <path>/var/lib/clickhouse/access/</path>
</access_control>
```

2. Удали или отключи `<users>` в `users.xml`, чтобы ClickHouse не мешал:

```xml
<!-- <users>...</users> -->
```

3. Перезапусти ClickHouse:

```bash
sudo systemctl restart clickhouse-server
```

4. Теперь можно выполнять из ClickHouse SQL-команды:

```sql
CREATE USER peerdb_user IDENTIFIED BY 'yourpassword';

GRANT INSERT, SELECT, DROP, CREATE TABLE ON peerdb.* TO peerdb_user;
GRANT ALTER ADD COLUMN ON peerdb.* TO peerdb_user;
GRANT CREATE TEMPORARY TABLE, s3 ON *.* TO peerdb_user;
```

---

## 🧪 Альтернатива (временная): комбинировать users.xml и SQL

* Можешь завести пользователя в `users.xml`, но **GRANT'ы сделать позже в SQL**, **если RBAC включён**.
* В этом случае `users.xml` будет просто местом создания пользователя, а всё управление правами — через SQL.

---

## 🟨 Вывод

| Действие                 | Поддерживается в `users.xml`? |
| ------------------------ | ----------------------------- |
| `CREATE USER`            | ✅ (ограниченно)               |
| `GRANT INSERT, SELECT`   | ❌                             |
| `GRANT ALTER ADD COLUMN` | ❌                             |
| `GRANT s3`               | ❌                             |
| `GRANT CREATE TEMPORARY` | ❌                             |
| Полный RBAC через SQL    | ✅ (при включённом RBAC)       |

---


