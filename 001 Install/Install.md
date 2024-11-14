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

> Есть табличка весом в 300гб, необходимо перелить все данные из нее в другую табличку. Насколько хорошая идея делать "INSERT INTO SELECT *"? Клик умеет такие операции оптимизировать?

Такую мелочь за 100 сек проливает
Делаем так в том числе и с табличками выше 1ТБ и из удаленного сервера с remote функцией
