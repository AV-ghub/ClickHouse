# Handling Correlated Subqueries in ClickHouse

The error you're encountering is because ClickHouse doesn't support correlated subqueries in the same way PostgreSQL does. The subquery can't reference columns from the outer query (`c.complexcallid` in this case) unless they're constants or CTEs.

## Solutions

### Option 1: Use a JOIN instead

```sql
SELECT 
    c.callid as callid,
    multiIf(cc.count > 0, 1, 0) AS iscomplex
FROM dbo_calls c
LEFT JOIN (
    SELECT complexcallid, count() as count 
    FROM dbo_complexcall 
    GROUP BY complexcallid
) cc ON cc.complexcallid = c.complexcallid
WHERE ...
```

### Option 2: Use ARRAY JOIN with a subquery (for more complex cases)

```sql
SELECT 
    c.callid as callid,
    multiIf(cc_count > 0, 1, 0) AS iscomplex
FROM dbo_calls c
LEFT JOIN (
    SELECT complexcallid, count() as cc_count
    FROM dbo_complexcall
    GROUP BY complexcallid
) USING (complexcallid)
WHERE ...
```

### Option 3: Use the `IN` operator if you just need existence check

```sql
SELECT 
    c.callid as callid,
    multiIf(c.complexcallid IN (
        SELECT complexcallid FROM dbo_complexcall
    ), 1, 0) AS iscomplex
FROM dbo_calls c
WHERE ...
```

## Performance Considerations

1. The JOIN approach (Option 1) is generally the most performant in ClickHouse
2. Make sure you have proper indexes on the join columns (`complexcallid` in both tables)
3. If the `dbo_complexcall` table is large, consider materializing the counts in advance

Given that `dbo.complexcall` has unique `complexcallid` values and you only need to check existence (not count), you have several optimized options in ClickHouse:

### Option 1: Use `IN` (most straightforward translation)
```sql
SELECT DISTINCT
    c.CallID as CallID,
    c.ComplexCallId IN (SELECT complexcallid FROM dbo_complexcall) AS IsComplex
FROM dbo_Calls AS c
```

### Option 2: Use `LEFT JOIN` with `isNotZero` (most performant)
```sql
SELECT DISTINCT
    c.CallID as CallID,
    isNotZero(cc.complexcallid) AS IsComplex
FROM dbo_Calls AS c
LEFT JOIN dbo_complexcall cc ON cc.complexcallid = c.ComplexCallId
```

### Option 3: Use `ANY` operator (alternative approach)
```sql
SELECT DISTINCT
    c.CallID as CallID,
    hasAny(array(c.ComplexCallId), 
           (SELECT groupArray(complexcallid) FROM dbo_complexcall)) AS IsComplex
FROM dbo_Calls AS c
```

## Performance Recommendations:

1. **Best choice**: Option 2 (LEFT JOIN) will be most efficient since:
   - ClickHouse is optimized for joins
   - No subquery execution per row
   - Directly uses the uniqueness of complexcallid

2. If `dbo_complexcall` is very large, consider:
   ```sql
   SELECT DISTINCT
       c.CallID as CallID,
       c.ComplexCallId IN (
           SELECT complexcallid FROM dbo_complexcall SETTINGS optimize_move_to_prewhere = 0
       ) AS IsComplex
   FROM dbo_Calls AS c
   ```

3. If you're using ReplacingMergeTree or other engines with duplicates, add `FINAL` modifier:
   ```sql
   FROM dbo_Calls FINAL AS c
   ```

With your table sizes (~500M rows in `calls` and ~50M in `complexcall`), here are the key nuances and optimizations for Option 2 (LEFT JOIN approach):

### Critical Nuances for Large Tables:

1. **Memory Usage**:
   - The join will need to build a hash table in memory for the right table (`complexcall`)
   - At ~50M rows, this will require several GBs of RAM (plan for ~2-4GB)

2. **Join Algorithm Selection**:
   ClickHouse will likely use:
   - **Hash Join** (default for this size)
   - Might switch to **Merge Join** if memory constraints hit

### Recommended Optimizations:

