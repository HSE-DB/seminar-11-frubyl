## Задание 2

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

4. Создайте полнотекстовый индекс:
    ```sql
    CREATE INDEX t_books_fts_idx ON t_books 
    USING GIN (to_tsvector('english', title));
    ```

5. Найдите книги, содержащие слово 'expert':
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE to_tsvector('english', title) @@ to_tsquery('english', 'expert');
    ```
    
    *План выполнения:*
    ```
    Bitmap Heap Scan on t_books  (cost=21.03..1336.08 rows=750 width=33) (actual time=0.381..0.382 rows=1 loops=1)
      Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
      Heap Blocks: exact=1
      ->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..20.84 rows=750 width=0) (actual time=0.373..0.373 rows=1 loops=1)
            Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
    Planning Time: 1.193 ms
    Execution Time: 0.424 ms
    ```

    *Объясните результат:*
    PostgreSQL эффективно использует GIN индекс для полнотекстового поиска. Bitmap Index Scan по GIN индексу
    быстро находит документы, содержащие слово 'expert' (найдена 1 книга). GIN индексы оптимизированы для
    полнотекстового поиска, храня инвертированный индекс токенов. Execution Time всего 0.424 мс,
    что демонстрирует высокую эффективность для текстового поиска. Heap Blocks: exact=1 означает,
    что нужен только один блок данных, без lossy блоков.

6. Удалите индекс:
    ```sql
    DROP INDEX t_books_fts_idx;
    ```

7. Создайте таблицу lookup:
    ```sql
    CREATE TABLE t_lookup (
         item_key VARCHAR(10) NOT NULL,
         item_value VARCHAR(100)
    );
    ```

8. Добавьте первичный ключ:
    ```sql
    ALTER TABLE t_lookup 
    ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key);
    ```

9. Заполните данными:
    ```sql
    INSERT INTO t_lookup 
    SELECT 
         LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
         'Value_' || generate_series(1, 150000);
    ```

10. Создайте кластеризованную таблицу:
     ```sql
     CREATE TABLE t_lookup_clustered (
          item_key VARCHAR(10) PRIMARY KEY,
          item_value VARCHAR(100)
     );
     ```

11. Заполните её теми же данными:
     ```sql
     INSERT INTO t_lookup_clustered 
     SELECT * FROM t_lookup;
     
     CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey;
     ```

12. Обновите статистику:
     ```sql
     ANALYZE t_lookup;
     ANALYZE t_lookup_clustered;
     ```

13. Выполните поиск по ключу в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ```
     Index Scan using t_lookup_pk on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.083..0.085 rows=1 loops=1)
       Index Cond: ((item_key)::text = '0000000455'::text)
     Planning Time: 1.113 ms
     Execution Time: 0.181 ms
     ```

     *Объясните результат:*
     Используется Index Scan по первичному ключу (B-tree индекс). Поиск очень быстрый (0.181 мс),
     так как B-tree индекс позволяет найти нужную запись за O(log N) операций. Найдена ровно 1 строка.
     Index Scan напрямую читает данные через индекс, что эффективно для точечных запросов по ключу.

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ```
     Index Scan using t_lookup_clustered_pkey on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.126..0.127 rows=1 loops=1)
       Index Cond: ((item_key)::text = '0000000455'::text)
     Planning Time: 0.379 ms
     Execution Time: 0.166 ms
     ```

     *Объясните результат:*
     Также используется Index Scan по первичному ключу. Execution Time немного лучше (0.166 мс против 0.181 мс),
     но разница незначительна. При поиске по первичному ключу кластеризация не дает большого преимущества,
     так как индекс уже обеспечивает быстрый доступ. Однако Planning Time лучше (0.379 мс против 1.113 мс),
     что может быть связано с лучшей локальностью данных после кластеризации.

15. Создайте индекс по значению для обычной таблицы:
     ```sql
     CREATE INDEX t_lookup_value_idx ON t_lookup(item_value);
     ```

16. Создайте индекс по значению для кластеризованной таблицы:
     ```sql
     CREATE INDEX t_lookup_clustered_value_idx 
     ON t_lookup_clustered(item_value);
     ```

17. Выполните поиск по значению в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```
     Index Scan using t_lookup_value_idx on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.091..0.092 rows=0 loops=1)
       Index Cond: ((item_value)::text = 'T_BOOKS'::text)
     Planning Time: 0.666 ms
     Execution Time: 0.140 ms
     ```

     *Объясните результат:*
     Используется Index Scan по вторичному индексу item_value. Значение 'T_BOOKS' не найдено в таблице (rows=0).
     Execution Time 0.140 мс показывает быстрый поиск благодаря B-tree индексу. Вторичный индекс требует
     дополнительного обращения к heap для получения полных данных строки, но для несуществующих значений
     это не требуется.

18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```
     Index Scan using t_lookup_clustered_value_idx on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.054..0.054 rows=0 loops=1)
       Index Cond: ((item_value)::text = 'T_BOOKS'::text)
     Planning Time: 0.342 ms
     Execution Time: 0.094 ms
     ```

     *Объясните результат:*
     Кластеризованная таблица показывает лучшую производительность: Execution Time 0.094 мс против 0.140 мс
     в обычной таблице (улучшение на ~33%). Planning Time также лучше (0.342 мс против 0.666 мс).
     Хотя данные не найдены, кластеризация улучшает кэш-локальность и уменьшает количество страниц,
     которые нужно читать, что ускоряет работу даже для отрицательных результатов поиска.

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:

     *Сравнение:*
     Кластеризованная таблица показала следующие преимущества:

     1. Поиск по первичному ключу:
        - Обычная таблица: Execution Time 0.181 мс, Planning Time 1.113 мс
        - Кластеризованная: Execution Time 0.166 мс, Planning Time 0.379 мс
        - Улучшение: ~8% execution time, ~66% planning time

     2. Поиск по вторичному индексу (item_value):
        - Обычная таблица: Execution Time 0.140 мс, Planning Time 0.666 мс
        - Кластеризованная: Execution Time 0.094 мс, Planning Time 0.342 мс
        - Улучшение: ~33% execution time, ~49% planning time

     Выводы:
     - Кластеризация дает наибольший эффект при поиске по вторичным индексам
     - Улучшается локальность данных, что положительно влияет на кэширование
     - Planning Time всегда лучше в кластеризованных таблицах
     - Для поиска по первичному ключу эффект меньше, так как B-tree уже обеспечивает хорошую производительность