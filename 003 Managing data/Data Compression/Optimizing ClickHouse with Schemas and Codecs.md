## [Optimizing ClickHouse with Schemas and Codecs](https://clickhouse.com/blog/optimize-clickhouse-codecs-compression-schema)
### [Being strict on types](https://clickhouse.com/blog/optimize-clickhouse-codecs-compression-schema#being-strict-on-types)
Let's identify the ranges of these fields and reduce our schema accordingly to use the appropriate integer length based on their [supported ranges](https://clickhouse.com/docs/en/sql-reference/data-types/int-uint/#uint-ranges)   
```
Int64 -> Int16
Float64 -> Int16
etc
```
Before considering codecs, let's tidy up the String types. Our weather type can be represented by an **[Enum](https://clickhouse.com/docs/en/sql-reference/data-types/enum)**, reducing its representation from a variable string of N bytes to 8 bits per value. We can also dictionary encode our name and station_id columns using the **[LowCardinality](https://clickhouse.com/docs/en/sql-reference/data-types/lowcardinality)** super type.   

### [Specialized codecs](https://clickhouse.com/blog/optimize-clickhouse-codecs-compression-schema#specialized-codecs)
With **[Column Compression Codecs](https://clickhouse.com/docs/en/sql-reference/statements/create/table#column_compression_codec)**, however, we can change the algorithm (and its settings) used to encode and compress each column.   
In ClickHouse Cloud, we utilize the **ZSTD** compression algorithm (with a default value of 1) by default.    
**Delta** compression works well on slowly changing numerics by replacing two neighboring values with their difference (except for the first value which stays unchanged). This generates a smaller number which requires fewer bits for storage.    
**DoubleDelta** takes the 2nd derivative. This can be particularly effective when the first delta is large and constant, e.g., dates at periodic intervals.   
```
CREATE TABLE noaa_codec_v4
(
    `station_id` LowCardinality(String),
    `date` Date32 CODEC(Delta, ZSTD),
    `tempAvg` Int16 CODEC(Delta, ZSTD),
```
**T64**, when compressed with ZSTD tends to effectively reduce the size by around 25% on our largest integer fields.   
The smaller the maximum value of a block is, the better it compresses.    
This means T64 is effective when the true column values are very small compared to the range of the data type or if the column is sparsely populated, i.e., **only very few values are non-zero**.   

The most effective codec for each column:
```
`station_id` LowCardinality(String),
`date` Date32,
`tempAvg` Int16,
`tempMax` Int16,
`tempMin` Int16,
`precipitation` UInt16,
`snowfall` UInt16,
`snowDepth` UInt16,
`percentDailySun` UInt8,
`averageWindSpeed` UInt16,
`maxWindSpeed` UInt16,
`weatherType` Enum8,
`lat` Float32,
`lon` Float32,
`elevation` Int16,
`name` LowCardinality(String)

┌─name─────────────┬─best_codec──────────────────┬─compressed_size─┐
│ snowfall         │ CODEC(T64, ZSTD(1))         │ 28.35 MiB       │
│ tempMax          │ CODEC(T64, ZSTD(1))         │ 394.96 MiB      │
│ lat              │ DEFAULT                     │ 3.46 MiB        │
│ tempMin          │ CODEC(T64, ZSTD(1))         │ 382.42 MiB      │
│ date             │ CODEC(DoubleDelta, ZSTD(1)) │ 24.11 MiB       │
│ tempAvg          │ CODEC(T64, ZSTD(1))         │ 101.30 MiB      │
│ lon              │ DEFAULT                     │ 3.47 MiB        │
│ name             │ DEFAULT                     │ 4.20 MiB        │
│ location         │ DEFAULT                     │ 15.00 MiB       │
│ weatherType      │ DEFAULT                     │ 16.30 MiB       │
│ elevation        │ DEFAULT                     │ 1.84 MiB        │
│ station_id       │ DEFAULT                     │ 2.95 MiB        │
│ snowDepth        │ CODEC(ZSTD(1))              │ 41.74 MiB       │
│ precipitation    │ CODEC(T64, ZSTD(1))         │ 408.17 MiB      │
│ averageWindSpeed │ CODEC(T64, ZSTD(1))         │ 9.33 MiB        │
│ maxWindSpeed     │ CODEC(T64, ZSTD(1))         │ 6.14 MiB        │
│ percentDailySun  │ DEFAULT                     │ 1.46 MiB        │
└──────────────────┴─────────────────────────────┴─────────────────┘
```

