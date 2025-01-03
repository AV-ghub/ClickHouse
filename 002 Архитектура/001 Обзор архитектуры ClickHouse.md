### [Обзор архитектуры ClickHouse](https://clickhouse.com/docs/ru/development/architecture#merge-tree)

Существует два различных подхода для увеличения скорости обработки запросов: 
* выполнение векторизованного запроса
* генерация кода во время выполнения (runtime code generation)

Генерация кода во время выполнения выигрывает, если объединяет большое число операций.   
Выполнение векторизованного запроса упрощает использование SIMD-инструкций CPU.

ClickHouse использует выполнение векторизованного запроса и имеет ограниченную начальную поддержку генерации кода во время выполнения.

### Столбцы
Почти все операции не изменяют данные (immutable): они не изменяют содержимое столбцов, а создают новые с изменёнными значениями.

### Поля
IColumn имеет метод оператор [] для получения значения по индексу n как Field, а также метод insert для добавления Field в конец колонки.   
Эти методы не очень эффективны, так как требуют временных объектов Field, представляющих индивидуальное значение.    
Есть более эффективные методы, такие как insertFrom, insertRangeFrom и т.д.

### Дырявые абстракции (Leaky Abstractions)
Различные функции на столбцах могут быть реализованы обобщённым, неэффективным путем.   
Или специальным путем, используя знания о внутреннем распределение данных в памяти в конкретной реализации IColumn.   
Для этого функции приводятся к конкретному типу IColumn и работают напрямую с его внутренним представлением.   

### Функции
Обычные функции не изменяют число строк и работают так, как если бы обрабатывали каждую строку независимо.   
В действительности же функции вызываются не к отдельным строкам, а блокам данных для реализации векторизованного выполнения запросов.

### Сервер
Сервер предоставляет несколько различных интерфейсов.

* HTTP-интерфейс для любых сторонних клиентов.
* TCP-интерфейс для родного ClickHouse-клиента и межсерверной взаимодействия при выполнении распределенных запросов.
* Интерфейс для передачи данных при репликации.

Для всех сторонних приложений мы рекомендуем использовать HTTP-интерфейс, потому что он прост и удобен в использовании.   
TCP-интерфейс тесно связан с внутренними структурами данных: он использует внутренний формат для передачи блоков данных и использует специальное кадрирование для сжатых данных.

### Выполнение распределенных запросов (Distributed Query Execution)
Вы можете создать распределённую (Distributed) таблицу на одном или всех серверах в кластере.   
Такая таблица сама по себе не хранит данные — она только предоставляет возможность “просмотра” всех локальных таблиц на нескольких узлах кластера.   
При выполнении SELECT **распределенная таблица переписывает запрос**, выбирает удаленные узлы в соответствии с настройками балансировки нагрузки и отправляет им запрос.   
Распределенная таблица просит удаленные сервера обработать запрос до той стадии, когда промежуточные результаты с разных серверов могут быть объединены.   
Затем он **получает промежуточные результаты и объединяет их**.   

Распределенная таблица пытается возложить как можно больше работы на удаленные серверы и сократить объем промежуточных данных, передаваемых по сети.

С**итуация усложняется при использовании подзапросов в случае IN или JOIN**, когда каждый из них использует таблицу Distributed.   
Есть **разные стратегии** для выполнения таких запросов.

Глобального плана выполнения распределённых запросов не существует. Каждый узел имеет собственный локальный план для своей части работы.   
У нас есть простое однонаправленное выполнение распределенных запросов: мы отправляем запросы на удаленные узлы и затем объединяем результаты.   
Но это невозможно для сложных запросов **GROUP BY высокой кардинальности или запросов с большим числом временных данных в JOIN**: в таких случаях нам необходимо перераспределить (“**reshuffle**”) данные между узлами, что требует дополнительной координации.   

> **ClickHouse не поддерживает выполнение запросов такого рода**.  

