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