```sql
SELECT DISTINCT
    c.CallID as CallID,
    isNotZero(cc.complexcallid) AS IsComplex
FROM dbo_Calls AS c
LEFT JOIN dbo_complexcall cc ON cc.complexcallid = c.ComplexCallId
SETTINGS 
    join_algorithm = 'hash',  -- Force hash join
    max_rows_in_join = 100000000,  -- Increase if needed
    max_bytes_in_join = 20000000000,  -- ~20GB
    join_use_nulls = 0  -- Slightly more efficient
```

### Alternative Approaches for Better Performance:

1. **Pre-filter Complex Calls First**:
```sql
WITH complex_ids AS (
    SELECT complexcallid FROM dbo_complexcall
)
SELECT 
    c.CallID,
    isNotZero(c.ComplexCallId IN complex_ids) AS IsComplex
FROM dbo_Calls c
```

2. **Use Dictionary if Frequently Queried**:
```sql
CREATE DICTIONARY complexcall_dict
( complexcallid UInt64 )
PRIMARY KEY complexcallid
SOURCE(CLICKHOUSE(TABLE 'dbo_complexcall'))
LIFETIME(MIN 300 MAX 360)
LAYOUT(HASHED());

-- Then query:
SELECT 
    CallID,
    dictHas('complexcall_dict', ComplexCallId) AS IsComplex
FROM dbo_Calls
```

### Hardware Considerations:
- Ensure enough RAM for the hash table (50M entries × ~40 bytes = ~2GB)
- SSD storage highly recommended for tables this size
- Consider partitioning your tables if not already done

# Partitioning and Sharding Strategies for Your ClickHouse Tables

Given your table sizes (~500M rows in `calls` and ~50M in `complexcall`), here are optimized partitioning and sharding approaches:

## Partitioning Recommendations

### For `dbo_calls` (500M rows):

```sql
CREATE TABLE dbo_calls (
    CallID UInt64,
    ComplexCallId UInt64,
    -- other columns
    CallDate DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(CallDate)  -- Time-based partitioning
ORDER BY (ComplexCallId, CallID)  -- Optimized for your join query
SETTINGS index_granularity = 8192;
```

**Why this works**:
- Time-based partitioning helps with:
  - Data expiration (TTL)
  - Query pruning for time-range queries
- Ordering by `ComplexCallId` first optimizes your join queries

### For `dbo_complexcall` (50M rows):

```sql
CREATE TABLE dbo_complexcall (
    complexcallid UInt64,
    -- other columns
    creation_date Date
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(creation_date)  -- Or consider PARTITION BY tuple() for single partition
ORDER BY (complexcallid)  -- Single column as it's unique
SETTINGS index_granularity = 8192;
```

## Sharding Strategies

### Option 1: Distributed Table Approach

```sql
-- On each shard:
CREATE TABLE dbo_calls_local (...) ENGINE = MergeTree()...;

-- Distributed table:
CREATE TABLE dbo_calls AS dbo_calls_local
ENGINE = Distributed(
    cluster_name,  -- Your cluster name
    currentDatabase(),
    dbo_calls_local,
    rand()  -- Or use ComplexCallId for co-located joins
);
```

### Option 2: Join-Aware Sharding (Best for your use case)

```sql
CREATE TABLE dbo_calls_local (...) 
ENGINE = MergeTree()...
ORDER BY (ComplexCallId, CallID);

CREATE TABLE dbo_complexcall_local (...)
ENGINE = MergeTree()...
ORDER BY (complexcallid);

-- Distributed tables with co-location:
CREATE TABLE dbo_calls AS dbo_calls_local
ENGINE = Distributed(
    cluster_name,
    currentDatabase(),
    dbo_calls_local,
    ComplexCallId  -- Same sharding key as complexcall
);

CREATE TABLE dbo_complexcall AS dbo_complexcall_local
ENGINE = Distributed(
    cluster_name,
    currentDatabase(),
    dbo_complexcall_local,
    complexcallid  -- Same sharding key
);
```

## Optimized Query for Partitioned/Sharded Environment

