## Задание 3

1. Создайте таблицу с большим количеством данных:
    ```sql
    CREATE TABLE test_cluster AS 
    SELECT 
        generate_series(1,1000000) as id,
        CASE WHEN random() < 0.5 THEN 'A' ELSE 'B' END as category,
        md5(random()::text) as data;
    ```

2. Создайте индекс:
    ```sql
    CREATE INDEX test_cluster_cat_idx ON test_cluster(category);
    ```

3. Измерьте производительность до кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    ```
    "Bitmap Heap Scan on test_cluster  (cost=5600.73..20262.64 rows=502233 width=39) (actual time=30.435..204.468 rows=500162 loops=1)"
    "Recheck Cond: (category = 'A'::text)"
    "Heap Blocks: exact=8334"
    "->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..5475.17 rows=502233 width=0) (actual time=28.773..28.775 rows=500162 loops=1)"
    "Index Cond: (category = 'A'::text)"
    "Planning Time: 0.749 ms"
    "Execution Time: 229.516 ms"
    ```
    
    *Объясните результат:*
    Индекс используется, но не очень эффективно, так как у нас бинарные категории   

4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*
    ```
    CLUSTER
    ```

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    ```
    "Bitmap Heap Scan on test_cluster  (cost=5600.73..20212.64 rows=502233 width=39) (actual time=31.703..164.282 rows=500162 loops=1)"
    "Recheck Cond: (category = 'A'::text)"
    "Heap Blocks: exact=4169"
    "->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..5475.17 rows=502233 width=0) (actual time=30.950..30.951 rows=500162 loops=1)"
    "Index Cond: (category = 'A'::text)"
    "Planning Time: 0.619 ms"
    "Execution Time: 188.071 ms"
    ```
    
    *Объясните результат:*
    Точно также использовался индекс 

6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*
    В данном случае использование индекса дало прирост производительности благодаря последовательному чтению 