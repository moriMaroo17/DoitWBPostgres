# Задание 4

1. Запустил контейнер с постгрей и прыгнул туда через ssh с 2х терминалов

    ```sh
    $ ssh postgres@localhost
    $ psql
    ```

2. Создаем и заполняем таблицу
    
    ```sql
    drop table if exists accounts;
    create table accounts(id integer, amount numeric);
    insert into accounts values (1,2000.00), (2,2000.00), (3,2000.00);
    ```

3. Провоцируем блокировку

    - 1 терминал
    ```sql
    begin;
    update accounts set amount = amount + 1 where id = 1;
    ```

    - 2 терминал
    ```sql
    begin;
    update accounts set amount = amount + 1 where id = 2;
    ```

    - 1 терминал
    ```sql
    update accounts set amount = amount + 1 where id = 2;
    ```

    - 2 терминал
    ```sql
    update accounts set amount = amount + 1 where id = 1;
    ```

    Во 2 терминале ловим дедлок, получил сообщение

    ```
    ERROR:  deadlock detected
    DETAIL:  Process 81 waits for ShareLock on transaction 789; blocked by process 83.
    Process 83 waits for ShareLock on transaction 790; blocked by process 81.
    HINT:  See server log for query details.
    CONTEXT:  while updating tuple (0,1) in relation "accounts"
    ```

    В сообщении получил пиды транзакций, которые друг друга залочили.