```sql
SELECT 
    c.CallID,
    isNotZero(cc.complexcallid) AS IsComplex
FROM dbo_calls c
LEFT JOIN dbo_complexcall cc ON cc.complexcallid = c.ComplexCallId
WHERE c.CallDate BETWEEN '2023-01-01' AND '2023-12-31'  -- Partition pruning
SETTINGS 
    distributed_perfect_shard = 1,  -- Enable for co-located joins
    join_algorithm = 'parallel_hash';
```

## Additional Recommendations:

1. **TTL for Data Rotation**:
```sql
ALTER TABLE dbo_calls MODIFY TTL CallDate + INTERVAL 1 YEAR;
```

2. **Materialized Views for Hot Data**:
```sql
CREATE MATERIALIZED VIEW dbo_calls_recent
ENGINE = MergeTree()...
PARTITION BY toYYYYMM(CallDate)
ORDER BY (ComplexCallId)
AS SELECT * FROM dbo_calls
WHERE CallDate > now() - INTERVAL 3 MONTH;
```

3. **Monitoring Partitions**:
```sql
SELECT 
    partition,
    rows,
    bytes_on_disk
FROM system.parts
WHERE table = 'dbo_calls';
```

# Concrete ClickHouse Configuration Examples

Here are complete, production-ready configurations for your scenario with 500M+ rows:

## 1. Single Node Configuration (Non-Distributed)

**/etc/clickhouse-server/config.d/tables.xml**:
```xml
<yandex>
    <profiles>
        <default>
            <max_memory_usage>20000000000</max_memory_usage> <!-- 20GB -->
            <max_bytes_before_external_group_by>10000000000</max_bytes_before_external_group_by>
            <max_threads>16</max_threads>
        </default>
    </profiles>

    <storage_configuration>
        <disks>
            <fast_ssd>
                <path>/mnt/clickhouse/fast_ssd/</path>
            </fast_ssd>
        </disks>
        <policies>
            <hot_cold>
                <volumes>
                    <hot>
                        <disk>fast_ssd</disk>
                    </hot>
                </volumes>
            </hot_cold>
        </policies>
    </storage_configuration>
</yandex>
```

**Table DDLs**:
```sql
CREATE TABLE dbo_calls
(
    CallID UInt64,
    ComplexCallId UInt64,
    CallDate DateTime,
    -- other columns
    INDEX idx_complex ComplexCallId TYPE bloom_filter GRANULARITY 3
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(CallDate)
ORDER BY (ComplexCallId, CallID)
TTL CallDate + INTERVAL 2 YEAR
SETTINGS 
    storage_policy = 'hot_cold',
    min_bytes_for_wide_part = 1073741824, -- 1GB
    min_rows_for_wide_part = 10000000;

CREATE TABLE dbo_complexcall
(
    complexcallid UInt64,
    creation_date Date,
    -- other columns
    INDEX idx_id complexcallid TYPE bloom_filter GRANULARITY 1
)
ENGINE = MergeTree()
ORDER BY complexcallid
SETTINGS 
    storage_policy = 'hot_cold',
    min_bytes_for_wide_part = 1073741824;
```

## 2. Distributed Cluster Configuration (3 shards, 2 replicas)

**/etc/clickhouse-server/config.xml** (on each node):
```xml
<remote_servers>
    <production_cluster>
        <shard>
            <weight>1</weight>
            <replica>
                <host>node1</host>
                <port>9000</port>
            </replica>
            <replica>
                <host>node2</host>
                <port>9000</port>
            </replica>
        </shard>
        <shard>
            <weight>1</weight>
            <replica>
                <host>node3</host>
                <port>9000</port>
            </replica>
            <replica>
                <host>node4</host>
                <port>9000</port>
            </replica>
        </shard>
        <shard>
            <weight>1</weight>
            <replica>
                <host>node5</host>
                <port>9000</port>
            </replica>
            <replica>
                <host>node6</host>
                <port>9000</port>
            </replica>
        </shard>
    </production_cluster>
</remote_servers>
```

