# Задание 1: BRIN индексы и bitmap-сканирование

1. Удалите старую базу данных, если есть:
   ```shell
   docker compose down
   ```

2. Поднимите базу данных из src/docker-compose.yml:
   ```shell
   docker compose down && docker compose up -d
   ```

3. Обновите статистику:
   ```sql
   ANALYZE t_books;
   ```

4. Создайте BRIN индекс по колонке category:
   ```sql
   CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category);
   ```

5. Найдите книги с NULL значением category:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE category IS NULL;
   ```
   
   *План выполнения:*
   ```
   Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.010..0.010 rows=0 loops=1)
     Recheck Cond: (category IS NULL)
     ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.007..0.007 rows=0 loops=1)
           Index Cond: (category IS NULL)
   Planning Time: 0.446 ms
   Execution Time: 0.056 ms
   ```

   *Объясните результат:*
   PostgreSQL использует BRIN индекс для поиска NULL значений. Bitmap Heap Scan работает в два этапа:
   сначала Bitmap Index Scan по BRIN индексу находит блоки, которые могут содержать NULL значения,
   затем Bitmap Heap Scan проверяет эти блоки. В данном случае NULL значений не найдено (rows=0).
   Время выполнения очень быстрое (0.056 мс) благодаря компактности BRIN индекса.

6. Создайте BRIN индекс по автору:
   ```sql
   CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
   ```

7. Выполните поиск по категории и автору:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books 
   WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*
   ```
   Bitmap Heap Scan on t_books  (cost=12.16..2343.87 rows=1 width=33) (actual time=8.869..8.870 rows=0 loops=1)
     Recheck Cond: ((category)::text = 'INDEX'::text)
     Rows Removed by Index Recheck: 150000
     Filter: ((author)::text = 'SYSTEM'::text)
     Heap Blocks: lossy=1225
     ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.16 rows=73781 width=0) (actual time=0.061..0.061 rows=12250 loops=1)
           Index Cond: ((category)::text = 'INDEX'::text)
   Planning Time: 0.494 ms
   Execution Time: 8.944 ms
   ```

   *Объясните результат (обратите внимание на bitmap scan):*
   Запрос использует только один BRIN индекс (t_books_brin_cat_idx) по category, так как BRIN индексы не могут
   быть объединены для bitmap AND операции. Сначала выполняется Bitmap Index Scan по category='INDEX',
   который находит 1225 lossy блоков. Затем Bitmap Heap Scan проверяет все строки в этих блоках (150000 строк)
   и применяет фильтр author='SYSTEM'. Большое количество удаленных строк (Rows Removed by Index Recheck: 150000)
   показывает, что BRIN индекс читает все блоки целиком, что является его особенностью - низкая точность, но малый размер.

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   ```
   Sort  (cost=3100.11..3100.12 rows=5 width=7) (actual time=23.475..23.476 rows=6 loops=1)
     Sort Key: category
     Sort Method: quicksort  Memory: 25kB
     ->  HashAggregate  (cost=3100.00..3100.05 rows=5 width=7) (actual time=23.427..23.429 rows=6 loops=1)
           Group Key: category
           Batches: 1  Memory Usage: 24kB
           ->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=7) (actual time=0.006..7.563 rows=150000 loops=1)
   Planning Time: 0.439 ms
   Execution Time: 23.615 ms
   ```

   *Объясните результат:*
   PostgreSQL выполняет Sequential Scan всей таблицы, так как BRIN индекс не может эффективно помочь
   в получении уникальных значений. HashAggregate используется для группировки и удаления дубликатов,
   затем результат сортируется. BRIN индекс не подходит для этого типа запросов, так как он хранит
   только минимальные и максимальные значения для диапазонов блоков, а не список уникальных значений.

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   ```
   Aggregate  (cost=3100.03..3100.05 rows=1 width=8) (actual time=6.988..6.989 rows=1 loops=1)
     ->  Seq Scan on t_books  (cost=0.00..3100.00 rows=14 width=0) (actual time=6.985..6.985 rows=0 loops=1)
           Filter: ((author)::text ~~ 'S%'::text)
           Rows Removed by Filter: 150000
   Planning Time: 0.678 ms
   Execution Time: 7.053 ms
   ```

   *Объясните результат:*
   PostgreSQL не использует BRIN индекс по автору, так как BRIN индекс не эффективен для LIKE запросов
   с wildcards. Выполняется полное последовательное сканирование таблицы (Seq Scan) с применением
   фильтра. Все 150000 строк проверяются, но ни одна не соответствует условию (rows=0).
   BRIN индексы эффективны только для простых операций сравнения на отсортированных данных.

10. Создайте индекс для регистронезависимого поиска:
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
    ```

11. Подсчитайте книги, начинающиеся на 'O':
    ```sql
    EXPLAIN ANALYZE
    SELECT COUNT(*) 
    FROM t_books 
    WHERE LOWER(title) LIKE 'o%';
    ```
   
   *План выполнения:*
   ```
   Aggregate  (cost=3476.88..3476.89 rows=1 width=8) (actual time=24.006..24.006 rows=1 loops=1)
     ->  Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=0) (actual time=23.992..23.994 rows=1 loops=1)
           Filter: (lower((title)::text) ~~ 'o%'::text)
           Rows Removed by Filter: 149999
   Planning Time: 0.762 ms
   Execution Time: 24.059 ms
   ```

   *Объясните результат:*
   Несмотря на наличие индекса по LOWER(title), PostgreSQL выполняет Sequential Scan. Это происходит
   потому что B-tree индекс неэффективен для LIKE запросов с wildcard паттернами, даже если создан
   функциональный индекс. Для эффективного поиска по паттернам лучше использовать GIN индексы с
   pg_trgm расширением или полнотекстовый поиск. Найдена одна книга из 150000.

12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_brin_cat_idx;
    DROP INDEX t_books_brin_author_idx;
    DROP INDEX t_books_lower_title_idx;
    ```

13. Создайте составной BRIN индекс:
    ```sql
    CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
    USING brin(category, author);
    ```

14. Повторите запрос из шага 7:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE category = 'INDEX' AND author = 'SYSTEM';
    ```
   
   *План выполнения:*
   ```
   Bitmap Heap Scan on t_books  (cost=12.16..2343.87 rows=1 width=33) (actual time=0.593..0.594 rows=0 loops=1)
     Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
     Rows Removed by Index Recheck: 8845
     Heap Blocks: lossy=73
     ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.16 rows=73781 width=0) (actual time=0.042..0.042 rows=730 loops=1)
           Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Planning Time: 0.397 ms
   Execution Time: 0.644 ms
   ```

   *Объясните результат:*
   Составной BRIN индекс показывает значительное улучшение по сравнению с двумя отдельными индексами.
   Execution Time улучшилось с 8.944 мс до 0.644 мс (примерно в 14 раз быстрее).
   Heap Blocks уменьшилось с 1225 до 73, а количество проверенных строк с 150000 до 8845.
   Составной BRIN индекс позволяет фильтровать блоки сразу по обоим условиям, что дает лучшую
   селективность и значительно снижает количество блоков для проверки.