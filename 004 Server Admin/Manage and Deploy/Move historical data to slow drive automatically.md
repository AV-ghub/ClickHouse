## [Storage strategy in real life](https://medium.com/datadenys/scaling-clickhouse-using-amazon-s3-as-a-storage-94a9b9f2e6c7#6d10:~:text=Storage%20strategy%20in%20real%20life)

In production you might want to maintain Clickhouse performance, which is obviously poor when working with S3 storage. Common strategy is to use hot-cold storage approach, when recent part (hot) of your table is stored on high speed device (local SSD or NVMe) and historical part (cold) is stored on slow device (HDD or S3):


Great thing here is that Clickhouse manages this tiered storage stuff automatically. In order to use that, we just have to specify storage tiers in policies block:
```
<hot_cold>
  <volumes>
    <hot>
      <disk>default</disk>
    </hot>
    <cold>
      <disk>s3</disk>
    </cold>
  </volumes>
  <move_factor>0.1</move_factor>
</hot_cold>
This will ask Clickhouse to save table data on a local disk first (hot block). When available local disk space reaches move_factor*local_disk_size table data will be automatically relocated to S3 storage (cold block). We’ve used 10% movement factor which will result in data relocation once there’s less than 10% of free space left on local device.
```

Now (after Clickhouse server restart) we can create tables with this policy:
```
CREATE TABLE hot_col_table ( `id` UInt32, ... )
ENGINE = MergeTree ORDER BY id
SETTINGS storage_policy = 'hot_cold'
```
And we just use it in a standard way, Clickhouse will manage storage stuff automatically.