**Distributed Tables DDL** (execute on all nodes):
```sql
-- Local tables (same as single node config)
CREATE TABLE dbo_calls_local (...) ENGINE = MergeTree()...;
CREATE TABLE dbo_complexcall_local (...) ENGINE = MergeTree()...;

-- Distributed tables
CREATE TABLE dbo_calls
ENGINE = Distributed(
    'production_cluster',
    currentDatabase(),
    'dbo_calls_local',
    xxHash64(ComplexCallId) % 3  -- Even distribution
);

CREATE TABLE dbo_complexcall
ENGINE = Distributed(
    'production_cluster',
    currentDatabase(),
    'dbo_complexcall_local',
    xxHash64(complexcallid) % 3  -- Same distribution key
);
```

## 3. Optimized Query Templates

**Basic Join Query**:
```sql
SELECT 
    c.CallID,
    isNotZero(cc.complexcallid) AS IsComplex
FROM dbo_calls c
LEFT JOIN dbo_complexcall cc 
    ON cc.complexcallid = c.ComplexCallId
WHERE c.CallDate BETWEEN '2023-01-01' AND '2023-12-31'
SETTINGS 
    join_algorithm = 'parallel_hash',
    max_bytes_in_join = 20000000000,
    join_use_nulls = 0,
    distributed_group_by_no_merge = 1;
```

**Materialized View for Fast Existence Checks**:
```sql
CREATE MATERIALIZED VIEW dbo_complexcall_exists
ENGINE = Join(ANY, LEFT, complexcallid)
POPULATE AS
SELECT complexcallid FROM dbo_complexcall;

-- Then query:
SELECT 
    c.CallID,
    joinGet('dbo_complexcall_exists', 'complexcallid', c.ComplexCallId) IS NOT NULL AS IsComplex
FROM dbo_calls c;
```

## Monitoring Commands

**Check Partition Distribution**:
```sql
SELECT 
    partition,
    sum(rows) AS rows,
    formatReadableSize(sum(bytes_on_disk)) AS size,
    min(min_date) AS min_date,
    max(max_date) AS max_date
FROM clusterAllReplicas('production_cluster', system.parts)
WHERE table = 'dbo_calls_local'
GROUP BY partition
ORDER BY partition;
```

**Node Resource Usage**:
```sql
SELECT 
    hostName() AS host,
    formatReadableSize(memory_usage) AS memory,
    running_queries
FROM system.metrics
WHERE metric IN ('MemoryUsage', 'RunningQueries');
```

# Advanced ClickHouse Configuration Guide

## 1. ZooKeeper/Keeper Configuration for Cluster Coordination

### Minimal Production ZooKeeper Configuration (zoo.cfg)

```properties
# Core settings
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/var/lib/zookeeper
clientPort=2181
maxClientCnxns=60
autopurge.snapRetainCount=5
autopurge.purgeInterval=24

# Cluster nodes (3-node ensemble)
server.1=zk1:2888:3888
server.2=zk2:2888:3888
server.3=zk3:2888:3888
```

### ClickHouse Keeper (Modern Alternative) Configuration

**/etc/clickhouse-server/config.d/keeper.xml**:
```xml
<keeper_server>
    <tcp_port>9181</tcp_port>
    <server_id>1</server_id>
    
    <coordination_settings>
        <operation_timeout_ms>10000</operation_timeout_ms>
        <session_timeout_ms>30000</session_timeout_ms>
        <raft_logs_level>warning</raft_logs_level>
    </coordination_settings>

    <raft_configuration>
        <server>
            <id>1</id>
            <hostname>keeper1</hostname>
            <port>9234</port>
        </server>
        <server>
            <id>2</id>
            <hostname>keeper2</hostname>
            <port>9234</port>
        </server>
        <server>
            <id>3</id>
            <hostname>keeper3</hostname>
            <port>9234</port>
        </server>
    </raft_configuration>
    
    <log_storage_path>/var/lib/clickhouse/coordination/log</log_storage_path>
    <snapshot_storage_path>/var/lib/clickhouse/coordination/snapshots</snapshot_storage_path>
</keeper_server>
```

### ClickHouse Server Integration

