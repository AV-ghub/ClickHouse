[Using JOINs in ClickHouse](https://clickhouse.com/docs/en/guides/joining-tables)   

### [ClickHouse Playground](https://sql.clickhouse.com/)

#### part 1 [Join Types supported in ClickHouse](https://clickhouse.com/blog/clickhouse-fully-supports-joins-part1)
#### part 2 [ClickHouse Joins Under the Hood - Hash Join, Parallel Hash Join, Grace Hash Join](https://clickhouse.com/blog/clickhouse-fully-supports-joins-hash-joins-part2)
#### part 3 [ClickHouse Joins Under the Hood - Full Sorting Merge Join, Partial Merge Join - MergingSortedTransform](https://clickhouse.com/blog/clickhouse-fully-supports-joins-full-sort-partial-merge-part3)
#### part 4 [ClickHouse Joins Under the Hood - Direct Join](https://clickhouse.com/blog/clickhouse-fully-supports-joins-direct-join-part4)
#### part 5 [Choosing the Right Join Algorithm](https://clickhouse.com/blog/clickhouse-fully-supports-joins-how-to-choose-the-right-algorithm-part5)

#### Additional resources
[Schema Design](https://clickhouse.com/docs/en/data-modeling/schema-design)

[Dictionary](https://clickhouse.com/docs/en/dictionary)

[Join (SQL)](https://en.wikipedia.org/wiki/Join_(SQL))

[JOIN Clause](https://clickhouse.com/docs/en/sql-reference/statements/select/join#supported-types-of-join)

## Notes
The LEFT OUTER JOIN behaves like INNER JOIN; plus, for non-matching left table rows, ClickHouse returns **default values** for the right table’s columns.  
Note that ClickHouse can be configured to return NULLs instead of default values [join_use_nulls](https://clickhouse.com/docs/en/operations/settings/settings#join_use_nulls)   
> !!!-- **[Using Nullable](https://clickhouse.com/docs/en/sql-reference/data-types/nullable#storage-features) almost always negatively affects performance, keep this in mind when designing your databases** --!!!
