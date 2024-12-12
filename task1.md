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
   "Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.070..0.071 rows=0 loops=1)"
   "Recheck Cond: (category IS NULL)"
   "->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.055..0.056 rows=0 loops=1)"
   "Index Cond: (category IS NULL)"
   "Planning Time: 1.653 ms"
   "Execution Time: 0.114 ms"
   ```
   
   *Объясните результат:*
   Бриин индекс запомнил в каких блоках были NULL и позволил быстро найти все подходящие значения 

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
   "Bitmap Heap Scan on t_books  (cost=12.17..2455.11 rows=1 width=33) (actual time=32.204..32.207 rows=0 loops=1)"
   "Recheck Cond: ((category)::text = 'INDEX'::text)"
   "Rows Removed by Index Recheck: 150000"
   "Filter: ((author)::text = 'SYSTEM'::text)"
   "Heap Blocks: lossy=1225"
   "->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.17 rows=81196 width=0) (actual time=0.746..0.747 rows=12250 loops=1)"
   "Index Cond: ((category)::text =  'INDEX'::text)"
   "Planning Time: 0.508 ms"
   "Execution Time: 32.321 ms"
   ```
   
   *Объясните результат (обратите внимание на bitmap scan):*
   Был использован индекс 

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   ```
   "Sort  (cost=3100.11..3100.12 rows=5 width=7) (actual time=61.488..61.491 rows=6 loops=1)"
   "Sort Key: category"
   "Sort Method: quicksort  Memory: 25kB"
   "->  HashAggregate  (cost=3100.00..3100.05 rows=5 width=7) (actual time=61.338..61.341 rows=6 loops=1)"
   "Group Key: category"
   "Batches: 1  Memory Usage: 24kB"
   "->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=7) (actual time=0.065..15.387 rows=150000 loops=1)"
   "Planning Time: 1.884 ms"
   "Execution Time: 61.749 ms"
   ```
   
   *Объясните результат:*
   BRIN индекс запоминает только данные о блоках, поэтому не может быть использован в сортировках и в проверке уникальности 

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   ```
   "Aggregate  (cost=3100.04..3100.05 rows=1 width=8) (actual time=23.411..23.413 rows=1 loops=1)"
   "->  Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=0) (actual time=23.403..23.404 rows=0 loops=1)"
   "Filter: ((author)::text ~~ 'S%'::text)"
   "Rows Removed by Filter: 150000"
   "Planning Time: 0.816 ms"
   "Execution Time: 23.471 ms"
   ```
   
   *Объясните результат:*
   По сейм причине не может использоваться в LIKE 

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
   "Aggregate  (cost=3476.88..3476.89 rows=1 width=8) (actual time=89.861..89.864 rows=1 loops=1)"
   "->  Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=0) (actual time=89.850..89.853 rows=1 loops=1)"
   "Filter: (lower((title)::text) ~~ 'o%'::text)"
   "Rows Removed by Filter: 149999"
   "Planning Time: 0.528 ms"
   "Execution Time: 89.914 ms"
   ```
   
   *Объясните результат:*
   В прошлом дз писал почему не робит

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
   ````
   "Bitmap Heap Scan on t_books  (cost=12.17..2455.11 rows=1 width=33) (actual time=1.496..1.499 rows=0 loops=1)"
   "Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))"
   "Rows Removed by Index Recheck: 8831"
   "Heap Blocks: lossy=73"
   "->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.17 rows=81196 width=0) (actual time=0.029..0.029 rows=730 loops=1)"
   "Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))"
   "Planning Time: 0.253 ms"
   "Execution Time: 1.536 ms"
   ```
   
   *Объясните результат:*
   BRIN индекс смог вынести полезную информцию из блоков, которая была использована для поиска 