## [Migrating from PostgreSQL to ClickHouse](https://clickhouse.com/docs/en/migrations/postgresql/overview)
### [Why use ClickHouse over Postgres?](https://clickhouse.com/docs/en/migrations/postgresql/overview#why-use-clickhouse-over-postgres)
ClickHouse is designed for fast analytics, specifically **GROUP BY queries**.   
Postgres is an OLTP database designed for **transactional workloads**.   

OLTP databases typically hit performance limitations when used for analytical queries over large datasets.   
OLAP. The primary objective of these databases is to ensure that engineers can efficiently **query and aggregate over vast datasets**.    
Real-time OLAP systems like ClickHouse allow this analysis to happen as **data is ingested in real time**.

## [Adding Real Time Analytics to a Supabase Application With ClickHouse](https://clickhouse.com/blog/adding-real-time-analytics-to-a-supabase-application)
### [Foreign Data Wrappers - a single endpoint](https://clickhouse.com/blog/adding-real-time-analytics-to-a-supabase-application#foreign-data-wrappers---a-single-endpoint)
We can query ClickHouse directly using client.    
If we want to communicate through a single interface we can use [Foreign Data Wrappers](https://supabase.com/blog/postgres-foreign-data-wrappers-rust) (FDW).    
These allow us to connect to external systems such as ClickHouse, querying the data in place.  

> Foreign Data Wrappers have a concept of "push down". This means that the FDW runs the query on remote data source. This is useful for performance reasons, as the remote data source can often perform the query more efficiently than Postgres. Push down is also useful for security reasons, as the remote data source can enforce access control. Limited push-down support has been added as a Proof of Concept, but this will be a key focus of Wrappers.
> You can follow development of all the Wrappers in the official GitHub Repo.

Whereas other similar technologies may aim to pull the datasets into the query engine to perform filtering and aggregations,    
FDWs emphasize relying on an extract-only methodology where the query is pushed down and aggregation and filter performed in the target data source.    
**Only the results are then returned** to Postgres for display.





## Additional resources
### Integrations
[Rewriting PostgreSQL queries in ClickHouse](https://clickhouse.com/docs/en/migrations/postgresql/rewriting-queries)   
[Loading data from PostgreSQL to ClickHouse](https://clickhouse.com/docs/en/migrations/postgresql/dataset)    

### Chats
[Apache ECharts](https://echarts.apache.org/en/index.html)   

### FDW
[Postgres FDW for Clickhouse](https://github.com/messagebird/clickhouse-postgres-fdw)

