# EmbeddedRocksDB engine
The EmbeddedRocksDB engine in ClickHouse is a specialized engine that integrates RocksDB (a high-performance embedded key-value store from Facebook) directly into ClickHouse.  
It's designed for use cases requiring frequent single-row updates, point lookups, or transactional behavior, which are typically challenging for ClickHouse's native MergeTree-based engines.

## Key Features of EmbeddedRocksDB
### Optimized for Frequent Updates
Unlike ClickHouse's default engines (e.g., MergeTree), which struggle with row-level updates, EmbeddedRocksDB efficiently handles high-frequency updates and deletes thanks to RocksDB's LSM-tree design.

### Low-Latency Point Queries

RocksDB excels at random reads, making EmbeddedRocksDB a good choice for workloads requiring fast key-value lookups (e.g., user profiles, session data).

### Transactional Support (ACID)

Supports atomic writes (but not full multi-statement transactions like PostgreSQL).

Useful for ensuring consistency in write-heavy applications.

### Disk-Based with Compression

Data is stored on disk (not purely in-memory) but supports compression (e.g., LZ4, ZSTD).

More storage-efficient than purely in-memory engines like Memory.

### No Background Merges

Unlike ReplacingMergeTree or CollapsingMergeTree, EmbeddedRocksDB does not require background merges to resolve updates.

Updates are applied immediately.

## When to Use EmbeddedRocksDB
### ✅ Use Cases:

* Frequent single-row updates/deletes (e.g., counters, user preferences).
* Key-value workloads with high read/write ratios.
* Applications needing low-latency point queries.
* Scenarios where ReplacingMergeTree’s eventual consistency is unacceptable.

### ❌ When Not to Use It:

* Analytical queries (e.g., large scans, aggregations).
* Append-only bulk inserts (MergeTree is better).
* Extremely high-throughput writes (RocksDB has write amplification).

### Example Usage
#### 1. Create a Table
```sql
CREATE TABLE kv_store (
    key String,
    value String,
    timestamp DateTime
) ENGINE = EmbeddedRocksDB
PRIMARY KEY (key);
```
* PRIMARY KEY defines the lookup key (like a traditional key-value store).
* Supports arbitrary columns (not just key-value pairs).

#### 2. Insert/Update Data
```sql
INSERT INTO kv_store VALUES ('user:123', '{"name": "Alice", "age": 30}', now());
```
Updates are performed via INSERT (RocksDB handles upserts automatically).

#### 3. Point Query
```sql
SELECT value FROM kv_store WHERE key = 'user:123';
```
Fast lookup by primary key.

#### 4. Delete a Row
```sql
ALTER TABLE kv_store DELETE WHERE key = 'user:123';
```
(Yes, ClickHouse uses ALTER TABLE for deletes, even with RocksDB.)

## Performance Considerations
### Write Amplification

RocksDB’s LSM-tree design can cause write amplification under heavy workloads.

Tune RocksDB parameters (e.g., write_buffer_size, level0_slowdown_writes_trigger) if needed.

### Compaction Overhead

Background compaction can impact performance during heavy writes.

Monitor with system.rocksdb tables (e.g., system.rocksdb_stats).

### Memory Usage

RocksDB uses block cache (configurable via rocksdb_block_cache_size).

Default: 512MB. Increase for read-heavy workloads.

### No Native Replication

Unlike ReplicatedMergeTree, EmbeddedRocksDB does not support replication out-of-the-box.

Use external tools (e.g., Kafka) for replication.

## Comparison with Other Engines
Engine	            Best For	Update Frequency	Point Lookups	Analytics   
EmbeddedRocksDB	    Frequent updates, key-value	⭐⭐⭐⭐⭐	⭐⭐⭐⭐⭐	⭐  
ReplacingMergeTree	Eventually consistent updates	⭐⭐⭐	⭐⭐	⭐⭐⭐⭐⭐  
MergeTree	          Bulk inserts, analytics	⭐	⭐	⭐⭐⭐⭐⭐  
Memory	            Temporary data, caching	⭐⭐⭐⭐	⭐⭐⭐⭐⭐	⭐⭐  

