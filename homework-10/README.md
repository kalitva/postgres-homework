### Репликация

1. На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение
    - Сразу создадим два кластера
        ```shell
        sudo pg_createcluster 15 vm1 --start
        sudo pg_createcluster 15 vm2 --start
        ```
    - Зайдем в каждый из них, установим настройку `wal_level` и пароль
        ```sql
        alter system set wal_level = logical
        \password
        postgres
        ```
    - Перезапустим кластеры чтобы настройка `wal_level` сработала
        ```shell
        sudo pg_ctlcluster 15 vm1 restart
        sudo pg_ctlcluster 15 vm2 restart
        ```
    - Подключимся к vm1, создадим бд и таблицы
        ```sql
        create database vm1;
        \c vm1
        create table test1(id int primary key, name varchar);
        create table test2(id int primary key, name varchar);
        ```

2. На 2 ВМ создаем таблицы test2 для записи, test для запросов на чтение.
    ```sql
    create database vm2;
    \c vm2
    create table test1(id int primary key, name varchar);
    create table test2(id int primary key, name varchar);
    ```

3. Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 c ВМ №2
    - Подключимся к vm1 и cоздадим публикацию таблицы test1
        ```sql
        create publication test1_pub for table test1;
        ```
    - На vm2 создадим публикацию таблицы test2
        ```sql
        create publication test2_pub for table test2;
        ```
    - Подписка на test2 (vm1)
        ```sql
        create subscription vm1_test2_sub
        connection 'host=localhost port=5433 user=postgres password=postgres dbname=vm2' 
        publication test2_pub with (copy_data = true);
        ```

4. Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1.
    - Выполним команды
        ```sql
        create subscription test1_sub
        connection 'host=localhost port=5432 user=postgres password=postgres dbname=vm1' 
        publication test1_pub with (copy_data = true);
        ```
    - Убедимся, что все работает: вставим в таблицу test1 на vm1 данные и прочитаем на vm2. 
    Аналогично для таблицы test2 - вставим данные на vm2 и прочитаем из vm1
        ```shell
        vm1=# insert into test1 values(1, 'a'), (2, 'b'), (3, 'c');     │vm2=# select * from test1;
        INSERT 0 3                                                      │ id | name 
        vm1=# select * from test2;                                      │----+------
         id | name                                                      │  1 | a
        ----+------                                                     │  2 | b
          1 | x                                                         │  3 | c
          2 | y                                                         │(3 rows)
          3 | z                                                         │
        (3 rows)                                                        │vm2=# insert into test2 values(1, 'x'), (2, 'y'), (3, 'z');
                                                                        │INSERT 0 3
        vm1=#                                                           │vm2=# 
        ```
    - Очистим таблицы чтобы не было лишней путаницы далее
        ```sql
        delete from test1; -- на vm1
        delete from test2; -- на vm2
        ```
5. 3 ВМ использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ).
    - Создадим новый кластер
        ```shell
        sudo pg_createcluster 16 vm3 --start
        ```
    - Подключимся новому кластеру, создадим бд и таблицы
        ```sql
        create database vm3;
        \c vm3
        create table test1(id int primary key, name varchar);
        create table test2(id int primary key, name varchar);
        ```
    - Создадим подписки  
        ```sql
        create subscription vm3_test1_sub
        connection 'host=localhost port=5432 user=postgres password=postgres dbname=vm1' 
        publication test1_pub with (copy_data = true);

        create subscription vm3_test2_sub
        connection 'host=localhost port=5433 user=postgres password=postgres dbname=vm2' 
        publication test2_pub with (copy_data = true);
        ```
