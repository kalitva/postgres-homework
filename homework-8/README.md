### Нагрузочное тестирование и тюнинг PostgreSQL

- поставить на неё PostgreSQL 15 любым способом
    - Создал новый кластер:
        ```shell
         sudo pg_ctlcluster 15 main start
        ```
    - И запустил `pg_bench` для сравнения далее
        ```shell
        sudo -i -u postgres
        pgbench -i postgres
        pgbench -c 8 -j 4 -P 10 -T 60 -U postgres postgres
        ...
        number of failed transactions: 0 (0.000%)
        latency average = 10.718 ms
        latency stddev = 6.711 ms
        initial connection time = 18.022 ms
        tps = 746.207562 (without initial connection time)
        ```

- настроить кластер PostgreSQL 15 на максимальную производительность не обращая внимание на 
возможные проблемы с надежностью в случае аварийной перезагрузки виртуальной машины
    ```properties
    shared_buffers = 4GB
    bgwriter_delay = 1s

    fsync = off
    full_page_writes = off

    max_worker_processes = 4
    max_parallel_workers_per_gather = 2
    max_parallel_maintenance_workers = 2
    max_parallel_workers = 4
    parallel_leader_participation = on
    ```

- нагрузить кластер через утилиту через утилиту pgbench 
    ```shell
    pgbench -c 8 -j 4 -P 10 -T 60 -U postgres postgres
    ```
- написать какого значения tps удалось достичь, показать какие параметры в какие значения 
устанавливали и почему    
    <br />
    Основной прирост к производительности сделали настройки `fsync` и `full_page_writes`. Также 
    ускорило работу увеличение кэша `shared_buffers`, (по умолчанию стоит совсем небольшое 
    значение). Добавил настройки concurrency, что хорошо повлияло на `latency average`, 
    `latency stddev` и `initial connection time`. Также пробовал менять значения параметров
    `work_mem`, `maintenance_work_mem`, `wal_level` и другие - разницы либо не было, либо
    показатели были даже немного хуже, поэтому их не стал менять. В итоге результаты бенчмарка 
    получились такими:
    ```shell
    ...
    number of failed transactions: 0 (0.000%)
    latency average = 2.334 ms
    latency stddev = 1.465 ms
    initial connection time = 22.377 ms
    tps = 3425.385420 (without initial connection time)
    ```
