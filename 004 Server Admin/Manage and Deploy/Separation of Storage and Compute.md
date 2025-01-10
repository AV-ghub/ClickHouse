## [Separation of Storage and Compute](https://clickhouse.com/docs/en/guides/separation-storage-compute)
Self-managed ClickHouse allows for separation of storage and compute.  
### [Using Multiple Block Devices for Data Storage](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/mergetree#table_engine-mergetree-multiple-volumes)
If several disks are available, the “hot” data may be located on fast disks (for example, NVMe SSDs or in memory), while the “cold” data - on relatively slow ones (for example, HDD).  
Data part is the minimum movable unit for MergeTree-engine tables. The data belonging to one part are stored on one disk. Data parts can be moved between disks in the background (according to user settings) as well as by means of the ALTER queries.  
The names given to the described entities can be found in the system.disks
```
:) select * from system.disks\G
SELECT *
FROM system.disks
Query id: b19bbea5-b0cb-4179-a62a-3fb6a4fda06e
Row 1:
──────
name:                default
path:                /var/lib/clickhouse/
free_space:          219591442432
total_space:         905565224960
...
```