**/etc/clickhouse-server/config.d/zookeeper.xml**:
```xml
<yandex>
    <zookeeper>
        <node index="1">
            <host>zk1</host>
            <port>2181</port>
        </node>
        <node index="2">
            <host>zk2</host>
            <port>2181</port>
        </node>
        <node index="3">
            <host>zk3</host>
            <port>2181</port>
        </node>
        <session_timeout_ms>30000</session_timeout_ms>
        <operation_timeout_ms>10000</operation_timeout_ms>
    </zookeeper>
    
    <distributed_ddl>
        <path>/clickhouse/ddl</path>
        <profile>default</profile>
    </distributed_ddl>
</yandex>
```

## 2. Backup Strategies for Large-Scale Deployment

### Full Backup with clickhouse-backup

**/etc/clickhouse-backup/config.yml**:
```yaml
general:
  remote_storage: s3
  disable_progress_bar: false
  backups_to_keep_local: 3
  backups_to_keep_remote: 10

clickhouse:
  username: backup_user
  password: "secure_password"
  host: localhost
  port: 9000
  data_path: "/var/lib/clickhouse"
  skip_tables:
    - system.*
    - INFORMATION_SCHEMA.*
    - information_schema.*

s3:
  access_key: "AWS_ACCESS_KEY"
  secret_key: "AWS_SECRET_KEY"
  bucket: "clickhouse-backups"
  region: "us-east-1"
  path: "backups/"
  endpoint: ""
  compression_level: 1
  compression_format: tar
  sse: AES256
  disable_ssl: false
  part_size: 1073741824
```

### Incremental Backup Strategy

```bash
# Daily incremental
0 2 * * * /usr/bin/clickhouse-backup create incremental_$(date +\%Y\%m\%d)

# Weekly full backup
0 3 * * 0 /usr/bin/clickhouse-backup create full_$(date +\%Y\%m\%d)

# Monthly archive
0 4 1 * * /usr/bin/clickhouse-backup upload full_$(date +\%Y\%m01)
```

### Table-Specific Backup Commands

```sql
-- Freeze partitions (atomic copy)
ALTER TABLE dbo_calls FREEZE PARTITION '202301';

-- Backup to S3 directly
BACKUP TABLE dbo_calls TO S3(
    'https://storage.yandexcloud.net/backups/clickhouse',
    'AWS_ACCESS_KEY_ID', 
    'AWS_SECRET_ACCESS_KEY'
) SETTINGS compression_method='zstd', compression_level=3;
```

## 3. Advanced Query Profiling Techniques

### Runtime Profiling Configuration

**/etc/clickhouse-server/users.d/profiling.xml**:
```xml
<yandex>
    <profiles>
        <default>
            <log_queries>1</log_queries>
            <query_profiler_real_time_period_ns>10000000</query_profiler_real_time_period_ns>
            <query_profiler_cpu_time_period_ns>10000000</query_profiler_cpu_time_period_ns>
            <allow_introspection_functions>1</allow_introspection_functions>
            <max_query_size>1073741824</max_query_size>
        </default>
    </profiles>
</yandex>
```

### Query Analysis Commands

**Detailed Query Inspection**:
```sql
-- Get query pipeline
EXPLAIN PIPELINE
SELECT c.CallID, isNotZero(cc.complexcallid) AS IsComplex
FROM dbo_calls c LEFT JOIN dbo_complexcall cc ON cc.complexcallid = c.ComplexCallId;

-- Analyze JOIN performance
SET allow_experimental_analyzer = 1;
EXPLAIN JOIN(ALL, LEFT) 
SELECT count() FROM dbo_calls c LEFT JOIN dbo_complexcall cc ON cc.complexcallid = c.ComplexCallId;

-- Profile CPU usage
SELECT 
    query_id,
    ProfileEvents['OSCPUVirtualTimeMicroseconds'] AS cpu_microseconds,
    ProfileEvents['RealTimeMicroseconds'] AS realtime_microseconds
FROM system.query_log
WHERE event_date = today()
ORDER BY cpu_microseconds DESC
LIMIT 10;
```

### Performance Metrics Collection

