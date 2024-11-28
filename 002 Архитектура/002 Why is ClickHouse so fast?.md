[Why is ClickHouse so fast?](https://chistadata.com/why-clickhouse-is-so-fast/)

When an analytical query is executing, the execution can be logically broken down into two steps:

* Step 1: Load data from disk into RAM
* Step 2: Process data in CPU(s) as per query execution plan

ClickHouse has been optimized for extreme performance on both these fronts.

### Vectorized Query Execution
Instead of processing data value by value, ClickHouse processes data in large arrays, which could be **the entire length of the column loaded into RAM**.   
Vectorized query execution reduces CPU cache miss rates, utilizes the **SIMD** (single instruction multiple data) capabilities of modern CPUs,    
reduces branching overheads and minimizes data movements, which saves precious milliseconds of query execution time, reducing latency.

### Massively Parallel & Scalable Processing 
Not only can a single large query be split into the various CPU cores of a single node,    
but a **single query** can also **leverage all the other CPU cores and disks corresponding to all the other shards** in the cluster.

In addition to this, ClickHouse supports multiple types of caches:   
* from data cache types (read cache, write cache, compression cache, dictionary cache)
* to query cache,
* to filesystem cache, 

all of which leverage the power of recent query executions to reduce future query runtimes.  

### Attention to Low-level Details
* ClickHouse chooses one of 30+ hash table data structure implementations for every GROUP BY execution, based on the use case.
* Support for massive concurrency: each node with the right configurations can individually support 100s of users.

