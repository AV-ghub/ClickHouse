[src](https://clickhouse.com/docs/en/optimize/sparse-primary-indexes#generic-exclusion-search-algorithm)

### Identifying single rows efficiently

In order to significantly improve the compression ratio for the content column while still achieving fast retrieval of specific rows, 
pastila.nl is using **two hashes** (and a compound primary key) for identifying a specific row:

* a hash of the content, as discussed above, that is distinct for distinct data, and
* a [locality-sensitive hash](https://en.wikipedia.org/wiki/Locality-sensitive_hashing) (fingerprint) **that does not change on small changes** of data

### Общее заключение
Для обеспечения максимально эффективного поиска не только по первому полю ключа (максимальная эффективность), но и по составляющим его вторичным полям, поля ключа, в отличие от b-tree индексов, необходимо располагать в порядке возрастания их кардинальности.
Это даст:
* максимальное пожатие на первом поле ключа с самой низкой кардинальностью (много повторяющихся значений)
* быстрое сужение поиска к искомому полю с отбросом максимального количества гранул

> Если при этом еще есть поддерживающий узкокоррелирующий колоночный индекс, то профит возможно может быть сопоставимым с поиском по первому полю ключа с убывающей кардинальностью.
