[Dictoinary](https://clickhouse.com/docs/en/dictionary)

Dictionaries are useful for:

* Improving the **performance** of queries, especially when used **with JOINs**
* **Enriching** ingested data on the fly without slowing down the ingestion process

Dictionaries can be used to speed up a specific type of JOIN: the [LEFT ANY type](https://clickhouse.com/blog/clickhouse-fully-supports-joins-part1#left--right--inner-any-join)

If this is the case, ClickHouse can exploit the dictionary to perform a **Direct Join**. 

Dictionaries are usually held in memory ([ssd_cache](https://clickhouse.com/docs/en/sql-reference/dictionaries#ssd_cache) is the exception)
