# Полный перечень параметров для CSV импорта в ClickHouse
## Полный перечень параметров SETTINGS для CSV:

```sql
INSERT INTO my_table
FROM INFILE '/home/import/table.csv'
SETTINGS 
    -- Основные параметры разделителей и кавычек
    format_csv_delimiter = '|',
    format_csv_allow_single_quotes = 0,
    format_csv_allow_double_quotes = 1,
    
    -- Параметры обработки строк
    input_format_csv_skip_first_lines = 1,
    input_format_csv_skip_trailing_empty_lines = 1,
    input_format_csv_trim_whitespace = 1,
    input_format_csv_skip_blank_lines = 1,
    
    -- Параметры для NULL и значений по умолчанию
    input_format_csv_empty_as_default = 1,
    input_format_csv_null_as_default = 1,
    input_format_defaults_for_omitted_fields = 1,
    
    -- Параметры экранирования
    format_csv_escape_rule = 'CSV',  -- или 'Quoted', 'JSON', 'None'
    format_csv_allow_errors = 0,
    input_format_csv_allow_variable_number_of_columns = 0,
    
    -- Параметры производительности
    input_format_csv_max_rows_to_read_for_schema_inference = 1000,
    input_format_csv_max_bytes_to_read_for_schema_inference = 1048576,
    
    -- Параметры кодировки и парсинга
    input_format_csv_use_best_effort_in_schema_inference = 1,
    input_format_csv_try_infer_numbers_from_strings = 1,
    input_format_csv_try_infer_dates_from_strings = 1,
    input_format_csv_try_infer_datetimes_from_strings = 1,
    
    -- Параметры для больших файлов
    input_format_csv_buffer_size = 1048576,
    input_format_csv_skip_rows_with_errors = 0,
    input_format_csv_max_rows_to_skip_with_errors = 0
    
FORMAT CSV
```

## Группировка по категориям:

### 1. **Разделители и кавычки:**
```sql
format_csv_delimiter = '|',                    -- Разделитель полей
format_csv_allow_single_quotes = 0,            -- Разрешить одинарные кавычки
format_csv_allow_double_quotes = 1,            -- Разрешить двойные кавычки
format_csv_escape_rule = 'CSV',                -- Правила экранирования
```

### 2. **Обработка строк:**
```sql
input_format_csv_skip_first_lines = 1,         -- Пропустить строки заголовка
input_format_csv_skip_trailing_empty_lines = 1,-- Пропустить пустые строки в конце
input_format_csv_trim_whitespace = 1,          -- Обрезать пробелы вокруг значений
input_format_csv_skip_blank_lines = 1,         -- Пропустить полностью пустые строки
```

### 3. **NULL и значения по умолчанию:**
```sql
input_format_csv_empty_as_default = 1,         -- Пустые строки как DEFAULT
input_format_csv_null_as_default = 1,          -- NULL как DEFAULT  
input_format_defaults_for_omitted_fields = 1,  -- Использовать DEFAULT для пропущенных полей
```

### 4. **Обработка ошибок:**
```sql
format_csv_allow_errors = 0,                   -- Разрешить ошибки парсинга
input_format_csv_allow_variable_number_of_columns = 0, -- Переменное число колонок
input_format_csv_skip_rows_with_errors = 0,    -- Пропускать строки с ошибками
input_format_csv_max_rows_to_skip_with_errors = 0, -- Максимум строк для пропуска
```

### 5. **Определение схемы:**
```sql
input_format_csv_max_rows_to_read_for_schema_inference = 1000,
input_format_csv_max_bytes_to_read_for_schema_inference = 1048576,
input_format_csv_use_best_effort_in_schema_inference = 1,
input_format_csv_try_infer_numbers_from_strings = 1,
input_format_csv_try_infer_dates_from_strings = 1,
input_format_csv_try_infer_datetimes_from_strings = 1,
```

## Практический пример:

```sql
INSERT INTO sales_data
FROM INFILE '/home/import/sales.csv'
SETTINGS 
    format_csv_delimiter = '|',
    format_csv_allow_single_quotes = 0,
    format_csv_allow_double_quotes = 1,
    input_format_csv_skip_first_lines = 1,
    input_format_csv_trim_whitespace = 1,
    input_format_csv_empty_as_default = 1,
    input_format_csv_null_as_default = 1,
    format_csv_escape_rule = 'CSV',
    input_format_csv_buffer_size = 2097152
FORMAT CSV
```

## Проверка доступных параметров:

```sql
-- Посмотреть все параметры связанные с CSV
SELECT name, value, description
FROM system.settings 
WHERE name LIKE '%csv%' 
   OR name LIKE '%format%'
ORDER BY name;

-- Или все настройки форматов
SELECT * FROM system.settings 
WHERE name LIKE 'input_format_%' 
   OR name LIKE 'format_%';
```

## Важные нюансы:

