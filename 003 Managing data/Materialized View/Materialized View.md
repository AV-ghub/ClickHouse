## [Materialized View](https://clickhouse.com/docs/en/materialized-view)
Materialized views allow users to **shift the cost of computation from query time to insert time**, resulting in faster SELECT queries.  

ClickHouse materialized view is just a trigger that **runs a query on blocks of data as they are inserted** into a table.  
The result of this query is inserted into a second "target" table.    
Should more rows be inserted, **results will again be sent to the target table** where the intermediate results will be updated **and merged**.    
This **merged result is the equivalent of** running the query over **all of the original data**. 

Materialized views in ClickHouse are **updated in real time** as data flows into the table they are based on, functioning more like continually updating indexes.   



## Additional resources
[Chaining Materialized Views in ClickHouse](https://clickhouse.com/blog/chaining-materialized-views)