### Merge Tree
 Данные в таблице MergeTree хранятся “частями” (“parts”). Каждая часть хранит данные отсортированные по первичному ключу (данные упорядочены лексикографически). Все столбцы таблицы хранятся в отдельных файлах column.bin в этих частях. Файлы состоят из сжатых блоков. Каждый блок обычно содержит от 64 КБ до 1 МБ несжатых данных.   

Сам первичный ключ является “разреженным” (sparse). Он не относится к каждой отдельной строке, а только к некоторым диапазонам данных.   
Отдельный файл **«primary.idx»** имеет значение первичного ключа для каждой N-й строки, где N называется гранулярностью индекса (**index_granularity**, обычно N = 8192).   
Также для каждого столбца у нас есть файлы **column.mrk** с “метками” (“marks”), которые обозначают смещение для каждой N-й строки в файле данных.   
 
Мы сделали индекс разреженным, потому что мы должны иметь возможность поддерживать триллионы строк на один сервер без существенных расходов памяти на индексацию.   
Кроме того, поскольку **первичный ключ** является разреженным, он **не уникален**: он не может проверить наличие ключа в таблице во время INSERT. Вы можете иметь множество строк с одним и тем же ключом в таблице.

MergeTree не является LSM (Log-structured merge-tree — журнально-структурированным деревом со слиянием).   
Это делает его пригодным только для вставки данных в пакетах, а не по отдельным строкам и не очень часто — **примерно раз в секунду это нормально, а тысячу раз в секунду — нет**.

### Репликация
Репликация использует асинхронную **multi-master**-схему.   
Вы можете вставить данные в любую реплику, которая имеет открытую сессию в ZooKeeper, и данные реплицируются на все другие реплики асинхронно.   
Поскольку ClickHouse не поддерживает UPDATE, **репликация исключает конфликты** (conflict-free replication).   

> Поскольку **подтверждение вставок кворумом не реализовано**, **только что вставленные данные могут быть потеряны** в случае сбоя одного узла.

Репликация является **физической**: между узлами передаются только сжатые части, а не запросы. 

Реплика восстанавливает свою согласованность путем загрузки отсутствующих и поврежденных частей из других реплик.   
Когда в локальной файловой системе есть неожиданные или испорченные данные, ClickHouse не удаляет их, а перемещает в отдельный каталог и забывает об этом.

> Кластер ClickHouse состоит из независимых сегментов (shards), а каждый сегмент состоит из реплик.  
> Кластер не является эластичным (not elastic), поэтому после добавления нового сегмента данные не будут автоматически распределены между ними.
> Вместо этого нужно изменить настройки, чтобы выровнять нагрузку на кластер. 

### [Краеугольные камни ClickHouse](https://habr.com/ru/companies/wildberries/articles/821865/)

Все, что вы вставляете в ClickHouse, попадает на диск сразу же, не задерживаясь в MemTable. ClickHouse формирует **кусок данных** (здесь и далее по тексту под «куском» подразумевается та самая таблица внутри таблицы, то есть SSTable) из того, что вы вставляете в таблицу, и данные сразу попадают на диск. Если вы вставили 100 строк, то кусок у вас образуется из ста строк, если 500, то из пятиста и т. д. Только если в вашей вставке **больше миллиона строк** (**либо больше ~250 мегабайт**, смотря что наступит раньше), ClickHouse начнет дробить вставку на несколько кусков. Гарантируется атомарность в рамках одного куска. Это означает, что ваши вставки в ClickHouse будут атомарны до тех пор, пока вы вставляете в одну таблицу на одном сервере, в одну партицию и вставляете меньше, чем миллион строк. Настройку с миллионом строк или 250 мегабайтами можно подтюнить, если вам для чего‑то хочется увеличить границы атомарности, и вы готовы отдать на вставки чуть больше памяти.

**Гранулой** называют какое‑то количество строк внутри куска данных, на которые приходится одна индексная запись. 



