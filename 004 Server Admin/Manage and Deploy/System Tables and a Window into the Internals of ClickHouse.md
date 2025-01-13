## [System Tables and a Window into the Internals of ClickHouse](https://clickhouse.com/blog/clickhouse-debugging-issues-with-system-tables)
System tables are located in the system database and are only available for reading.   
They cannot be dropped or altered, but their partition can be detached and old records can be removed using TTL.   

The complete list of system tables is accessible via the 
```
SHOW TABLES FROM system
```
### [Hot tips for querying system tables](https://clickhouse.com/blog/clickhouse-debugging-issues-with-system-tables#hot-tips-for-querying-system-tables)
#### What settings were changed from the default value
```
SELECT *
FROM system.settings
WHERE changed
LIMIT 2
FORMAT Vertical

Row 1:
──────
name:        max_insert_threads
value:       4
changed:     1
```
#### [What are the long-running queries? Which queries took up the most memory?](https://clickhouse.com/blog/clickhouse-debugging-issues-with-system-tables#what-are-the-long-running-queries-which-queries-took-up-the-most-memory)
```
SELECT
    type,
    event_time,
    query_duration_ms,
    initial_query_id,
    formatReadableSize(memory_usage) AS memory,
    `ProfileEvents.Values`[indexOf(ProfileEvents.Names, 'UserTimeMicroseconds')] AS userCPU,
    `ProfileEvents.Values`[indexOf(ProfileEvents.Names, 'SystemTimeMicroseconds')] AS systemCPU,
    normalizedQueryHash(query) AS normalized_query_hash,
    substring(normalizeQuery(query) AS query, 1, 100)
FROM clusterAllReplicas(default, system.query_log)
ORDER BY query_duration_ms DESC
LIMIT 2
FORMAT Vertical
```
#### [Which queries have failed](https://clickhouse.com/blog/clickhouse-debugging-issues-with-system-tables#which-queries-have-failed)
```
SELECT
    type,
    query_start_time,
    query_duration_ms,
    query_id,
    query_kind,
    is_initial_query,
    normalizeQuery(query) AS normalized_query,
    concat(toString(read_rows), ' rows / ', formatReadableSize(read_bytes)) AS read,
    concat(toString(written_rows), ' rows / ', formatReadableSize(written_bytes)) AS written,
    concat(toString(result_rows), ' rows / ', formatReadableSize(result_bytes)) AS result,
    formatReadableSize(memory_usage) AS `memory usage`,
    exception,
    concat('\n', stack_trace) AS stack_trace,
    user,
    initial_user,
    multiIf(empty(client_name), http_user_agent, concat(client_name, ' ', toString(client_version_major), '.', toString(client_version_minor), '.', toString(client_version_patch))) AS client,
    client_hostname,
    databases,
    tables,
    columns,
    used_aggregate_functions,
    used_aggregate_function_combinators,
    used_database_engines,
    used_data_type_families,
    used_dictionaries,
    used_formats,
    used_functions,
    used_storages,
    used_table_functions,
    thread_ids,
    ProfileEvents,
    Settings
FROM system.query_log
WHERE type IN ['3', '4']
ORDER BY query_start_time DESC
LIMIT 1
FORMAT Vertical
```
#### [What are the common errors](https://clickhouse.com/blog/clickhouse-debugging-issues-with-system-tables#what-are-the-common-errors)
```
SELECT
    name,
    code,
    value,
    last_error_time,
    last_error_message,
    last_error_trace AS remote
FROM system.errors
LIMIT 1
FORMAT Vertical
```
#### [Are parts being created when the rows are inserted](https://clickhouse.com/blog/clickhouse-debugging-issues-with-system-tables#are-parts-being-created-when-the-rows-are-inserted)
```
SELECT
    event_time,
    event_time_microseconds,
	table,
    rows,
	*
FROM system.part_log
WHERE (database = 'default') AND (event_type IN ['NewPart'])
ORDER BY event_time ASC
LIMIT 1
FORMAT VERTICAL
```
#### [What is the status of the in-progress merges](https://clickhouse.com/blog/clickhouse-debugging-issues-with-system-tables#what-is-the-status-of-the-in-progress-merges)
```
SELECT
    hostName(),
    database,
    table,
    round(elapsed, 0) AS time,
    round(progress, 4) AS percent,
    formatReadableTimeDelta((elapsed / progress) - elapsed) AS ETA,
    num_parts,
    formatReadableSize(memory_usage) AS memory_usage,
    result_part_name
FROM system.merges
ORDER BY (elapsed / percent) - elapsed ASC
FORMAT Vertical
```
#### [Are there parts with errors](https://clickhouse.com/blog/clickhouse-debugging-issues-with-system-tables#are-there-parts-with-errors)
```
SELECT
    event_date,
    event_type,
    table,
    error AS error_code,
    errorCodeToName(error) AS error_code_name,
    count() as c
FROM system.part_log
WHERE (error_code != 0) AND (event_date > (now() - toIntervalMonth(1)))
GROUP BY
    event_date,
    event_type,
    error,
    table
ORDER BY
    event_date DESC,
    event_type ASC,
    table ASC,
    error ASC
```
#### [Are there long-running mutations that are stuck](https://clickhouse.com/blog/clickhouse-debugging-issues-with-system-tables#are-there-long-running-mutations-that-are-stuck)
```
SELECT
    database,
    table,
    mutation_id,
    command,
    create_time,
    parts_to_do_names,
    parts_to_do,
    is_done,
    latest_failed_part,
    latest_fail_time,
    latest_fail_reason
FROM system.mutations
WHERE NOT is_done
ORDER BY create_time DESC
```
#### [How much disk space are the tables using](https://clickhouse.com/blog/clickhouse-debugging-issues-with-system-tables#how-much-disk-space-are-the-tables-using)
```
SELECT
    hostName(),
    database,
    table,
    sum(rows) AS rows,
    formatReadableSize(sum(bytes_on_disk)) AS total_bytes_on_disk,
    formatReadableSize(sum(data_compressed_bytes)) AS total_data_compressed_bytes,
    formatReadableSize(sum(data_uncompressed_bytes)) AS total_data_uncompressed_bytes,
    round(sum(data_compressed_bytes) / sum(data_uncompressed_bytes), 3) AS compression_ratio
FROM system.parts
WHERE database != 'system'
GROUP BY
    hostName(),
    database,
    table
ORDER BY sum(bytes_on_disk) DESC FORMAT Vertical
```
#### Table size
```
SELECT
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio
FROM system.columns
WHERE `table` = 'dbo_calls'
```
#### Table columns size
```
SELECT
    name,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio
FROM system.columns
WHERE table = 'dbo_calls'
GROUP BY name
ORDER BY sum(data_compressed_bytes) DESC
```

## Additional resources
[System Tables](https://clickhouse.com/docs/en/operations/system-tables)   












