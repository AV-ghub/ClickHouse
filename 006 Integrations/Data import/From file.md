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