### Advanced Configuration
Configure RocksDB settings in config.xml or per-table:
```xml
<rocksdb>
    <write_buffer_size>268435456</write_buffer_size> <!-- 256MB -->
    <level0_slowdown_writes_trigger>20</level0_slowdown_writes_trigger>
</rocksdb>
```
Run HTML
Or via SQL:
```sql
CREATE TABLE ... ENGINE = EmbeddedRocksDB
SETTINGS rocksdb_write_buffer_size = 268435456;
```
### Limitations
#### No Native Replication

* Requires external solutions for HA (e.g., Kafka + consumers).

#### Slower for Scans

* Not optimized for SELECT * FROM table (use MergeTree instead).

#### No SQL Joins

* Joins are still performed by ClickHouse (not RocksDB-native).

### Final Recommendation
#### Use EmbeddedRocksDB when:
* You need frequent updates/deletes with low latency.
* Your workload resembles a key-value store (not analytics).
* You can tolerate no native replication.

For mixed workloads, consider combining it with MergeTree tables (e.g., use EmbeddedRocksDB for real-time updates and MergeTree for analytics).

## Tuning EmbeddedRocksDB 
Requires balancing write performance, read latency, and resource usage. Below are key parameters and recommendations based on your workload type (write-heavy, read-heavy, or mixed).

### 1. Critical RocksDB Parameters
Configure these in config.xml or per-table via SETTINGS:

#### Write Performance (Throughput vs. Latency)
Parameter	Description	Recommended Value (Adjust Based on Workload)    
rocksdb_write_buffer_size	Size of memtable (per-table). Larger = fewer flushes but more memory.	64MB (default) → 256MB (write-heavy)   
rocksdb_max_write_buffer_number	Max memtables before stalling writes. Increase for bursty writes.	2 (default) → 4-6 (high write volume)   
rocksdb_level0_slowdown_writes_trigger	Delay writes when L0 files exceed this count. Lower = smoother latency.	20 (default) → 10 (latency-sensitive)   
rocksdb_level0_stop_writes_trigger	Stop writes when L0 files hit this count.	36 (default) → 24 (strict latency needs)   

#### Read Performance (Cache & Compression)
Parameter	Description	Recommended Value   
rocksdb_block_cache_size	Size of block cache for hot data. Increase for read-heavy workloads.	512MB (default) → 2-4GB (large datasets)   
rocksdb_use_adaptive_mutex	Reduces lock contention for high-concurrency reads.	false (default) → true (many readers)   
rocksdb_compression	Compression algorithm. Trade-off between CPU and storage.	LZ4 (default) → ZSTD (better ratio)    

#### Background Compaction (I/O Impact)  
Parameter	Description	Recommended Value  
rocksdb_max_background_compactions	Parallel compaction threads. Increase for SSDs/NVMe.	2 (default) → 4-8 (high-end storage)   
rocksdb_max_background_flushes	Parallel memtable flush threads.	1 (default) → 2-4 (write-heavy)   
rocksdb_bytes_per_sync	Sync data to disk incrementally (reduce write spikes).	1MB (default) → 4MB (spinning disks)   
 
