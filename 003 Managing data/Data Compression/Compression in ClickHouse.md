### [Compression in ClickHouse](https://clickhouse.com/docs/en/data-compression/compression-in-clickhouse)
#### Choosing the right column compression codec
With column compression codecs, we can change the algorithm (and its settings) used to encode and compress **each column**.  
Typically, encodings are applied first before compression is used.      
For MergeTree-engine family you can change the **default compression method** in the [compression](https://clickhouse.com/docs/en/operations/server-configuration-parameters/settings#compression) section of a server configuration.For MergeTree-engine family you can change the default compression method in the compression section of a server configuration.


### Additional resources
[Optimizing ClickHouse with Schemas and Codecs](https://clickhouse.com/blog/optimize-clickhouse-codecs-compression-schema)   
[Column Compression Codecs](https://clickhouse.com/docs/en/sql-reference/statements/create/table#column_compression_codec)
