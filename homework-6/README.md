### Работа с журналами

- Настройте выполнение контрольной точки раз в 30 секунд.
    ```
    В файле postgres.conf установил параметр checkpoint_timeout = 30s
    ```

- 10 минут c помощью утилиты pgbench подавайте нагрузку.
    ```shell
    pgbench -i postgres
    pgbench -c 8 -P 60 -T 600 -U postgres postgres
    ...
    number of transactions actually processed: 164800
    number of failed transactions: 0 (0.000%)
    latency average = 29.069 ms
    latency stddev = 18.728 ms
    initial connection time = 37.910 ms
    tps = 274.657887 (without initial connection time)
    ```

- Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем
приходится в среднем на одну контрольную точку.

    ```
    postgres=# select pg_size_pretty(sum(size)) from pg_ls_waldir();
     pg_size_pretty
    ----------------
     64 MB
    (1 row)


    -- В среднем 3Мб на контрольную точку
    ```

- Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так
произошло?
    ```shell
    Все чекпойнты выполнились с периодом в 30 секунд, кроме первого...
    Видимо первый был выполнен до того как пошла нагрузка

    vbox@ubuntu:~$ sudo cat /var/log/postgresql/postgresql-15-main.log | grep 'checkpoint starting: time'
    2024-07-10 22:38:43.020 MSK [8989] LOG:  checkpoint starting: time
    2024-07-10 22:39:43.120 MSK [8989] LOG:  checkpoint starting: time
    2024-07-10 22:40:13.104 MSK [8989] LOG:  checkpoint starting: time
    2024-07-10 22:40:43.140 MSK [8989] LOG:  checkpoint starting: time
    2024-07-10 22:41:13.108 MSK [8989] LOG:  checkpoint starting: time
    2024-07-10 22:41:43.152 MSK [8989] LOG:  checkpoint starting: time
    2024-07-10 22:42:13.131 MSK [8989] LOG:  checkpoint starting: time
    2024-07-10 22:42:43.087 MSK [8989] LOG:  checkpoint starting: time
    2024-07-10 22:43:13.140 MSK [8989] LOG:  checkpoint starting: time
    2024-07-10 22:43:43.105 MSK [8989] LOG:  checkpoint starting: time
    2024-07-10 22:44:13.100 MSK [8989] LOG:  checkpoint starting: time
    2024-07-10 22:44:43.078 MSK [8989] LOG:  checkpoint starting: time
    2024-07-10 22:45:13.087 MSK [8989] LOG:  checkpoint starting: time
    2024-07-10 22:45:43.072 MSK [8989] LOG:  checkpoint starting: time
    2024-07-10 22:46:13.120 MSK [8989] LOG:  checkpoint starting: time
    2024-07-10 22:46:43.059 MSK [8989] LOG:  checkpoint starting: time
    2024-07-10 22:47:13.128 MSK [8989] LOG:  checkpoint starting: time
    2024-07-10 22:47:43.159 MSK [8989] LOG:  checkpoint starting: time
    2024-07-10 22:48:13.112 MSK [8989] LOG:  checkpoint starting: time
    2024-07-10 22:48:43.145 MSK [8989] LOG:  checkpoint starting: time
    2024-07-10 22:49:13.108 MSK [8989] LOG:  checkpoint starting: time
    2024-07-10 22:50:43.128 MSK [8989] LOG:  checkpoint starting: time
    ```
- Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.
    ```shell
    pgbench -c 8 -P 60 -T 600 -U postgres postgres
    ...
    number of transactions actually processed: 861175
    number of failed transactions: 0 (0.000%)
    latency average = 5.495 ms
    latency stddev = 3.300 ms
    initial connection time = 33.133 ms
    tps = 1435.315892 (without initial connection time)

    Асинхронный режим дает существенный выигрыш в производительности за счет того, что запись
    кэша на диск не блокирует исполнение. Однако при этом не гарантируется восстановление бд
    после сбоя
    ```

- Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте
несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте
выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?
    - Создадим новый кластер и запустим его
        ```shell
        sudo pg_createcluster 15 checksums -- --data-checksums
        sudo pg_ctlcluster 15 checksums start
        ```
    - Создадим таблицу и вставим данные
        ```sql
        create table t (c int);
        insert into t values (1), (2), (3), (4), (5);
        ```
    - Посмотрим путь файла созданной таблицы
        ```sql
        postgres=# select pg_relation_filepath('t');
         pg_relation_filepath
        ----------------------
         base/5/16388
        (1 row)
        ```
    - Остановим кластер, откроем файл и отредактируем пару байтов
        ```shell
        sudo pg_ctlcluster 15 checksums stop
        sudo -i -u postgres
        vi /var/lib/postgresql/15/checksums/base/5/16388
        ...
        ```
    - Запустим кластер и сделаем запрос, постгрес выдает сообщение об ошибке
        ```sql
        postgres=# select * from t;
        WARNING:  page verification failed, calculated checksum 30611 but expected 61872
        ERROR:  invalid page in block 0 of relation base/5/16388
        ```
    - Чтобы проигнорировать ошибку, нужно установить параметр `set ignore_checksum_failure to on`,
     в таком случае бд выдаст предупреждение. Так делать рекоммендуется только в целях дебага, 
     иначе может привести к непредсказуемым ошибкам (хоть и в моем случае селект выдал правильные
     данные)  
