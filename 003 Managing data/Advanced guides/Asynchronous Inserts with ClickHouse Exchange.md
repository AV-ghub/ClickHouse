# [Asynchronous Inserts with ClickHouse Exchange](https://clickhouse.com/blog/asynchronous-data-inserts-in-clickhouse)
[Asynchronous Inserts (async_insert)](https://clickhouse.com/docs/optimize/asynchronous-inserts)

When you enable asynchronous inserts, ClickHouse buffers the data in memory and periodically flushes it to the target table.  
This is controlled by settings like **async_insert** and **wait_for_async_insert**.

## The EXCHANGE Operation
The EXCHANGE operation in ClickHouse allows you to atomically swap the contents of two tables.  
This is useful in scenarios where you want to switch between two tables without downtime or data loss.

### Scenario
You have two tables that are continuously inserting data, and you periodically switch them using the EXCHANGE operation.  
You want to use asynchronous inserts for these tables, but you're concerned about what happens to the buffered data when the EXCHANGE operation is fired.

### What Happens to Buffered Data During EXCHANGE?
When you perform an EXCHANGE operation, ClickHouse swaps the metadata of the two tables.  
However, any data that is still in the asynchronous insert buffer (not yet flushed to disk) is not affected by the EXCHANGE operation.  
This means:   
* Buffered Data: Data that is still in the asynchronous insert buffer will remain in the buffer and will be flushed to the original table (the one that was the target before the EXCHANGE).
* Post-Exchange Inserts: After the EXCHANGE, new asynchronous inserts will go to the new target table (the one that was swapped in).

### Potential Issues
* Data Consistency: If you rely on the EXCHANGE operation to switch tables and expect all data to be immediately available in the new table, you might face issues because the buffered data will still be flushed to the old table.
* Data Loss: If the old table is dropped or modified after the EXCHANGE, you could lose the buffered data that hasn't been flushed yet.

## Recommendations
Flush Buffers Before Exchange: Before performing the EXCHANGE operation, you can manually flush the asynchronous insert buffers to ensure all data is written to the tables.  
This can be done by disabling asynchronous inserts temporarily or using the SYSTEM FLUSH ASYNC INSERT QUEUE	SYSTEM FLUSH LOGS command.

Avoid Asynchronous Inserts for Critical Swaps: If the EXCHANGE operation is critical and you cannot afford any delay or inconsistency, consider not using asynchronous inserts for these tables.

Monitor and Handle Buffered Data: Implement a mechanism to monitor the asynchronous insert buffers and handle any remaining data after the EXCHANGE operation.

Conclusion
While it is technically possible to use asynchronous inserts with tables that are periodically switched using the EXCHANGE operation, you need to be cautious about the buffered data.  
Flushing the buffers before the swap or avoiding asynchronous inserts for critical operations are potential solutions to ensure data consistency and avoid data loss.

# How Asynchronous Inserts Work
When you enable asynchronous inserts (via the async_insert setting), ClickHouse buffers the data in memory for a short period of time before flushing it to the target table.  
This **buffer is associated with the physical table** (i.e., the underlying storage and metadata of the table), not just the table's name.

## What Happens During an EXCHANGE Operation
When you use the EXCHANGE TABLES operation, ClickHouse swaps the metadata of the two tables. This means:

The table names are logically swapped.

The underlying data and storage of the tables are also swapped.

However, the asynchronous insert buffer is not affected by this metadata swap.  
It remains tied to the original physical table (the one it was initially buffering data for).  
This means:
* If data is still in the buffer when the EXCHANGE happens, it will be flushed to the original table, even if that table now has a different name.
* After the EXCHANGE, new asynchronous inserts will go to the new physical table (the one that was swapped in).

### Example Scenario
Let’s say you have two tables:
```
table_A
table_B
```

You enable asynchronous inserts for both tables. Data is being buffered for table_A.  
Then, you perform an EXCHANGE TABLES operation:
```
EXCHANGE TABLES table_A AND table_B;
```

### After the exchange:

The table previously known as table_A is now named table_B.
The table previously known as table_B is now named table_A.

### However:

Any data still in the asynchronous insert buffer for the original table_A will be flushed to the physical table that was originally table_A (now named table_B).

New asynchronous inserts will go to the new table_A (which was originally table_B).

### Key Takeaway
The asynchronous insert buffer is physically tied to the original table's storage and metadata, not just its name.  
Even if the table name changes (e.g., via EXCHANGE), the buffer will continue to flush data to the original physical table.

### Implications for Your Use Case
If you rely on the EXCHANGE operation to switch tables and want to ensure data consistency:
* Flush Buffers Before Exchange: Use SYSTEM FLUSH LOGS or disable asynchronous inserts temporarily to ensure all buffered data is flushed before performing the EXCHANGE.
* Avoid Asynchronous Inserts for Critical Swaps: If the timing of the EXCHANGE is critical, consider not using asynchronous inserts for these tables.
* Monitor Buffered Data: Implement logic to handle any remaining buffered data after the EXCHANGE.

## Conclusion
Yes, the asynchronous insert buffer is tightly bound to the physical table, not just its name.  
This means that even after an EXCHANGE, the buffer will continue to flush data to the original table, regardless of its new name.

# SYSTEM FLUSH ASYNC INSERT QUEUE
This command is specifically designed for asynchronous inserts.  
When you issue this command:
* It forces ClickHouse to immediately flush all data from the asynchronous insert queue (in-memory buffer) to the target tables.
* After flushing, the buffer is cleared, and the data is written to the underlying storage of the tables.

## Use Case for SYSTEM FLUSH ASYNC INSERT QUEUE
* You want to ensure that all buffered data is written to the tables before performing an operation like EXCHANGE TABLES.
* You need to guarantee that no data remains in the asynchronous insert queue, which could otherwise be lost or misdirected after a table swap.

### Example
```
SYSTEM FLUSH ASYNC INSERT QUEUE;
```
This ensures that all data in the asynchronous insert buffer is flushed to the tables before the swap, avoiding any issues with data being written to the wrong table after the EXCHANGE.

### Additional Notes
* Timing: Flushing the asynchronous insert queue can take some time if there is a large amount of buffered data. Plan accordingly if you need to perform the EXCHANGE operation immediately afterward.
* Idempotency: Both commands are idempotent, meaning you can safely run them multiple times without causing issues.
* Performance Impact: Flushing the asynchronous insert queue can temporarily increase disk I/O, so be mindful of the performance impact during peak loads.

















