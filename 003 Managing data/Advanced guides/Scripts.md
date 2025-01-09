#### values table function
```
WITH
    left_table AS (SELECT * FROM VALUES('c UInt32', 1, 2, 3)),
    right_table AS (SELECT * FROM VALUES('c UInt32', 2, 2, 3, 3, 4))
SELECT
    l.c AS l_c,
    r.c AS r_c
FROM left_table AS l
LEFT ANY JOIN right_table AS r ON l.c = r.c;
```
#### Graph to pdf
```
./clickhouse client --host ekyyw56ard.us-west-2.aws.clickhouse.cloud --secure --port 9440 --password  --database=imdb_large --query "
EXPLAIN pipeline graph=1, compact=0
SELECT *
FROM actors a
JOIN roles r ON a.id = r.actor_id
SETTINGS max_threads = 2, join_algorithm = 'parallel_hash';" | dot -Tpdf > pipeline.pdf
```
