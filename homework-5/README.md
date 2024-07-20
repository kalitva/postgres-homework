### Настройка autovacuum с учетом особеностей производительности

- Создать инстанс ВМ с 2 ядрами и 4 Гб ОЗУ и SSD 10GB
    ```shell
    Создал ubuntu 20.04
    ```

- Установить на него PostgreSQL 15 с дефолтными настройками
    ```shell
    sudo apt install -y postgresql-common
    sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
    sudo apt install postgresql-15
    ```
- Создать БД для тестов
    ```
    sudo -i -u postgres
    pgbench -i postgres
    ```

- Запустить pgbench -c8 -P 6 -T 60 -U postgres postgres
    ```shell
    postgres@ubuntu:~$ pgbench -c8 -P 6 -T 60 -U postgres postgres
    ...
    number of transactions actually processed: 17136
    number of failed transactions: 0 (0.000%)
    latency average = 27.952 ms
    latency stddev = 14.832 ms
    initial connection time = 42.277 ms
    tps = 285.613157 (without initial connection time)
    ```

- Применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла
    ```
    Применил и перезапустил кластер
    ```

- Протестировать заново
    ```shell
    postgres@ubuntu:~$ pgbench -c8 -P 6 -T 60 -U postgres postgres
    ...
    number of transactions actually processed: 17800
    number of failed transactions: 0 (0.000%)
    latency average = 26.906 ms
    latency stddev = 14.055 ms
    initial connection time = 43.613 ms
    tps = 296.699684 (without initial connection time)
    ```

- Что изменилось и почему?
    ```
    Дефолтные параметры постгрес расчитаны из соображений, чтобы постгрес могла работать на не самых
    мощных машинах, поэтому с новыми настройками быстродействие улучшилось за счет того, что были
    переопределены параметры, например размер кэшэй и памяти для некоторых операций
    ```

- Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк
    ```sql
    create table t as select md5(random()::text) c from generate_series(1, 1000000);
    ```

- Посмотреть размер файла с таблицей
    ```sql
    postgres=# select pg_size_pretty(pg_table_size('t'));
     pg_size_pretty
    ----------------
     65 MB
    (1 row)
    ```

- 5 раз обновить все строчки и добавить к каждой строчке любой символ
    ```sql
    update t set c = concat(c, 'a');
    update t set c = concat(c, 'b');
    update t set c = concat(c, 'c');
    update t set c = concat(c, 'd');
    update t set c = concat(c, 'e');
    ```

- Посмотреть размер файла с таблицей
    ```
    postgres=# select pg_size_pretty(pg_table_size('t'));
     pg_size_pretty
    ----------------
     391 MB
    (1 row)
    ```

- Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум
    ```
    postgres=# select n_dead_tup, last_autovacuum from pg_stat_user_tables where relname = 't';
     n_dead_tup |        last_autovacuum
    ------------+-------------------------------
        4999814 | 2024-07-07 19:03:51.971326+03
    (1 row)
    ```

- Подождать некоторое время, проверяя, пришел ли автовакуум
    ```
    postgres=# select n_dead_tup, last_autovacuum from pg_stat_user_tables where relname = 't';
     n_dead_tup |        last_autovacuum
    ------------+-------------------------------
              0 | 2024-07-07 19:04:51.748697+03
    (1 row)
    ```

- 5 раз обновить все строчки и добавить к каждой строчке любой символ
    ```sql
    update t set c = concat(c, 'a');
    update t set c = concat(c, 'b');
    update t set c = concat(c, 'c');
    update t set c = concat(c, 'd');
    update t set c = concat(c, 'e');
    ```

- Посмотреть размер файла с таблицей
    ```
    postgres=# select pg_size_pretty(pg_table_size('t'));
     pg_size_pretty
    ----------------
     414 MB
    (1 row)
    ```

- Отключить Автовакуум на конкретной таблице
    ```sql
    alter table t set (autovacuum_enabled = false);
    ```

- 10 раз обновить все строчки и добавить к каждой строчке любой символ
    ```sql
    update t set c = concat(c, 'a');
    update t set c = concat(c, 'b');
    update t set c = concat(c, 'c');
    update t set c = concat(c, 'd');
    update t set c = concat(c, 'e');
    update t set c = concat(c, 'f');
    update t set c = concat(c, 'g');
    update t set c = concat(c, 'h');
    update t set c = concat(c, 'i');
    update t set c = concat(c, 'j');
    ```

- Посмотреть размер файла с таблицей
    ```
    postgres=# select pg_size_pretty(pg_table_size('t'));
     pg_size_pretty
    ----------------
     841 MB
    (1 row)
    ```

- Объясните полученный результат
    ```
    Размер таблиц растет так как при обновлении постгрес удаляет строку таблицы и вставляет
    заново. При этом служба autovacuum просто отмечает участки памяти как свободные, чтобы
    память вернулась операционной системе, нужно вызвать vacuum full
    ```

- Не забудьте включить автовакуум)
    ```sql
    alter table t set (autovacuum_enabled = true);
    ```

- Задание со \*: Написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в
искомой таблице. Не забыть вывести номер шага цикла.
    ```sql
    do
    $$
    begin
       for i in 1..10 loop
           raise notice 'counter: %', i;
           update t set c = md5(random()::text);
       end loop;
    end;
    $$;
    ```