1. **Порядок важен** - SETTINGS должно быть перед FORMAT
2. **Некоторые параметры доступны только в новых версиях** ClickHouse
3. **Не все параметры совместимы** друг с другом
4. **Лучше тестировать** на небольших файлах перед обработкой больших объемов

Этот полный перечень покрывает практически все возможные сценарии импорта CSV в ClickHouse!

# Параметры определения схемы 
Относятся к **автоматическому определению схемы данных** (schema inference) в ClickHouse.   
Они позволяют ClickHouse автоматически определять типы данных при импорте CSV без явного указания структуры.

## Подробное назначение каждого параметра:

### 1. **`input_format_csv_max_rows_to_read_for_schema_inference = 1000`**
- **Назначение**: Ограничивает количество строк, которые ClickHouse анализирует для определения типов данных
- **Зачем**: Чтобы не читать весь огромный файл для определения схемы
- **Пример**: Если файл содержит 1M строк, ClickHouse проанализирует только первые 1000 для определения типов

### 2. **`input_format_csv_max_bytes_to_read_for_schema_inference = 1048576`**
- **Назначение**: Ограничивает объем данных (1 МБ) для анализа схемы
- **Зачем**: Защита от очень широких таблиц с большими текстовыми полями
- **Пример**: Даже если в первых 10 строках есть огромные JSON-поля, анализ не превысит 1 МБ

### 3. **`input_format_csv_use_best_effort_in_schema_inference = 1`**
- **Назначение**: Включить "усиленный" режим определения типов
- **Зачем**: Более агрессивное преобразование строк в числа/даты
- **Пример**: Строка `"123.45"` будет определена как `Float64`, а не `String`

### 4. **`input_format_csv_try_infer_numbers_from_strings = 1`**
- **Назначение**: Попытаться преобразовать строки в числовые типы
- **Зачем**: Автоматическое определение чисел в строковых полях
- **Пример**: 
  - `"42"` → `Int32`
  - `"3.14"` → `Float64`
  - `"123abc"` → останется `String` (нечисловое значение)

### 5. **`input_format_csv_try_infer_dates_from_strings = 1`**
- **Назначение**: Определять строки как даты `Date`
- **Зачем**: Автоматическое распознавание дат
- **Пример**: 
  - `"2023-12-25"` → `Date`
  - `"25/12/2023"` → зависит от локали

### 6. **`input_format_csv_try_infer_datetimes_from_strings = 1`**
- **Назначение**: Определять строки как даты-время `DateTime`
- **Зачем**: Автоматическое распознавание временных меток
- **Пример**: 
  - `"2023-12-25 14:30:00"` → `DateTime`
  - `"2023-12-25T14:30:00Z"` → `DateTime`

## Практический пример использования:

```sql
-- ClickHouse автоматически определит типы данных
INSERT INTO inferred_table
FROM INFILE '/home/data/auto_detect.csv'
SETTINGS 
    input_format_csv_max_rows_to_read_for_schema_inference = 500,
    input_format_csv_max_bytes_to_read_for_schema_inference = 524288,
    input_format_csv_use_best_effort_in_schema_inference = 1,
    input_format_csv_try_infer_numbers_from_strings = 1,
    input_format_csv_try_infer_dates_from_strings = 1,
    input_format_csv_try_infer_datetimes_from_strings = 1,
    input_format_csv_skip_first_lines = 1
FORMAT CSV
```

## Что происходит внутри:

1. **ClickHouse читает первые 500 строк** (или 512 КБ)
2. **Анализирует каждое поле** на предмет возможных типов
3. **Применяет эвристики** для определения наиболее подходящего типа
4. **Создает временную схему** и загружает данные по этой схеме

## Пример преобразования:

**CSV файл:**
```
id,name,price,date,created_at
1,"Product A",19.99,2023-12-25,2023-12-25 10:30:00
2,"Product B",29.50,2023-12-26,2023-12-26 14:45:00
```

**Автоматически определенные типы:**
- `id` → `Int32`
- `name` → `String` 
- `price` → `Float64`
- `date` → `Date`
- `created_at` → `DateTime`

## Когда это особенно полезно:

1. **Исследовательский анализ** - быстро загрузить данные без ручного описания схемы
2. **Временные импорты** - для одноразовых задач
3. **Неизвестная структура** - когда заранее не известны типы данных
4. **Прототипирование** - быстрый старт работы с данными

## Ограничения и риски:

```sql
-- Лучше явно указать типы для критичных данных
INSERT INTO precise_table
FROM INFILE '/home/data/important.csv'
SETTINGS 
    input_format_csv_empty_as_default = 1,
    input_format_csv_null_as_default = 1
FORMAT CSVWithNames

-- Или использовать функцию file() с явной схемой
INSERT INTO precise_table
SELECT *
FROM file(
    '/home/data/important.csv', 
    'CSVWithNames',
    'id Int32, name String, price Decimal(10,2), date Date'
)
```

Эти параметры делают импорт в ClickHouse **интеллектуальным и адаптивным**, особенно полезным для исследовательской работы с данными!
