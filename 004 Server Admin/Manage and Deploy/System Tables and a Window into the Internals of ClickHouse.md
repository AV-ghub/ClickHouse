## [System Tables and a Window into the Internals of ClickHouse](https://clickhouse.com/blog/clickhouse-debugging-issues-with-system-tables)
System tables are located in the system database and are only available for reading.   
They cannot be dropped or altered, but their partition can be detached and old records can be removed using TTL.   

## Additional resoyrces
[System Tables](https://clickhouse.com/docs/en/operations/system-tables)   
