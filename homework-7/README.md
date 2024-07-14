### Механизм блокировок

1. Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых
более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.
    - Установим параметры
        ```sql
        alter system set log_lock_waits = on;
        alter system set deadlock_timeout = 200;
        select pg_reload_conf();
        ```
    - Откроем сессию, создадим таблицу и наполним данными
        ```sql
        create table accounts (
            id int primary key,
            name varchar,
            amount int
        );

        insert into accounts values
            (1, 'vasya', 100),
            (2, 'petya', 200),
            (3, 'masha', 300);
        ```
    - Начнем транзакцию и изменим счет Васи
        ```sql
        begin;
        update accounts set amount = amount + 10 where id = 1;
        ```
    - Откроем вторую сессию, откроем вторую транзакцию и попробуем изменить счет Васи
        ```sql
        begin;
        update accounts set amount = amount + 10 where id = 1;
        ```
    - Закоммитим транзакцию в первой сессии (200 миллисекунд точно прошли) и убедимся, что таймаут
    залогировался
        ```shell
        $ sudo cat /var/log/postgresql/postgresql-15-main.log | grep 'postgres@postgres LOG'
        2024-07-14 11:20:36.817 MSK [419487] postgres@postgres LOG:  process 419487 still waiting for ShareLock on transaction 448344 after 200.097 ms
        2024-07-14 11:20:42.535 MSK [419487] postgres@postgres LOG:  process 419487 acquired ShareLock on transaction 448344 after 5918.347 ms
        ```
2. Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах.
Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите
список блокировок и объясните, что значит каждая.     
    - Начнем транзакцию в первой сессии(pid 432446), в pg_locks появилась исключительная блокировка 
    Exclusive для так называемого виртуального айди транзакции. Такой айди локальный для бэкенд 
    процесса, используется для read-only транзакций и помогает сэкономить номер для "нормальной" 
    транзакции
        ```sql
        postgres=# select * from pg_locks where pid = 432446;
          locktype  | database | relation | page | tuple | virtualxid | transactionid | classid | objid | objsubid | virtualtransaction |  pid   |     mode      | granted | fastpath | waitstart
        ------------+----------+----------+------+-------+------------+---------------+---------+-------+----------+--------------------+--------+---------------+---------+----------+-----------
         virtualxid |          |          |      |       | 3/76       |               |         |       |          | 3/76               | 432446 | ExclusiveLock | t       | t        |
        ```
    - Изменим счет Пети
        ```sql
        begin;
        update accounts set amount = amount + 20 where id = 2;
        ```
    - Появились новые блокировки. Добавилась исключительная блокировка на айди транзакции так как
    начались изменяться данные, этот айди глобальный, то есть актуален для всех процессов. Также
    добавились две блокировки RowExclusive - для строки таблицы и индекса
        ```sql
        postgres=# select * from pg_locks where pid = 432556;                                                                                                                                                                
           locktype    | database | relation | page | tuple | virtualxid | transactionid | classid | objid | objsubid | virtualtransaction |  pid   |       mode       | granted | fastpath |           waitstart            
        ---------------+----------+----------+------+-------+------------+---------------+---------+-------+----------+--------------------+--------+------------------+---------+----------+------------------------------- 
         relation      |        5 |    16444 |      |       |            |               |         |       |          | 4/453              | 432556 | RowExclusiveLock | t       | t        |                                
         relation      |        5 |    16439 |      |       |            |               |         |       |          | 4/453              | 432556 | RowExclusiveLock | t       | t        |                                
         virtualxid    |          |          |      |       | 4/453      |               |         |       |          | 4/453              | 432556 | ExclusiveLock    | t       | t        |                                
         transactionid |          |          |      |       |            |        448373 |         |       |          | 4/453              | 432556 | ExclusiveLock    | t       | f        |                                
         tuple         |        5 |    16439 |    0 |    37 |            |               |         |       |          | 4/453              | 432556 | ExclusiveLock    | t       | f        |                                
         transactionid |          |          |      |       |            |        448372 |         |       |          | 4/453              | 432556 | ShareLock        | f       | f        | 2024-07-14 22:06:02.503707+03  
        (6 rows)
    - Во второй сессии (pid = 432556) откроем транзакцию и также изменим счет Пети
        ```sql
        begin;
        update accounts set amount = amount + 20 where id = 2;
        ```
    - В этой сессии появились такие же 4 блокировки как и для предыдущей. Кроме этого процесс 
    пытается захватить транзакцию из предыдущей сессии (`granted = false`) в режиме Shared, ожидая 
    освобождения ресурса. Также захвачена блокировка tuple текущей версии строки в режиме Exclusive
        ```sql
        postgres=# select * from pg_locks where pid = 432556;                                                                                                                                                                
           locktype    | database | relation | page | tuple | virtualxid | transactionid | classid | objid | objsubid | virtualtransaction |  pid   |       mode       | granted | fastpath |           waitstart            
        ---------------+----------+----------+------+-------+------------+---------------+---------+-------+----------+--------------------+--------+------------------+---------+----------+------------------------------- 
         relation      |        5 |    16444 |      |       |            |               |         |       |          | 4/452              | 432556 | RowExclusiveLock | t       | t        |                                
         relation      |        5 |    16439 |      |       |            |               |         |       |          | 4/452              | 432556 | RowExclusiveLock | t       | t        |                                
         virtualxid    |          |          |      |       | 4/452      |               |         |       |          | 4/452              | 432556 | ExclusiveLock    | t       | t        |                                
         tuple         |        5 |    16439 |    0 |    31 |            |               |         |       |          | 4/452              | 432556 | ExclusiveLock    | t       | f        |                                
         transactionid |          |          |      |       |            |        448369 |         |       |          | 4/452              | 432556 | ExclusiveLock    | t       | f        |                                
         transactionid |          |          |      |       |            |        448368 |         |       |          | 4/452              | 432556 | ShareLock        | f       | f        | 2024-07-14 22:01:19.138083+03  
        ```
    - Сделаем те же дейcтвия в третьей сессии (pid = 433082)
        ```sql
        begin;
        update accounts set amount = amount + 20 where id = 2;
        ```
    - Появились такие же 4 блокировки как в первой сессии, помио них, блокировка `tuple` на версию
    строки из второй сессии в режиме Exclusive ожидает захвата (`granted = false`)
        ```sql
         postgres=# select * from pg_locks where pid = 433082;                                                                                                                                                                
           locktype    | database | relation | page | tuple | virtualxid | transactionid | classid | objid | objsubid | virtualtransaction |  pid   |       mode       | granted | fastpath |           waitstart            
        ---------------+----------+----------+------+-------+------------+---------------+---------+-------+----------+--------------------+--------+------------------+---------+----------+------------------------------- 
         relation      |        5 |    16444 |      |       |            |               |         |       |          | 5/548              | 433082 | RowExclusiveLock | t       | t        |                                
         relation      |        5 |    16439 |      |       |            |               |         |       |          | 5/548              | 433082 | RowExclusiveLock | t       | t        |                                
         virtualxid    |          |          |      |       | 5/548      |               |         |       |          | 5/548              | 433082 | ExclusiveLock    | t       | t        |                                
         tuple         |        5 |    16439 |    0 |    37 |            |               |         |       |          | 5/548              | 433082 | ExclusiveLock    | f       | f        | 2024-07-14 22:07:13.411769+03  
         transactionid |          |          |      |       |            |        448374 |         |       |          | 5/548              | 433082 | ExclusiveLock    | t       | f        |                                
        (5 rows)
        ```
3. Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, 
изучая журнал сообщений?
    - Исполним update в трех сессиях
        ```sql
        --session 1
        update accounts set amount = amount + 10 where id = 1;
        --session 2
        update accounts set amount = amount + 10 where id = 2;
        --session 3
        update accounts set amount = amount + 10 where id = 3;
        --session 1
        update accounts set amount = amount + 10 where id = 2;
        --session 2
        update accounts set amount = amount + 10 where id = 3;
        --session 3
        update accounts set amount = amount + 10 where id = 1;
        ```
    - В последнем запросе Постгрес выдает сообщение о дедлоке
        ```sql
        ERROR:  deadlock detected
        DETAIL:  Process 433082 waits for ShareLock on transaction 448399; blocked by process 432446.
        Process 432446 waits for ShareLock on transaction 448400; blocked by process 432556.
        Process 432556 waits for ShareLock on transaction 448401; blocked by process 433082.
        HINT:  See server log for query details.
        CONTEXT:  while updating tuple (0,19) in relation "accounts"
        ```
    - В логах сервера есть подробнсти
        ```shell
        $ sudo tail -20 /var/log/postgresql/postgresql-15-main.log
        ...
        2024-07-15 00:07:53.092 MSK [433082] postgres@postgres STATEMENT:  update accounts set amount = amount + 10 where id = 1;
        2024-07-15 00:07:53.092 MSK [433082] postgres@postgres ERROR:  deadlock detected
        2024-07-15 00:07:53.092 MSK [433082] postgres@postgres DETAIL:  Process 433082 waits for ShareLock on transaction 448399; blocked by process 432446.
                Process 432446 waits for ShareLock on transaction 448400; blocked by process 432556.
                Process 432556 waits for ShareLock on transaction 448401; blocked by process 433082.
                Process 433082: update accounts set amount = amount + 10 where id = 1;
                Process 432446: update accounts set amount = amount + 10 where id = 2;
                Process 432556: update accounts set amount = amount + 10 where id = 3;
        2024-07-15 00:07:53.092 MSK [433082] postgres@postgres HINT:  See server log for query details.
        2024-07-15 00:07:53.092 MSK [433082] postgres@postgres CONTEXT:  while updating tuple (0,19) in relation "accounts"
        2024-07-15 00:07:53.092 MSK [433082] postgres@postgres STATEMENT:  update accounts set amount = amount + 10 where id = 1;
        2024-07-15 00:07:53.092 MSK [432556] postgres@postgres LOG:  process 432556 acquired ShareLock on transaction 448401 after 9457.699 ms
        2024-07-15 00:07:53.092 MSK [432556] postgres@postgres CONTEXT:  while updating tuple (0,62) in relation "accounts"
        2024-07-15 00:07:53.092 MSK [432556] postgres@postgres STATEMENT:  update accounts set amount = amount + 10 where id = 3;
        ...
        ```
4. Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы 
(без where), заблокировать друг друга?

