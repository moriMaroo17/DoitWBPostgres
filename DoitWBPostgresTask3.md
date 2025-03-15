# Задание 3

1. Создаем таблицу

    ```sql
    CREATE TABLE IF NOT EXISTS test(i text);
    ```
2. Заполняем данными
    
    ```sql
    INSERT INTO test SELECT s.id FROM generate_series(1,1000000) AS s(id);
    ```

3. Смотрим размер файла с таблицей

    ```sql
    \dt+ test
                                  List of relations
    Schema | Name | Type  |  Owner   | Persistence | Access method | Size  | Description 
    --------+------+-------+----------+-------------+---------------+-------+-------------
    public | test | table | postgres | permanent   | heap          | 35 MB | 
    (1 row)
    ```

    ```sql
    SELECT
    n.nspname || '.' || c.relname AS table_name,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size,
    pg_size_pretty(pg_total_relation_size(c.reltoastrelid)) AS toast_size
    FROM pg_class c
    JOIN pg_namespace n
        ON c.relnamespace = n.oid
    WHERE
        relname = 'test';
     table_name  | total_size | toast_size 
    -------------+------------+------------
     public.test | 35 MB      | 8192 bytes
    (1 row)
    ```

4. 5 раз обновляем все строчки
    ```sql
    UPDATE test SET i = i || 'a';
    UPDATE test SET i = i || 'b';
    UPDATE test SET i = i || 'c';
    UPDATE test SET i = i || 'd';
    UPDATE test SET i = i || 'e';
    ```

5. Смотрим количество мертвых строчек и когда последний раз приходил автовакум

    ```sql
    SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_tables WHERE relname = 'test';
    relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
    ---------+------------+------------+--------+-------------------------------
    test    |    1000000 |    4999870 |    499 | 2025-03-13 11:02:48.965523+00
    (1 row)
    ``` 

6. Подождать пока придет автовакуум (он пришел)

    ```sql
    SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_tables WHERE relname = 'test';
    relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
    ---------+------------+------------+--------+-------------------------------
    test    |     999897 |          0 |      0 | 2025-03-13 15:22:35.600942+00
    (1 row)
    ```

7. Снова 5 раз обновляем строчки и смотрим размер

    ```sql
    UPDATE test SET i = i || 'a';
    UPDATE test SET i = i || 'b';
    UPDATE test SET i = i || 'c';
    UPDATE test SET i = i || 'd';
    UPDATE test SET i = i || 'e';
    ```

    ```sql
    \dt+ test
                                   List of relations
    Schema | Name | Type  |  Owner   | Persistence | Access method |  Size  | Description 
    --------+------+-------+----------+-------------+---------------+--------+-------------
    public | test | table | postgres | permanent   | heap          | 260 MB | 
    (1 row)
    ```

8. Гасим автовакуум на нашей таблице

    ```sql
    ALTER TABLE test SET (autovacuum_enabled = off);
    ```

8. 10 раз обновляем все строчки

    ```sql
    update test set i = i || 'z';
    update test set i = i || '1';
    update test set i = i || '2';
    update test set i = i || '3';
    update test set i = i || '4';
    update test set i = i || '5';
    update test set i = i || '6';
    update test set i = i || '7';
    update test set i = i || '8';
    update test set i = i || '9';
    ```

9. Смотрим размер файла с таблицей

    ```sql
    \dt+ test
                                   List of relations
    Schema | Name | Type  |  Owner   | Persistence | Access method |  Size  | Description 
    --------+------+-------+----------+-------------+---------------+--------+-------------
    public | test | table | postgres | permanent   | heap          | 952 MB | 
    (1 row)
    ```

10. Объяснение.
    Таблицу раздуло до почти 1Гб так как автовакуум был выключен и перестал чистить мертвые строки. 

11. Включаем автовакум

    ```sql
    ALTER TABLE test SET (autovacuum_enabled = on);
    ```