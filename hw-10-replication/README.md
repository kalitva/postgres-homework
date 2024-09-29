### Репликация

1. На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение
    - Сразу создадим два кластера
        ```shell
        sudo pg_createcluster 15 vm1 --start
        sudo pg_createcluster 15 vm2 --start
        ```
    - Зайдем в каждый из них, установим настройку `wal_level` и пароль
        ```sql
        alter system set wal_level = logical;
        \password
        postgres
        ```
    - Перезапустим кластеры чтобы настройка `wal_level` сработала
        ```shell
        sudo pg_ctlcluster 15 vm1 restart
        sudo pg_ctlcluster 15 vm2 restart
        ```
    - Подключимся к vm1, создадим таблицы
        ```sql
        create table test1(id int primary key, name varchar);
        create table test2(id int primary key, name varchar);
        ```

2. На 2 ВМ создаем таблицы test2 для записи, test для запросов на чтение.
    ```sql
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
        connection 'host=localhost port=5433 user=postgres password=postgres' 
        publication test2_pub with (copy_data = true);
        ```

4. Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1.
    - Выполним команды
        ```sql
        create subscription test1_sub
        connection 'host=localhost port=5432 user=postgres password=postgres' 
        publication test1_pub with (copy_data = true);
        ```
    - Убедимся, что все работает: вставим в таблицу test1 на vm1 данные и прочитаем на vm2. 
    Аналогично для таблицы test2 - вставим данные на vm2 и прочитаем из vm1
        ```shell
        postgres=# insert into test1 values(1, 'a');           │postgres=# select * from test1;
        INSERT 0 1                                             │ id | name 
        postgres=# select * from test2;                        │----+------
         id | name                                             │  1 | a
        ----+------                                            │(1 row)
          1 | z                                                │
        (1 row)                                                │postgres=# insert into test2 values(1, 'z');
                                                               │INSERT 0 1
        ```
    - Очистим таблицы 
        ```sql
        delete from test1; -- на vm1
        delete from test2; -- на vm2
        ```
5. 3 ВМ использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ).
    - Создадим новый кластер
        ```shell
        sudo pg_createcluster 16 vm3 --start
        ```
    - Подключимся к новому кластеру, создадим таблицы и зададим пароль
        ```sql
        create table test1(id int primary key, name varchar);
        create table test2(id int primary key, name varchar);
    
        \password
        postgres
        ```
    - Создадим подписки  
        ```sql
        create subscription vm3_test1_sub
        connection 'host=localhost port=5432 user=postgres password=postgres' 
        publication test1_pub with (copy_data = true);

        create subscription vm3_test2_sub
        connection 'host=localhost port=5433 user=postgres password=postgres' 
        publication test2_pub with (copy_data = true);
        ```

6. \* реализовать горячее реплицирование для высокой доступности на 4ВМ. Источником должна выступать 
ВМ №3. Написать с какими проблемами столкнулись.
    - На ВМ№3 нужно включить архивацию WAL файлов, требуется рестарт кластера
        ```sql
        alter system set archive_mode = on;
        alter system set archive_command = 'cp %p /var/lib/postgresql/archive/%f';
        ```
    - Создадим новый кластер
        ```shell
        sudo pg_createcluster 16 vm4
        ```
    - Удалим pgdata
        ```shell
        sudo rm -rf /var/lib/postgresql/16/vm4/
        ```
    - Пересоздадим pgdata с помощью `pg_basebackup`, скопировав данные с ВМ№3
        ```shell
        sudo -u postgres pg_basebackup -p 5434 -D /var/lib/postgresql/16/vm4 
        ```
    - Создадим файл `standby.signal` чтобы обозначить, кластер как реплику
        ```shell
        sudo -u postgres touch /var/lib/postgresql/16/vm4/standby.signal
        ```
    - Добавим настройки репликации: параметры подключения к мастеру и комманду копирования WAL
    файлов
        ```shell
        sudo -i -u postgres
        echo "primary_conninfo = 'host=localhost port=5434 user=postgres password=postgres'" >> /var/lib/postgresql/16/vm4/postgresql.auto.conf
        echo "restore_command = 'cp /var/lib/postgresql/archive/%f %p'" >> /var/lib/postgresql/16/vm4/postgresql.auto.conf
        ```
    - Запустим кластер
        ```shell
        sudo pg_ctlcluster 16 vm4 start
        ```
    - Вставим данные в test1 и test2 и убедимся, что все работает как ожидается
        ```shell
        postgres=# insert into test1 values(1, 'a');           │postgres=# select * from test1;
        INSERT 0 1                                             │ id | name 
        postgres=# select * from test2;                        │----+------
         id | name                                             │  1 | a
        ----+------                                            │(1 row)
          1 | z                                                │
        (1 row)                                                │postgres=# insert into test2 values(1, 'z');
                                                               │INSERT 0 1
                                                               │
                                                               │
        ───────────────────────────────────────────────────────┼───────────────────────────────────────────
        postgres=# select * from test1;                        │postgres=# select * from test1;
         id | name                                             │ id | name 
        ----+------                                            │----+------
          1 | a                                                │  1 | a
        (1 row)                                                │(1 row)
                                                               │
        postgres=# select * from test2;                        │postgres=# select * from test2;
         id | name                                             │ id | name 
        ----+------                                            │----+------
          1 | z                                                │  1 | z
        (1 row)                                                │(1 row)
        ```
