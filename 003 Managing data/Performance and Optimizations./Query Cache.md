## [Query Cache](https://clickhouse.com/docs/en/operations/query-cache)
```
SELECT some_expensive_calculation(column_1, column_2)
FROM table
SETTINGS use_query_cache = true
```
[enable_writes_to_query_cache](https://clickhouse.com/docs/en/operations/settings/settings#enable-writes-to-query-cache)   
[enable_reads_from_query_cache](https://clickhouse.com/docs/en/operations/settings/settings#enable-reads-from-query-cache)   
[query_cache_max_size_in_bytes](https://clickhouse.com/docs/en/operations/settings/settings#query-cache-max-size-in-bytes)  
[query_cache_max_entries](https://clickhouse.com/docs/en/operations/settings/settings#query-cache-max-entries)   
[query_cache_min_query_duration](https://clickhouse.com/docs/en/operations/settings/settings#query-cache-min-query-duration)   
[query_cache_min_query_runs](https://clickhouse.com/docs/en/operations/settings/settings#query-cache-min-query-runs)   
...

## Additional resources
### [Introducing the ClickHouse Query Cache](https://clickhouse.com/blog/introduction-to-the-clickhouse-query-cache-and-design)
```
:) select toDate(starttime), max(remotephonenumber), count()
from dbo_calls
where remotephonenumber like '99%'
group by toDate(starttime)
SETTINGS use_query_cache = true, query_cache_ttl = 3000

5 rows in set. Elapsed: 0.253 sec. Processed 30.01 million rows, 602.40 MB (118.57 million rows/s., 2.38 GB/s.)
Peak memory usage: 2.66 MiB.
```
```
SELECT
    toDate(starttime),
    max(remotephonenumber),
    count()
FROM dbo_calls
WHERE remotephonenumber LIKE '99%'
GROUP BY toDate(starttime)
SETTINGS use_query_cache = true
...
5 rows in set. Elapsed: 0.002 sec.
```
```
SELECT *
FROM system.query_cache
FORMAT vertical

Query id: f3184c8f-c2ff-44e5-8c8f-de5db46ac278

Row 1:
──────
query:       SELECT toDate(starttime), max(remotephonenumber), count() FROM dbo_calls WHERE remotephonenumber LIKE '99%' GROUP BY toDate(starttime) SETTINGS use_query_cache = true, query_cache_ttl = 3000
result_size: 592
tag:
stale:       0
shared:      0
compressed:  1
expires_at:  2025-01-13 19:50:24
key_hash:    18059452253961218132

1 row in set. Elapsed: 0.002 sec.
```

To quickly points to any issue run the query again after executing **SET send_logs_level = 'trace'**.
```
2023.01.29 12:15:26.592519 [ 1371064 ] {a645c5b7-09a2-456c-bc8b-c506828d3b69}  QueryCache: Skipped insert (query result too big), new_entry_size_in_bytes: 1328640,
new_entry_size_in_rows: 50
...
```



