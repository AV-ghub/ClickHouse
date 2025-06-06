## ClickPipes от ClickHouse 

Предназначена для интеграции данных из PostgreSQL в ClickHouse, но на данный момент она доступна только в облачной версии ClickHouse Cloud и находится на стадии публичной бета-версии. Это означает, что ClickPipes не поддерживает локальные инстансы ClickHouse или PostgreSQL. ([clickhouse.com][1])

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

**Рекомендации:**

* Для стабильной и поддерживаемой интеграции рекомендуется использовать таблицы ClickHouse с движком PostgreSQL.
* Если вы хотите настроить репликацию данных, рассмотрите использование PeerDB.
* Будьте осторожны при использовании экспериментальных функций, таких как `MaterializedPostgreSQL`, в продакшн-средах.([clickhouse.com][1], [github.com][3])

[1]: https://clickhouse.com/docs/ru/integrations/clickpipes/postgres?utm_source=chatgpt.com "Погружение данных из Postgres в ClickHouse (с использованием CDC) | ClickHouse Docs"
[2]: https://clickhouse.com/docs/ru/integrations/clickpipes/postgres/faq?utm_source=chatgpt.com "Часто задаваемые вопросы о ClickPipes для Postgres | ClickHouse Docs"
[3]: https://github.com/ClickHouse/clickhouse-docs/blob/main/docs/en/integrations/data-ingestion/dbms/postgresql/connecting-to-postgresql.md?utm_source=chatgpt.com "clickhouse-docs/docs/en/integrations/data-ingestion/dbms/postgresql/connecting-to-postgresql.md at main · ClickHouse/clickhouse-docs · GitHub"

---

##  PeerDB

Инструмент для репликации данных из PostgreSQL в ClickHouse, ориентированный на высокую производительность и простоту настройки. Он поддерживает два основных режима интеграции: CDC (Change Data Capture) и Query Replication.([threequants.io][1])

Как работает PeerDB?

PeerDB использует PostgreSQL Logical Replication Slots для отслеживания изменений в базе данных и их передачи в ClickHouse. Процесс репликации включает два этапа:([clickhouse.com][2])

1. **Начальная загрузка (Initial Load):** PeerDB выполняет параллельное считывание данных из PostgreSQL и их загрузку в ClickHouse, что значительно ускоряет процесс по сравнению с традиционными методами. ([blog.peerdb.io][3])

2. **Репликация изменений (CDC):** После начальной загрузки PeerDB продолжает отслеживать изменения в PostgreSQL через репликационный слот и передавать их в ClickHouse с минимальной задержкой. ([blog.peerdb.io][3])

PeerDB автоматически создает таблицы в ClickHouse с использованием движка `ReplacingMergeTree` (но можно выбрать другие варианты, также как и первичный ключ, но при этом PK из PG обязательно должен быть первым), что позволяет эффективно обрабатывать обновления и удаления данных. ([blog.peerdb.io][4])

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

## 🏗️ Настройка PeerDB для локальной интеграции

Для настройки PeerDB в локальной среде с PostgreSQL и ClickHouse выполните следующие шаги:

### 1. [Установите PeerDB](https://docs.peerdb.io/quickstart/quickstart)

```bash
git clone --recursive https://github.com/PeerDB-io/peerdb.git
cd peerdb

# Run docker containers: peerdb-server, postgres as catalog, temporal.
# This might take a few minutes, so get a cup of coffee! :)
./run-peerdb.sh
```

### 2. **Настройте PostgreSQL:**

   * Создайте публикацию для логической репликации:

     ```sql
     CREATE PUBLICATION peerdb_publication FOR ALL TABLES;
     ```
   * Создайте репликационный слот:

     ```sql
     SELECT pg_create_logical_replication_slot('peerdb_slot', 'pgoutput');
     ```

