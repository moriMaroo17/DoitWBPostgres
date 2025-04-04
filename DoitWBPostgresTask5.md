# Задание 5

1. Запустил контейнер с постгрей и прыгнул туда через ssh

    ```sh
    $ ssh postgres@localhost
    $ psql
    ```

2. Тащим тайские перевозки и заливаем в БД

    ```sh
    $ cd ~ && wget https://storage.googleapis.com/thaibus/thai_small.tar.gz && tar -xf thai_small.tar.gz && psql < thai.sql
    ```

3. Делаем запрос для использования в бенчмарке

    ```sh
    $ cat > ~/workload.sql << EOL
    \set r random(1, 5000000) 
    SELECT id, fkRide, fio, contact, fkSeat FROM book.tickets WHERE id = :r;
    EOL
    ```

4. Запускаем бенчмарк

    ```sh
    $ /usr/lib/postgresql/17/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres thai
    pgbench (17.4 (Debian 17.4-1.pgdg120+2))
    transaction type: /var/lib/postgresql/workload.sql
    scaling factor: 1
    query mode: simple
    number of clients: 8
    number of threads: 4
    maximum number of tries: 1
    duration: 10 s
    number of transactions actually processed: 415183
    number of failed transactions: 0 (0.000%)
    latency average = 0.192 ms
    initial connection time = 17.842 ms
    tps = 41586.622662 (without initial connection time)
    ```

    ```sh
    $ /usr/lib/postgresql/17/bin/pgbench -c 100 -j 4 -T 10 -f ~/workload.sql -n -U postgres thai
    pgbench (17.4 (Debian 17.4-1.pgdg120+2))
    transaction type: /var/lib/postgresql/workload.sql
    scaling factor: 1
    query mode: simple
    number of clients: 100
    number of threads: 4
    maximum number of tries: 1
    duration: 10 s
    number of transactions actually processed: 737687
    number of failed transactions: 0 (0.000%)
    latency average = 1.348 ms
    initial connection time = 97.542 ms
    tps = 74198.950032 (without initial connection time)
    ```

5. Разворачиваем pgbouncer

    >Так как делаю в докере, то ставлю через рута с помощью docker exec, по этому нет sudo

    ```sh
    $ DEBIAN_FRONTEND=noninteractive apt install -y pgbouncer

    $ service pgbouncer status
    pgbouncer is not running ... failed!

    $ service pgbouncer stop
    Stopping PgBouncer: pgbouncer.

    $ cat > temp.cfg << EOF 
    [databases]
    thai = host=127.0.0.1 port=5432 dbname=thai
    [pgbouncer]
    logfile = /var/log/postgresql/pgbouncer.log
    pidfile = /var/run/postgresql/pgbouncer.pid
    listen_addr = *
    listen_port = 6432
    auth_type = scram-sha-256
    auth_file = /etc/pgbouncer/userlist.txt
    admin_users = admindb
    EOF

    $ cat temp.cfg | sudo tee -a /etc/pgbouncer/pgbouncer.ini

    $ cat > temp2.cfg << EOF 
    "admindb" "admin123#"
    "postgres" "admin123#"
    EOF

    $ cat temp2.cfg | sudo tee -a /etc/pgbouncer/userlist.txt

    $ service pgbouncer start
    Starting PgBouncer: pgbouncer.
    ```

6. Задаем пароль
    
    > Не использую sudo так как выполняю подобные команды из под рута через docker exec

    ```sh
    $ psql -c "ALTER USER postgres WITH PASSWORD 'admin123#';"
    ```

    ```sh
    $ echo "localhost:5432:thai:postgres:admin123#" | tee -a /var/lib/postgresql/.pgpass && chmod 600 /var/lib/postgresql/.pgpass 
    ```

    ```sh
    $ chown postgres:postgres /var/lib/postgresql/.pgpass
    ```


7. Прыгаем на pgbouncer

    ```sh
    $ psql -p 6432 -h 127.0.0.1 -d thai -U postgres
    ```

    > Все работает, так что запускаем бенчмарки

8. Теперь меняем pgbouncer pool_mode и прогоняем бенч. 

    для начала выставил 
    ```
    [pgbouncer]
    pool_mode = transaction
    max_client_conn = 2000
    default_pool_size = 20
    ```

    Результаты следующие.
    - для pool_mode transaction
    
        ```sh
        $ /usr/lib/postgresql/17/bin/pgbench -c 120 -j 4 -T 10 -f ~/workload.sql -n -U postgres -p 6432 -h 127.0.0.1 thai
        Password: 
        pgbench (17.4 (Debian 17.4-1.pgdg120+2))
        transaction type: /var/lib/postgresql/workload.sql
        scaling factor: 1
        query mode: simple
        number of clients: 120
        number of threads: 4
        maximum number of tries: 1
        duration: 10 s
        number of transactions actually processed: 185925
        number of failed transactions: 0 (0.000%)
        latency average = 6.373 ms
        initial connection time = 132.231 ms
        tps = 18829.415471 (without initial connection time)
        ```

    - для pool_mode statement

        ```sh
        $ /usr/lib/postgresql/17/bin/pgbench -c 120 -j 4 -T 10 -f ~/workload.sql -n -U postgres -p 6432 -h 127.0.0.1 thai
        Password: 
        pgbench (17.4 (Debian 17.4-1.pgdg120+2))
        transaction type: /var/lib/postgresql/workload.sql
        scaling factor: 1
        query mode: simple
        number of clients: 120
        number of threads: 4
        maximum number of tries: 1
        duration: 10 s
        number of transactions actually processed: 197841
        number of failed transactions: 0 (0.000%)
        latency average = 5.976 ms
        initial connection time = 153.723 ms
        tps = 20080.149679 (without initial connection time)
        ```
    
    - для pool_mode session

        ```sh
        $ /usr/lib/postgresql/17/bin/pgbench -c 120 -j 4 -T 10 -f ~/workload.sql -n -U postgres -p 6432 -h 127.0.0.1 thai
        Password: 
        pgbench (17.4 (Debian 17.4-1.pgdg120+2))
        transaction type: /var/lib/postgresql/workload.sql
        scaling factor: 1
        query mode: simple
        number of clients: 120
        number of threads: 4
        maximum number of tries: 1
        duration: 10 s
        number of transactions actually processed: 154409
        number of failed transactions: 0 (0.000%)
        latency average = 7.686 ms
        initial connection time = 124.230 ms
        tps = 15613.362075 (without initial connection time)
        ```