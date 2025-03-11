# Эпиграф

> Все делаю через докер + GUI

# Подготовка

1. Стянул постгрю с докерхаба

`$ docker run -p 5432:5432 --name some-postgres -e POSTGRES_PASSWORD=mysecretpassword -d postgres
`

2. В контейнере ставлю ssh и запускаю его

`$ apt update && apt install ssh`
`$ service ssh restart`

# Задание

1. Открываю два терминала и с каждого прыгаю в контейнер под пользаком postgres и запускаю psql

`$ ssh postgres@localhost`
`$ psql`

2. Созднаю таблицу и наполняю ее данными
```sql
create table users (
  id serial primary key, 
  name varchar(50), 
  active bool
);
```

```sql
insert into users (name, active) 
values 
  ('tom', true), 
  ('ben', true), 
  ('ann', false);
```

3. Проверяю текущий уровень изоляции

```sql
SHOW default_transaction_isolation;
 default_transaction_isolation 
-------------------------------
 read committed
(1 row)
```

4. Начинаю новую транзакцию с дефолтным уровнем изоляции. 

    В первом окне начинаю транзакцию и добавляю новую запись

    ```sql
    begin;
    insert into users (name, active) 
    values 
        ('max', true);
    ```

    Во втором окне начинаю транзакцию и читаю записи

    ```sql
    begin;
    select * from users;

    id | name | active 
    ----+------+--------
    1 | tom  | t
    2 | ben  | t
    3 | ann  | f
    (3 rows)
    ```

    Новую запись не вижу, так как транзакция на запись еще не закомичена, а 
    дефолтный уровень изоляции не позволяет читать такие данные

5. Завершаю транзакцию в первом окне `commit;`
    и читаю еще раз во втором
    
    ```sql
    begin;
    select * from users;

    id | name | active 
    ----+------+--------
    1 | tom  | t
    2 | ben  | t
    3 | ann  | f
    4 | max  | t
    (3 rows)
    ```

    Запись видно, так как коммит на запись прошел и с текущим уровнем изоляции я могу ее прочитать.
    Заканчиваю транзакцию во втором окне `commit;`

6. Начинаю новые транзакции с уровнем repeatable read
    `begin transaction isolation level repeatable read;`

    Вставляю новые данные в первом окне

    ```sql
    insert into users (name, active) 
    values 
        ('luter', true);
    ```

    Читаю во втором

    ```sql
    select * from users;
    id | name | active 
    ----+------+--------
    1 | tom  | t
    2 | ben  | t
    3 | ann  | f
    7 | max  | t
    (4 rows)
    ```

    Новую строку не видно, так как repeatable read не позволяет "грязное" чтение

    Делаю коммит в первом окне `commit;` и читаю во втором

    ```sql
    select * from users;
    id | name | active 
    ----+------+--------
    1 | tom  | t
    2 | ben  | t
    3 | ann  | f
    7 | max  | t
    (4 rows)
    ```

    Во втором окне снова не вижу запись, так как подобное "неповторяющееся" чтение
    запрещено в repeatable read