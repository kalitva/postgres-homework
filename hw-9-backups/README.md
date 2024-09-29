### Бэкапы

1. Создаем ВМ/докер c ПГ.
    ```shell
    sudo pg_createcluster 15 main
    sudo pg_ctlcluster 15 main start
    ```

2. Создаем БД, схему и в ней таблицу.
    ```sql
    create database otus;
    \c otus;
    create schema otus;
    set search_path to otus;
    create table users (id int, name varchar);
    create table accounts (id int, user_id int, amount int);
    ```

3. Заполним таблицы автосгенерированными 100 записями.
    ```sql
    insert into users select generate_series(1,10) id, md5(random()::text) name;
    insert into accounts 
    select 
        generate_series(1,10) id, 
        random() * 10 user_id, 
        random() * 1000 amount;
    ```

4. Под линукс пользователем Postgres создадим каталог для бэкапов
    ```shell
    sudo -u postgres mkdir /tmp/backups
    ```

5. Сделаем логический бэкап используя утилиту COPY
    ```sql
    \copy accounts to '/tmp/backups/accounts.sql';
    ```

6. Восстановим в 2 таблицу данные из бэкапа.
    ```sql
    truncate table accounts;
    \copy accounts from '/tmp/backups/accounts.sql';
    ```

7. Используя утилиту pg_dump создадим бэкап в кастомном сжатом формате двух таблиц
    ```shell
    sudo -i -u postgres
    pg_dump -d otus --create -Fc > /tmp/backups/otus_dump.gz 
    ```
8. Используя утилиту pg_restore восстановим в новую БД только вторую таблицу!
   ```shell
   createdb otus2
   echo 'create schema otus;' | psql -d otus2
   pg_restore -d otus2 -t accounts /tmp/backups/otus_dump.gz
   ``` 
