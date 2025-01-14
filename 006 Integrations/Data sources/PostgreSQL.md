## [PostgreSQL](https://clickhouse.com/docs/en/integrations/postgresql)
[Migrating from PostgreSQL to ClickHouse](https://github.com/AV-ghub/ClickHouse/blob/main/006%20Integrations/Migration%20Guides/PostgreSQL/Migrating%20from%20PostgreSQL%20to%20ClickHouse.md)

### [Using the PostgreSQL Table Engine](https://clickhouse.com/docs/en/integrations/postgresql#using-the-postgresql-table-engine)
```
-- PG
INSERT INTO table1
  (id, column1)
VALUES
  (1, 'abc'),
  (2, 'def');

-- CH
SELECT * FROM db_in_ch.table1
┌─id─┬─column1─┐
│  1 │ abc     │
│  2 │ def     │
└────┴─────────┘

-- PG
INSERT INTO table1
  (id, column1)
VALUES
  (3, 'ghi'),
  (4, 'jkl');

-- CH
SELECT * FROM db_in_ch.table1
┌─id─┬─column1─┐
│  1 │ abc     │
│  2 │ def     │
│  3 │ ghi     │
│  4 │ jkl     │
└────┴─────────┘

INSERT INTO db_in_ch.table1
  (id, column1)
VALUES
  (5, 'mno'),
  (6, 'pqr');

-- PG
SELECT * FROM table1;
id | column1
----+---------
  1 | abc
  2 | def
  3 | ghi
  4 | jkl
  5 | mno
  6 | pqr
```
