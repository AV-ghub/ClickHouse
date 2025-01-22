### Overview
[Заметки про ClickHouse. Tutorial 101 — Большая подборка информации](https://ivan-shamaev.ru/clickhouse-101-course-on-learn-clickhouse-com/)   
[Golang ClickHouse](https://clickhouse.uptrace.dev/clickhouse/configuration.html)  
[ClickHouse v24.10 Release Webinar](https://www.youtube.com/watch?v=AamIAjURp4U)  
[Теория и практика использования ClickHouse в реальных приложениях. Александр Зайцев (2018г)](https://habr.com/ru/articles/512304/) 
[Краеугольные камни ClickHouse](https://habr.com/ru/companies/wildberries/articles/821865/)   
[Why is ClickHouse so fast?](https://chistadata.com/why-clickhouse-is-so-fast/)   
[ClickHouse Parts and Partitions: Part 1](https://chistadata.com/parts-and-partitions-in-clickhouse-part-i/)  
[Altinity® Knowledge Base for ClickHouse®](https://kb.altinity.com/)   
[Дом, милый дом: нюансы работы с ClickHouse. Часть 1](https://habr.com/ru/companies/nixys/articles/801029/)  
[Дом, милый дом: нюансы работы с ClickHouse. Часть 2](https://habr.com/ru/companies/nixys/articles/826850/)  
[Заметки про ClickHouse. Tutorial 101 — Большая подборка информации](https://datafinder.ru/products/zametki-pro-clickhouse-tutorial-101-bolshaya-podborka-informacii)   

#### Video
[Leaving The Two Tier Architecture Behind by Hannes Muehleisen (Dijkstra Award 2024)](https://www.youtube.com/playlist?list=PLO3lfQbpDVI-hyw4MyqxEk3rDHw95SzxJ)   
[ClickHouse тормозит / Кирилл Шваков (TrafficStars)](https://www.youtube.com/watch?v=efRryvtKlq0&list=PLO3lfQbpDVI-hyw4MyqxEk3rDHw95SzxJ&index=45&t=201s)   


### Theory
[Vectorization vs. Compilation in Query Execution](https://15721.courses.cs.cmu.edu/spring2016/papers/p5-sompolski.pdf)

### Architecture
[Обзор архитектуры ClickHouse](https://clickhouse.com/docs/ru/development/architecture#merge-tree)  
[ClickHouse Parts and Partitions](https://chistadata.com/parts-and-partitions-in-clickhouse-part-i/)   
[ClickHouse Performance: Inside the Query Execution Pipeline](https://chistadata.com/inside-query-execution-pipeline-clickhouse/)   
[CTO’s Guide to ColumnStores vs Row-based Databases for Real-time Analytics](https://chistadata.com/ctos-guide-to-clickhouse-columnstores/)
[ClickHouse Basic Tutorial: Keys & Indexes](https://dev.to/hoptical/clickhouse-basic-tutorial-keys-indexes-5d7a)   
[Индексы в ClickHouse](https://bigdataschool.ru/blog/news/clickhouse/indexes-in-clickhouse.html)   
> [A Practical Introduction to Primary Indexes in ClickHouse](https://clickhouse.com/docs/en/optimize/sparse-primary-indexes)   

### Benchmarks
[ClickBench — a Benchmark For Analytical DBMS](https://benchmark.clickhouse.com/)   
[ClickHouse Versions Benchmark](https://benchmark.clickhouse.com/versions/)  
[ClickHouse Hardware Benchmark](https://benchmark.clickhouse.com/hardware/)

### Tests
[How to Test Your Hardware with ClickHouse](https://clickhouse.com/docs/en/operations/performance-test)  
[ClickHouse, Redshift and 2.5 Billion Rows of Time Series Data](https://brandonharris.io/redshift-clickhouse-time-series/)   
[Testing the Performance of ClickHouse](https://clickhouse.com/blog/testing-the-performance-of-click-house)   
[fiddle](https://fiddle.clickhouse.com/)

### Performance, tune & optimization
[The Secrets of ClickHouse Performance Optimizations at BDTC 2019](https://www.youtube.com/watch?v=ZOZQCQEtrz8)  
[Tuning ClickHouse for High-velocity Data Ingestion in Distributed Tables](https://chistadata.com/fine-tuning-data-ingestion-in-clickhouse-distributed-tables/)   
[How to implement Bloom Filters in ClickHouse for Query Performance](https://chistadata.com/implementing-bloom-filters-in-clickhouse/)    
[Super charging your ClickHouse queries](https://clickhouse.com/blog/clickhouse-faster-queries-with-projections-and-primary-indexes)  
[A Practical Introduction to Primary Indexes in ClickHouse](https://clickhouse.com/docs/en/optimize/sparse-primary-indexes)   [ --> Notes](https://github.com/AV-ghub/ClickHouse/blob/main/999%20Links/Notes/Performance%20and%20Optimizations.md)    
[Understanding ClickHouse Data Skipping Indexes](https://clickhouse.com/docs/en/optimize/skipping-indexes)   [ --> Notes](https://github.com/AV-ghub/ClickHouse/blob/main/999%20Links/Notes/Understanding%20ClickHouse%20Data%20Skipping%20Indexes.md)

### Practice
[Наш опыт внедрения ClickHouse – аналитической CУБД](https://gs-studio.com/news-about-it/32777----clickhouse---c)   
[Local Наш опыт внедрения ClickHouse – аналитической CУБД](https://github.com/AV-ghub/ClickHouse/blob/main/002%20%D0%90%D1%80%D1%85%D0%B8%D1%82%D0%B5%D0%BA%D1%82%D1%83%D1%80%D0%B0/003%20%D0%9D%D0%B0%D1%88%20%D0%BE%D0%BF%D1%8B%D1%82%20%D0%B2%D0%BD%D0%B5%D0%B4%D1%80%D0%B5%D0%BD%D0%B8%D1%8F%20ClickHouse%20%E2%80%93%20%D0%B0%D0%BD%D0%B0%D0%BB%D0%B8%D1%82%D0%B8%D1%87%D0%B5%D1%81%D0%BA%D0%BE%D0%B9%20C%D0%A3%D0%91%D0%94.md)    
[Working with Time Series Data in ClickHouse](https://clickhouse.com/blog/working-with-time-series-data-and-functions-ClickHouse)   
[How to Use Query Parameters in ClickHouse](https://www.youtube.com/watch?v=mAvE7ZKVja4)   

### Admin
[Creating an Admin User in ClickHouse](https://chistadata.com/knowledge-base/creating-an-admin-user-in-clickhouse/)   
[Репликация данных](https://clickhouse.com/docs/ru/engines/table-engines/mergetree-family/replication)   

### Dev
[How to implement `pivot` in clickhouse just like in dolphindb](https://stackoverflow.com/questions/56074216/how-to-implement-pivot-in-clickhouse-just-like-in-dolphindb)   
[C# integration](https://clickhouse.com/integrations/csharp)    
[Processing JSON](https://clickhouse.com/docs/en/integrations/data-formats/json/overview)

### Util
[ZooKeeper](https://zookeeper.apache.org/doc/r3.9.3/zookeeperOver.html)   
















