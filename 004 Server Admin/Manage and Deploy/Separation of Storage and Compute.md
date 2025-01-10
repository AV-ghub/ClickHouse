## [Separation of Storage and Compute](https://clickhouse.com/docs/en/guides/separation-storage-compute)
Self-managed ClickHouse allows for separation of storage and compute.  
### [Using Multiple Block Devices for Data Storage](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/mergetree#table_engine-mergetree-multiple-volumes)
If several disks are available, the “hot” data may be located on fast disks (for example, NVMe SSDs or in memory), while the “cold” data - on relatively slow ones (for example, HDD).  
Data part is the minimum movable unit for MergeTree-engine tables. The data belonging to one part are stored on one disk. Data parts can be moved between disks in the background (according to user settings) as well as by means of the ALTER queries.  
The names given to the described entities can be found in
```
select * from system.disks\G
──────
name:                default
path:                /var/lib/clickhouse/
free_space:          219591442432
total_space:         905565224960
...
```
### [Configuring external storage](https://clickhouse.com/docs/en/operations/storing-data#configuring-external-storage)
Disk configuration requires:

* type section, equal to one of s3, azure_blob_storage, hdfs (unsupported), **local_blob_storage**, web.

type equal to object_storage

* object_storage_type, equal to one of s3, azure_blob_storage (or just azure from 24.3), hdfs (unsupported), **local_blob_storage** (or just **local** from 24.3), web.

### [Dynamic Configuration](https://clickhouse.com/docs/en/operations/storing-data#dynamic-configuration)
```
ATTACH TABLE uk_price_paid UUID 'cf712b4f-2ca8-435c-ac23-c4393efe52f7'
(
...
)
ENGINE = MergeTree
ORDER BY (postcode1, postcode2, addr1, addr2)
SETTINGS disk = disk(
    type=cache,
    max_size='1Gi',
    path='/var/lib/clickhouse/custom_disk_cache/',
    disk=disk(
      type=web,
      endpoint='https://raw.githubusercontent.com/ClickHouse/web-tables-demo/main/web/'
      )
  );
```
> The example uses type=web, but any disk type can be configured as dynamic, even Local disk. **Local disks require a path argument** to be inside the server config parameter custom_local_disks_base_directory, which has no default, so set that also when using local disk.

