### [Compression in ClickHouse](https://clickhouse.com/docs/en/data-compression/compression-in-clickhouse)
#### Choosing the right column compression codec
With column compression codecs, we can change the algorithm (and its settings) used to encode and compress **each column**.  
Typically, encodings are applied first before compression is used.      
For MergeTree-engine family you can change the **default compression method** in the [compression](https://clickhouse.com/docs/en/operations/server-configuration-parameters/settings#compression) section of a server configuration.For MergeTree-engine family you can change the default compression method in the compression section of a server configuration.

To remove current CODEC from the column and use default compression from config.xml:
```
ALTER TABLE codec_example MODIFY COLUMN float_value CODEC(Default);
```
Codecs can be combined in a pipeline, for example
```
CODEC(Delta, Default).
```
#### [Specialized Codecs (Delta, DoubleDelta, GCD, Gorilla, FPC, T64)](https://clickhouse.com/docs/en/sql-reference/statements/create/table#specialized-codecs)
These codecs are designed to make compression more effective by **exploiting specific features of the data**.    
Some of these codecs do not compress data themselves, they instead preprocess the data such that **a second compression stage** using a general-purpose codec can achieve a higher data compression rate.
```
CREATE TABLE posts_v4
(
    `Id` Int32 CODEC(Delta, ZSTD),
    ...
    `ViewCount` UInt32 CODEC(Delta, ZSTD),
    ...
    `AnswerCount` UInt16 CODEC(Delta, ZSTD),
    ...
)
ENGINE = MergeTree
ORDER BY (PostTypeId, toDate(CreationDate), CommentCount)
```

### Additional resources
[Optimizing ClickHouse with Schemas and Codecs](https://clickhouse.com/blog/optimize-clickhouse-codecs-compression-schema)   
[Column Compression Codecs](https://clickhouse.com/docs/en/sql-reference/statements/create/table#column_compression_codec)    
[Gorilla: A Fast, Scalable, In-Memory Time Series Database](http://www.vldb.org/pvldb/vol8/p1816-teller.pdf)    
[High Throughput Compression of Double-Precision Floating-Point Data](https://userweb.cs.txstate.edu/~burtscher/papers/dcc07a.pdf)    
### Admin
[Compressed and uncompressed size of each column](https://github.com/AV-ghub/ClickHouse/blob/main/003%20Managing%20data/Advanced%20guides/Scripts.md#compressed-and-uncompressed-size-of-each-column)


