#### [Aggregate Function Combinators](https://clickhouse.com/docs/en/sql-reference/aggregate-functions/combinators)   
To use a combinator, we have to do two things.  
* First, choose an **aggregate function** we want to use; let's say we want a sum() function.  
* Second, pick a **combinator** needed for our case; let's say we need an If combinator.

We can combine any number of combinators in a single function:
```
SELECT sumArrayIf(...)
```

#### Additional resources
[Using Aggregate Combinators in ClickHouse](https://clickhouse.com/blog/aggregate-functions-combinators-in-clickhouse-for-arrays-maps-and-states)

The principal advantage of using the conditional If, over a standard WHERE clause, is the **ability to compute multiple sums** for different clauses.   
We can aggregate on multiple conditions and functions within a single request:
```
SELECT
    countIf((status = 'confirmed') AND (confirm_time > (create_time + toIntervalMinute(1)))) AS num_confirmed_checked,
    sumIf(total_amount, (status = 'confirmed') AND (confirm_time > (create_time + toIntervalMinute(1)))) AS confirmed_checked_amount,
    countIf(status = 'declined') AS num_declined,
    sumIf(total_amount, status = 'declined') AS dec_amount,
    avgIf(total_amount, status = 'declined') AS dec_average
FROM payments
```


