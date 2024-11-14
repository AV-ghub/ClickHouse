[Install](https://clickhouse.com/docs/ru/getting-started/tutorial)

Данные находятся тут 
```
# cd /var/lib/clickhouse
# du -sh
34M     .
```

> For MC: Press Ctrl + Space on current directory and this will show it size.

> Просто считайте что вся дира **var/lib/clickhouse** это данные и монтируйте ее

![ex](https://github.com/AV-ghub/ClickHouse/blob/main/001%20Install/res/1.jpg)

В каталоге **metadata** хранятся описания .sql таблиц. Там внутри есть UUID этот UUID это имя каталога в store в data симлинки чтобы сохранить совместимость с прошлой структурой, с базами ordinary
