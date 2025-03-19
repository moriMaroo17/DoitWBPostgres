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

> По итогу в дирректории с логами я ничего не нашел, и пришлось откатывать транзакции и идти в `/var/lib/postgresql/`
Затем ставить nano `apt install nano`, после чего 
редактировать файл `postgresql.conf` следующий образом:

```
    #------------------------------------------------------------------------------
    # REPORTING AND LOGGING
    #------------------------------------------------------------------------------

    # - Where to Log -

    log_destination = 'stderr'             # Valid values are combinations of
                                            # stderr, csvlog, jsonlog, syslog, and
                                            # eventlog, depending on platform.
                                            # csvlog and jsonlog require
                                            # logging_collector to be on.

    # This is used when logging to stderr:
    logging_collector = off         # Enable capturing of stderr, jsonlog,
                                            # and csvlog into log files. Required
                                            # to be on for csvlogs and jsonlogs.
                                            # (change requires restart)

    # These are only used if logging_collector is on:
    log_directory = 'log'                   # directory where log files are written,
                                            # can be absolute or relative to PGDATA
    log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log' # log file name pattern,
                                                            # can include strftime() escapes
```

А именно в секции reporting and logging раскоментил следующие строки

После чего рестартанул докер контейнер. Проверил логгинг (судя по всему теперь должен работать)

```sql
    show logging_collector;
    logging_collector 
    -------------------
     on
    (1 row)
```

После этого я повторил весь процесс и уже нашел лог файл
```sh
    postgres@bb87836ca7ba$ cd var/lib/postgresql/log
    postgres@bb87836ca7ba$ ls
    postgresql-2025-03-19_141206.log
    postgres@bb87836ca7ba$ cat postgresql-2025-03-19_141206.log

    ...
    2025-03-19 14:16:06.363 UTC [86] ERROR:  deadlock detected
    2025-03-19 14:16:06.363 UTC [86] DETAIL:  Process 86 waits for ShareLock on transaction 791; blocked by process 88.
        Process 88 waits for ShareLock on transaction 792; blocked by process 86.
        Process 86: update accounts set amount = amount + 1 where id = 1;
        Process 88: update accounts set amount = amount + 1 where id = 2;
    2025-03-19 14:16:06.363 UTC [86] HINT:  See server log for query details.
    2025-03-19 14:16:06.363 UTC [86] CONTEXT:  while updating tuple (0,1) in relation "accounts"
    2025-03-19 14:16:06.363 UTC [86] STATEMENT:  update accounts set amount = amount + 1 where id = 1;
    2025-03-19 14:16:13.203 UTC [73] ERROR:  syntax error at or near "exit" at character 1
    2025-03-19 14:16:13.203 UTC [73] STATEMENT:  exit();
    ...

Ну и уже в логах видим кто кого и чем заблокировал