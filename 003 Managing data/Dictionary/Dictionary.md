[Dictoinary](https://clickhouse.com/docs/en/dictionary)

Dictionaries are useful for:

* Improving the **performance** of queries, especially when used **with JOINs**
* **Enriching** ingested data on the fly without slowing down the ingestion process

Dictionaries can be used to speed up a specific type of JOIN: the [LEFT ANY type](https://clickhouse.com/blog/clickhouse-fully-supports-joins-part1#left--right--inner-any-join)

If this is the case, ClickHouse can exploit the dictionary to perform a **Direct Join**. 

Dictionaries are usually held in memory ([ssd_cache](https://clickhouse.com/docs/en/sql-reference/dictionaries#ssd_cache) is the exception)

We use dictionary at query time **to remove a join**.

#### [Advanced Dictionary Topics](https://clickhouse.com/docs/en/dictionary#advanced-dictionary-topics)
[Layouts](https://clickhouse.com/docs/en/sql-reference/dictionaries#ways-to-store-dictionaries-in-memory)    
[Choosing a Layout](https://clickhouse.com/blog/faster-queries-dictionaries-clickhouse#choosing-a-layout)

#### Refreshing dictionaries
LIFETIME is the update interval for the dictionary, with the values here causing a periodic reload at a random interval.   
Random interval is necessary in order to distribute the load on the dictionary source when updating on a large number of servers.   
Setting (LIFETIME(0)) prevents dictionaries from updating.   
Dictionaries can be forcibly reloaded using the SYSTEM RELOAD DICTIONARY command.  

For database sources such as **ClickHouse** and **Postgres**, you can set up a query that will update the dictionaries only if they really changed (the response of the query determines this), rather than at a periodic interval. [Further details](https://clickhouse.com/docs/en/sql-reference/dictionaries#refreshing-dictionary-data-using-lifetime)