```sql
-- Live query monitoring
SELECT 
    elapsed,
    query,
    formatReadableSize(memory_usage) AS memory,
    threads,
    ProfileEvents['DiskReadElapsedMicroseconds'] AS disk_read_us,
    ProfileEvents['NetworkSendElapsedMicroseconds'] AS network_send_us
FROM system.processes
ORDER BY elapsed DESC;

-- Historical analysis
CREATE TABLE system.query_log_mv
ENGINE = MergeTree()
ORDER BY (event_date, query_start_time)
AS SELECT * FROM system.query_log
SETTINGS storage_policy = 'hot_cold';
```

### FlameGraph Generation

```bash
# Collect stack traces
clickhouse-client --query="
    SELECT arrayStringConcat(arrayMap(
        x -> demangle(addressToSymbol(x)), 
        trace
    ), ';') AS stack 
    FROM system.trace_log 
    WHERE event_date >= today() - 1 
    LIMIT 1000000" > stacks.txt

# Generate FlameGraph
./stackcollapse.pl < stacks.txt | ./flamegraph.pl > profile.svg
```

## Maintenance Operations

```sql
-- Optimize tables periodically
OPTIMIZE TABLE dbo_calls FINAL;

-- Cleanup old parts
ALTER TABLE dbo_calls CLEANUP;

-- Monitor replication
SELECT 
    database,
    table,
    is_leader,
    absolute_delay,
    replica_path
FROM system.replicas
WHERE is_session_expired = 1;
```

# Advanced ClickHouse Security and Monitoring Configurations

## 1. TLS/SSL Setup for Secure Cluster Communication

### SSL Certificate Generation (for all nodes)

```bash
# Generate CA (run once)
openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 -nodes \
  -keyout ca-key.pem -out ca-cert.pem -subj "/CN=ClickHouse CA"

# Generate node certificates (run per node)
openssl req -newkey rsa:2048 -nodes -keyout node-key.pem \
  -out node.csr -subj "/CN=$(hostname)"

openssl x509 -req -CA ca-cert.pem -CAkey ca-key.pem -in node.csr \
  -out node-cert.pem -days 365 -CAcreateserial -extfile <(
    echo "subjectAltName=DNS:$(hostname),IP:$(hostname -I | awk '{print $1}')"
```

### ClickHouse SSL Configuration

**/etc/clickhouse-server/config.d/ssl.xml**:
```xml
<yandex>
    <openSSL>
        <server>
            <certificateFile>/etc/clickhouse-server/ssl/node-cert.pem</certificateFile>
            <privateKeyFile>/etc/clickhouse-server/ssl/node-key.pem</privateKeyFile>
            <caConfig>/etc/clickhouse-server/ssl/ca-cert.pem</caConfig>
            <verificationMode>none</verificationMode>
            <loadDefaultCAFile>true</loadDefaultCAFile>
            <cacheSessions>true</cacheSessions>
            <disableProtocols>sslv2,sslv3</disableProtocols>
            <preferServerCiphers>true</preferServerCiphers>
        </server>
        <client>
            <caConfig>/etc/clickhouse-server/ssl/ca-cert.pem</caConfig>
            <certificateFile>/etc/clickhouse-server/ssl/node-cert.pem</certificateFile>
            <privateKeyFile>/etc/clickhouse-server/ssl/node-key.pem</privateKeyFile>
        </client>
    </openSSL>

    <https_port>8443</https_port>
    <tcp_port_secure>9440</tcp_port_secure>
    
    <interserver_http_port>9010</interserver_http_port>
    <interserver_https_port>9011</interserver_https_port>
    <interserver_https_host>$(hostname)</interserver_https_host>
</yandex>
```

### Distributed Table with SSL

```xml
<remote_servers>
    <secure_cluster>
        <shard>
            <replica>
                <host>node1</host>
                <port>9440</port>
                <secure>1</secure>
                <user>default</user>
                <password>password</password>
            </replica>
        </shard>
    </secure_cluster>
</remote_servers>
```

## 2. Advanced Resource Isolation Using cgroups v2

### Systemd Service Unit Override

**/etc/systemd/system/clickhouse-server.service.d/cgroup.conf**:
```ini
[Service]
CPUAccounting=yes
MemoryAccounting=yes
IOAccounting=yes
MemoryHigh=90%
MemoryMax=95%
CPUWeight=100
IOWeight=100
Slice=clickhouse.slice
Delegate=yes
```

