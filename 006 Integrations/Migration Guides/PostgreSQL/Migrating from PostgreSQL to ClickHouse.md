## [Migrating from PostgreSQL to ClickHouse](https://clickhouse.com/docs/en/migrations/postgresql/overview)
### [Why use ClickHouse over Postgres?](https://clickhouse.com/docs/en/migrations/postgresql/overview#why-use-clickhouse-over-postgres)
ClickHouse is designed for fast analytics, specifically **GROUP BY queries**.   
Postgres is an OLTP database designed for **transactional workloads**.   

OLTP databases typically hit performance limitations when used for analytical queries over large datasets.   
OLAP. The primary objective of these databases is to ensure that engineers can efficiently **query and aggregate over vast datasets**.    
Real-time OLAP systems like ClickHouse allow this analysis to happen as **data is ingested in real time**.