### 2. Workload-Specific Tuning
#### A. Write-Heavy Workload (e.g., counters, metrics)
Goal: Maximize write throughput, tolerate slight read latency.
```xml
<rocksdb>
    <write_buffer_size>268435456</write_buffer_size>  <!-- 256MB memtable -->
    <max_write_buffer_number>6</max_write_buffer_number>
    <level0_slowdown_writes_trigger>16</level0_slowdown_writes_trigger>
    <level0_stop_writes_trigger>24</level0_stop_writes_trigger>
    <max_background_compactions>4</max_background_compactions>
    <compression>none</compression>                  <!-- Disable compression for CPU savings -->
</rocksdb>
```
Run HTML
#### B. Read-Heavy Workload (e.g., user profiles)
Goal: Optimize for low-latency point queries.
```xml
<rocksdb>
    <block_cache_size>2147483648</block_cache_size>   <!-- 2GB block cache -->
    <use_adaptive_mutex>true</use_adaptive_mutex>
    <compression>LZ4</compression>                   <!-- Fast decompression -->
    <optimize_filters_for_hits>true</optimize_filters_for_hits>  <!-- Reduce bloom filter overhead -->
</rocksdb>
```
Run HTML
#### C. Mixed Workload (Reads + Updates)
Goal: Balance between write and read performance.
```xml
<rocksdb>
    <write_buffer_size>134217728</write_buffer_size>  <!-- 128MB memtable -->
    <max_write_buffer_number>4</max_write_buffer_number>
    <block_cache_size>1073741824</block_cache_size>   <!-- 1GB cache -->
    <compression>ZSTD</compression>                  <!-- Good balance -->
    <max_background_compactions>4</max_background_compactions>
</rocksdb>
```
Run HTML

### 3. Monitoring & Diagnostics
Check RocksDB metrics via ClickHouse system tables:
```sql
-- Monitor compaction stats
SELECT * FROM system.rocksdb_stats WHERE metric LIKE '%compaction%';

-- Check memtable/block cache usage
SELECT * FROM system.rocksdb WHERE name LIKE '%memtable%' OR name LIKE '%cache%';
```

#### Key metrics to watch:

* Write stalls: High rocksdb.write_stall counts indicate compaction bottlenecks.
* Cache hit rate: Low rocksdb.block_cache_hit_rate suggests insufficient cache.
* Pending compaction bytes: High values (rocksdb.estimate_pending_compaction_bytes) mean I/O backlog.

### 4. Practical Example
Create a table with tuned settings:
```sql
CREATE TABLE user_sessions (
    session_id String,
    user_data String,
    last_updated DateTime
) ENGINE = EmbeddedRocksDB
PRIMARY KEY (session_id)
SETTINGS
    rocksdb_write_buffer_size = 134217728,
    rocksdb_block_cache_size = 2147483648,
    rocksdb_compression = 'ZSTD';
```

### 5. When to Avoid Tuning
* Default settings are fine for small datasets (<10GB) or low traffic.
* SSD/NVMe users: Focus on increasing max_background_compactions and write_buffer_size.
* Spinning disks: Prioritize reducing bytes_per_sync and compaction threads.

### Final Checklist
* Start with workload-specific presets (Section 2).
* Monitor system.rocksdb tables for bottlenecks.
* Adjust incrementally: Change one parameter at a time and observe impact.Test under load: Use realistic queries (e.g., INSERT/SELECT benchmarks).

## Specific scenario
For high-throughput workload (10K updates/minute on a 200GB/1B-row table), tuning EmbeddedRocksDB requires optimizing for write throughput while minimizing read latency impact.  
Below is a battle-tested configuration tailored to your scale.

### 1. Baseline Assumptions
* Workload: 10K updates/minute (~167 writes/sec), 200GB compressed data (1B rows).
* Storage: SSD/NVMe (spinning disks won’t handle this throughput).

#### Requirements:
* Sustain write bursts without stalls.
* Keep point queries (<10ms latency) efficient.
* Avoid excessive disk I/O from compaction.