На самом деле PeerDB создает все это автоматом при создании зеркала. Также автоматом и удаляет при его удалении.  
Все это делается через UI (http://localhost:3000/)

### 3. **Настройте ClickHouse:**

   * Создайте базу данных и таблицы, соответствующие структуре данных в PostgreSQL.

На самом деле не обязательно, PeerDB также автоматом все это создает при создании зеркала (руками нужно создать, если хочется определенные типы полей, кодеки пожатия и т.п.), но не удаляет при его удалении и переиспользует при повторном создании зеркала.

   * В созданной базе создайте пользователя и [назначьте права](https://docs.peerdb.io/connect/clickhouse/clickhouse-cloud#permissions)

Нюанс. Пользователя лучше создать нового. Особенно, если существующий создан через xml config.

```sql
CREATE DATABASE test;

-- We recommend creating a separate user for PeerDB
CREATE USER pr IDENTIFIED BY 'pr';

-- PeerDB needs to create tables and insert data into the tables.
-- Drop table permission is needed for DROP MIRROR support
GRANT INSERT, SELECT, DROP, CREATE TABLE ON test.* to pr;

-- PeerDB uses an intermediary S3 stage for performance
GRANT CREATE TEMPORARY TABLE, s3 on *.* to pr;

-- For automatic column-addition on the tables in the mirror
GRANT ALTER ADD COLUMN ON test.* to pr;
```

Всё то же самое напрямую в users.xml (для забитого туда напрямую пользователя) сделать нельзя.   
Будет ругаться что файлик readonly, даже если на него раздать вообще все права.   
Только новый пользователь. В этом случае он уже пойдет по новой политике ролей (RBAC).



### 4. **Настройте PeerDB:**

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

### ✅ Установка Docker и Docker Compose на AlmaLinux 9

1. **Обновите систему:**

   ```bash
   sudo yum update -y
   ```



2. **Добавьте официальный репозиторий Docker:**

   ```bash
   sudo yum config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
   ```



3. **Установите Docker и Docker Compose:**

   ```bash
   sudo yum install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
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

### 🌐 Доступность Docker

Официальный репозиторий Docker для CentOS, который используется для AlmaLinux, доступен по адресу:

```
https://download.docker.com/linux/centos/docker-ce.repo
```

Чтобы использовать **логическую репликацию** в PostgreSQL (например, для CDC с PeerDB, Debezium и пр.), нужно внести изменения в конфигурацию PostgreSQL и перезапустить сервер:

---

## Включение логической репликации

### Файл конфигурации PostgreSQL

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

### Доступ в `pg_hba.conf`

```conf
host    replication     your_user       192.168.0.0/24         md5
```

---

### Перезапустите PostgreSQL

```bash
sudo systemctl restart postgresql
```

---

### Создайте пользователя с правами репликации

Выполните в `psql` под пользователем `postgres`:

```sql
CREATE ROLE replication_user WITH REPLICATION LOGIN PASSWORD 'your_password';
```

Или просто пускаем все от postgres.

---

### Проверьте настройки

Подключитесь к PostgreSQL и проверьте:

```sql
SHOW wal_level;
```
должен возвращать:

```
logical
```

```sql
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

## ✅ Установка PeerDB без Docker на AlmaLinux

На данный момент PeerDB официально поддерживает развертывание только через Docker Compose, и в документации нет описания установки без использования Docker. ([docs.peerdb.io][1])

Однако, если вы хотите установить PeerDB на AlmaLinux без использования Docker, вам потребуется выполнить следующие шаги:

### 1. **Установите необходимые зависимости:**

   PeerDB требует наличия Node.js и npm. Установите их с помощью следующих команд:

   ```bash
   sudo dnf install -y nodejs npm
   ```

### 2. **Склонируйте репозиторий PeerDB:**

   Скачайте исходный код PeerDB с GitHub:

   ```bash
   git clone --recursive https://github.com/PeerDB-io/peerdb.git
   cd peerdb
   ```

### 3. **Установите зависимости:**

   В каталоге проекта выполните команду для установки всех необходимых пакетов:

   ```bash
   npm install
   ```

### 4. **Настройте базу данных:**

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

### 5. **Настройте конфигурацию PeerDB:**

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

### 6. **Запустите PeerDB:**

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

---























## Разбор логов PeerDB

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

## 🔥 Ошибка:

```
failed to sync records:
QRepSync Error: code: 499,
message: Could not list objects in bucket 'peerdbbucket' with prefix ...
S3 exception: 
message: 'Code: 198. DB::NetException: Not found address of host: host.docker.internal. (DNS_ERROR)'
```

---

### 🔍 Основной смысл:

* ClickHouse настраивается PeerDB для чтения S3 подобного буфера (MinIO), находящегося в контейнере с PeerDB.
* При этом он пфтается обращаться к адресу `host.docker.internal`, прописанному в docker-compose.yml.
* Который **не разрешается в этой среде**, в которой работает ClickHouse.

---

### 🎯 Почему это происходит?

**PeerDB использует S3-совместимый backend**, например MinIO, и URL `host.docker.internal` был прописан как endpoint (или `external_url`) для S3.

### Где находится конфигурация PeerDB?

```bash
ps aux | grep peerdb
```

Чтобы увидеть, с какими параметрами он запускается.

### Если PeerDB запускается из Docker

* Конфигурационные файлы находятся снаружи контейнера (в точке запуска - .../peerdb/run-peerdb.sh).
* Это docker-compose.yml

### Что поменять в конфиге?

```yaml
#  PEERDB_CLICKHOUSE_AWS_CREDENTIALS_AWS_ENDPOINT_URL_S3: http://host.docker.internal:9001
  PEERDB_CLICKHOUSE_AWS_CREDENTIALS_AWS_ENDPOINT_URL_S3: http://<Host IP>:9001
```

---













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