### ClickHouse cgroups Configuration

**/etc/clickhouse-server/config.d/cgroups.xml**:
```xml
<yandex>
    <cgroups>
        <path>/sys/fs/cgroup/clickhouse/</path>
        <cpu>
            <shares>1024</shares>
            <period>100000</period>
            <quota>80000</quota>
        </cpu>
        <memory>
            <limit_in_bytes>80%</limit_in_bytes>
            <soft_limit_in_bytes>70%</soft_limit_in_bytes>
        </memory>
        <io>
            <weight>500</weight>
        </io>
    </cgroups>

    <max_server_memory_usage_to_ram_ratio>0.9</max_server_memory_usage_to_ram_ratio>
    <max_concurrent_queries>100</max_concurrent_queries>
    <max_thread_pool_size>10000</max_thread_pool_size>
</yandex>
```

### Monitoring cgroups

```bash
# Check CPU usage
cat /sys/fs/cgroup/clickhouse/cpu.stat

# Check memory
cat /sys/fs/cgroup/clickhouse/memory.current
cat /sys/fs/cgroup/clickhouse/memory.high

# Check IO
cat /sys/fs/cgroup/clickhouse/io.stat
```

## 3. Prometheus + Grafana Integration

### ClickHouse Metrics Exporter

**/etc/clickhouse-server/config.d/prometheus.xml**:
```xml
<yandex>
    <prometheus>
        <endpoint>/metrics</endpoint>
        <port>9363</port>
        <metrics>true</metrics>
        <events>true</events>
        <asynchronous_metrics>true</asynchronous_metrics>
        <status_info>true</status_info>
        <collect_process_metrics>true</collect_process_metrics>
        <send_events>true</send_events>
    </prometheus>
</yandex>
```

### Prometheus Configuration

**/etc/prometheus/prometheus.yml**:
```yaml
scrape_configs:
  - job_name: 'clickhouse'
    scrape_interval: 15s
    static_configs:
      - targets: ['node1:9363', 'node2:9363', 'node3:9363']
    metrics_path: '/metrics'
    
  - job_name: 'clickhouse-system'
    scrape_interval: 30s
    static_configs:
      - targets: ['node1:9100', 'node2:9100', 'node3:9100']
```

### Grafana Dashboard (JSON Snippet)

```json
{
  "panels": [{
    "title": "Query Throughput",
    "type": "graph",
    "targets": [{
      "expr": "rate(clickhouse_query_started_total[1m])",
      "legendFormat": "Queries/sec"
    }]
  }, {
    "title": "Memory Usage",
    "type": "gauge",
    "targets": [{
      "expr": "clickhouse_memory_usage / clickhouse_max_server_memory_usage",
      "format": "percentunit"
    }]
  }],
  "templating": {
    "list": [{
      "name": "host",
      "query": "label_values(clickhouse_metrics, instance)"
    }]
  }
}
```

### Alert Rules

**/etc/prometheus/rules/clickhouse.rules.yml**:
```yaml
groups:
- name: clickhouse-alerts
  rules:
  - alert: HighMemoryUsage
    expr: clickhouse_memory_usage / clickhouse_max_server_memory_usage > 0.9
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "High memory usage on {{ $labels.instance }}"
      
  - alert: ReplicasDown
    expr: up{job="clickhouse"} == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "ClickHouse replica down: {{ $labels.instance }}"
```

## 4. Advanced Network Tuning

**/etc/sysctl.d/99-clickhouse.conf**:
```conf
# Increase TCP buffer sizes
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# Connection tracking
net.netfilter.nf_conntrack_max = 1000000
net.ipv4.tcp_max_syn_backlog = 30000
net.core.somaxconn = 32768

# TIME-WAIT optimization
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 30
```

## 5. Kernel Tuning for High Throughput

**/etc/security/limits.d/clickhouse.conf**:
```conf
clickhouse soft nofile 262144
clickhouse hard nofile 524288
clickhouse soft memlock unlimited
clickhouse hard memlock unlimited
clickhouse soft nproc 131072
clickhouse hard nproc 262144
```
