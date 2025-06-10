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