### 2. Recommended Configuration
#### A. Table Creation with Tuned Settings
```sql
CREATE TABLE high_volume_updates (
    id UInt64,                -- Primary key (adjust type as needed)
    data String,              -- Your payload
    version UInt32,           -- For conflict resolution (optional)
    updated_at DateTime
) ENGINE = EmbeddedRocksDB
PRIMARY KEY (id)
SETTINGS
    rocksdb_write_buffer_size = 536870912,           -- 512MB memtable (absorbs bursts)
    rocksdb_max_write_buffer_number = 6,             -- Allow more in-memory buffering
    rocksdb_level0_slowdown_writes_trigger = 12,     -- Aggressive compaction
    rocksdb_level0_stop_writes_trigger = 20,         -- Prevent write stalls
    rocksdb_max_background_compactions = 8,          -- Leverage SSD parallelism
    rocksdb_max_background_flushes = 4,              -- Faster memtable flushes
    rocksdb_block_cache_size = 4294967296,           -- 4GB block cache (hot data)
    rocksdb_compression = 'ZSTD',                    -- Balance speed/ratio
    rocksdb_optimize_filters_for_hits = 1;           -- Skip bloom filters for hot data
```
#### B. Key Tuning Rationale
Parameter	Why This Value?  
write_buffer_size=512MB	Larger memtables reduce flush frequency (critical for 167 writes/sec).  
max_write_buffer_number=6	Prevents write stalls during bursts.  
level0_slowdown_writes_trigger=12	Triggers compaction earlier to smooth I/O.  
max_background_compactions=8	Parallelize compaction on SSDs (avoid backlog).  
block_cache_size=4GB	Cache ~1-2% of your 200GB data (adjust if reads target specific ranges).  
ZSTD compression	Saves space without excessive CPU (benchmark with your data).  

### 3. Operational Optimizations
#### A. Write Path
Batch Inserts: Group updates into batches of 100-1K rows (reduces RPC overhead).
```sql
INSERT INTO high_volume_updates VALUES (1, 'data1', 1, now()), (2, 'data2', 1, now()), ...;
```
* Avoid ALTER TABLE DELETE: Use tombstone markers if possible (deletes are expensive).

#### B. Read Path
* Cache Hot Keys: Ensure frequent queries use PRIMARY KEY (RocksDB excels at point lookups).
* Avoid Full Scans: EmbeddedRocksDB is not optimized for SELECT *.

#### C. Background Jobs
Schedule Compactions: During off-peak, force major compaction to reduce read amplification:
```sql
SYSTEM START ROCKSDB COMPACTION high_volume_updates;
```
* (Monitor with SELECT * FROM system.rocksdb_stats WHERE metric LIKE 'compaction%').

### 4. Monitoring & Alerts
#### Critical Metrics
```sql
-- Check write stalls (goal: 0)
SELECT value FROM system.rocksdb_stats 
WHERE metric = 'rocksdb.write_stall';

-- Block cache efficiency (goal: >95%)
SELECT value FROM system.rocksdb_stats 
WHERE metric = 'rocksdb.block_cache_hit_rate';

-- Pending compaction bytes (goal: <1GB)
SELECT value FROM system.rocksdb_stats 
WHERE metric = 'rocksdb.estimate_pending_compaction_bytes';
```
#### Alert Thresholds
* Write stalls: >5/minute → Increase max_background_compactions or reduce level0_stop_writes_trigger.
* Cache hit rate: <90% → Increase block_cache_size.
* Pending compaction: >5GB → Check disk I/O or throttle writes.

### 5. Scaling Beyond Single Node
Since EmbeddedRocksDB lacks native replication:

* Shard Data: Distribute keys across multiple tables/nodes (e.g., by key range).
* Use Kafka: Stream updates to multiple ClickHouse nodes (ensures consistency).
* Fallback to ReplicatedMergeTree: If consistency is critical, but expect higher latency.

### 6. Emergency Tuning
If writes still stall:
```sql
ALTER TABLE high_volume_updates 
MODIFY SETTING rocksdb_level0_slowdown_writes_trigger = 8;
```
### Final Checklist
* Deploy with recommended settings (Section 2).
* Benchmark: Simulate 10K updates/minute with clickhouse-benchmark.
* Monitor: Focus on write stalls and cache hits.
* Adjust: Incrementally tweak write_buffer_size and compaction threads.

### Next Steps
* Start with the Tuned Configuration: Implement the EmbeddedRocksDB settings early to validate performance.
* Benchmark Realistic Workloads: Use clickhouse-benchmark or custom scripts to simulate ~10K updates/minute.
* Monitor Key Metrics: Watch for write stalls, cache efficiency, and compaction backlog.
* Iterate Gradually: Adjust one parameter at a time (e.g., write_buffer_size or block_cache_size) based on test results.

