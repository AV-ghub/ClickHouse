# 1. **Многобайтовые разделители в BCP**

```bash
# Разделитель из двух символов
bcp ... -t "||" ...

# Разделитель из трех символов  
bcp ... -t "|||" ...

# Любая комбинация символов
bcp ... -t "§§§" ...
bcp ... -t "###" ...
bcp ... -t "END_FIELD" ...  # Да, даже текст!
```

## 2. **Специальные Unicode-символы (рекомендуется)**

```bash
# Непечатаемые символы Unicode
bcp ... -t "␟" ...     # U+241F - SYMBOL FOR UNIT SEPARATOR
bcp ... -t "␞" ...     # U+241E - SYMBOL FOR RECORD SEPARATOR  
bcp ... -t "␝" ...     # U+241D - SYMBOL FOR GROUP SEPARATOR
bcp ... -t "␜" ...     # U+241C - SYMBOL FOR FILE SEPARATOR

# Редкие математические символы
bcp ... -t "‡" ...     # Двойной крест
bcp ... -t "†" ...     # Крест
bcp ... -t "§" ...     # Параграф
bcp ... -t "¶" ...     # Абзац
```

## 3. **Экранирование специальных символов в CMD**

```batch
# Для символов, которые могут интерпретироваться CMD
bcp ... -t "^|" ...    # Экранирование管道 в CMD
bcp ... -t "^^" ...    # Экранирование самого символа ^
```

## 4. **Рекомендуемые "безопасные" разделители**

### Вариант A: **Unicode Unit Separator** (самый надежный)
```bash
bcp "SELECT * FROM table" queryout "data.csv" ^
-S "srv" -d "db" -U "usr" -P "pwd" ^
-c -C 65001 ^
-t "␟" ^     # U+241F - практически гарантированно нет в данных
-r "␞" ^     # U+241E - разделитель строк
-q
```

### Вариант B: **Комбинация редко используемых символов**
```bash
bcp ... -t "§|§" ...    # Маловероятно в обычных данных
bcp ... -t "~#~" ...
bcp ... -t ":::" ...
```

### Вариант C: **HEX-коды символов**
```bash
# Для точного контроля (в PowerShell)
bcp ... -t "$([char]0x241F)" ...
```

## 5. **Практический пример с безопасным разделителем**

```batch
@echo off
set DELIM=␟
set ROW_DELIM=␞

bcp "SELECT * FROM [dbo].[_srcr_batch]" queryout "C:\Unload\data.csv" ^
-S "srv" -d "db" -U "usr" -P "pwd" ^
-c -C 65001 ^
-t "%DELIM%" ^
-r "%ROW_DELIM%" ^
-q -N > nul 2>&1

echo Export completed with delimiter: %DELIM%
```

## 6. **Проверка на конфликты в данных**

```sql
-- Проверить, есть ли выбранный разделитель в данных
SELECT 
    COUNT(*) as total_rows,
    SUM(CASE WHEN CHARINDEX('␟', [text_field]) > 0 THEN 1 ELSE 0 END) as conflicts
FROM [dbo].[your_table]
```

## 7. **Для импорта в ClickHouse с кастомным разделителем**

```bash
clickhouse-client --query "
INSERT INTO table 
FORMAT CSV
" \
--format_csv_delimiter="␟" \
--input_format_csv_skip_first_lines=0 < data.csv
```

## 8. **Альтернатива: использование формата XML/JSON**

Если данные очень сложные, лучше использовать структурированные форматы:

```bash
# Экспорт в JSON
bcp "SELECT * FROM table FOR JSON PATH" queryout "data.json" ...

# Экспорт в XML
bcp "SELECT * FROM table FOR XML PATH" queryout "data.xml" ...
```

## 9. **Самый надежный способ: предобработка данных**

```bash
# Экранирование разделителей в данных перед экспортом
bcp "SELECT 
        REPLACE(REPLACE(field1, '|', '||'), '␟', '␟␟') as field1,
        field2
     FROM table" queryout "data.csv" ...
```

## Рекомендация:

**Используйте Unicode-символы U+241F или U+241E** - они специально designed для разделения данных и практически никогда не встречаются в пользовательском контенте.

```bash
# Идеальный вариант
bcp ... -t "␟" -r "␞" ...
```

Это гарантирует отсутствие коллизий при импорте/экспорте!
