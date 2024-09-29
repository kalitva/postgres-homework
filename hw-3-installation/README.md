### Установка и настройка PostgreSQL

- создайте виртуальную машину c Ubuntu 20.04/22.04 LTS в GCE/ЯО/Virtual Box/докере
    ```
    Установлена ubuntu 20.04
    ```

- поставьте на нее PostgreSQL 15 через sudo apt
    ```shell
    sudo apt install -y postgresql-common
    sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
    sudo apt install postgresql-15
    ```

- проверьте что кластер запущен через sudo -u postgres pg_lsclusters
    ```shell
    vbox@ubuntu:~$ sudo -u postgres pg_lsclusters
    Ver Cluster Port Status Owner    Data directory              Log file
    15  main    5433 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log

    ```

- зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым
    ```shell
    sudo su postgres
    psql
    create table t (c integer);
    insert into t values (1);
    ```

- остановите postgres
    ```shell
    systemctl stop postgresql
    ```

- создайте новый диск к ВМ размером 10GB
    ```
    Создал
    ```

- проинициализируйте диск согласно инструкции и подмонтировать файловую систему
    ```shell
    sudo fdisk /dev/sdb
    sudo mkfs -t ext4 /dev/sdb1
    ```
- перезагрузите инстанс и убедитесь, что диск остается примонтированным (если не так смотрим в сторону fstab)

    ```shell
    vbox@ubuntu:~$ sudo -i -u root
    root@ubuntu:~# mkdir /mnt/data
    root@ubuntu:~# blkid | grep /dev/sdb1
    /dev/sdb1: UUID="7950a756-fc57-4f54-863a-de75ac380049" TYPE="ext4" PARTUUID="cff45df6-01"
    root@ubuntu:~# echo "UUID=7950a756-fc57-4f54-863a-de75ac380049 /mnt/data ext4  defaults  0  2" >> /etc/fstab
    root@ubuntu:~# reboot
    ...
    vbox@ubuntu:~$ mount | grep sdb1
    /dev/sdb1 on /mnt/data type ext4 (rw,relatime)
    ```

- сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/
    ```shell
    sudo chown -R postgres:postgres /mnt/data/
    ```

- перенесите содержимое /var/lib/postgres/15 в /mnt/data - mv /var/lib/postgresql/15/mnt/data
    ```shell
    sudo systemctl stop postgresql.service
    sudo mv /var/lib/postgresql/15/ /mnt/data/
    ```

- попытайтесь запустить кластер
    ```shell
    box@ubuntu:~$ sudo -u postgres pg_ctlcluster 15 main start
    Error: /var/lib/postgresql/15/main is not accessible or does not exist
    ```

- напишите получилось или нет и почему
    ```
    Нет, так как мы перенсли каталог с данными
    ```

- задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/15/main который
надо поменять и поменяйте его
    ```
    В файле postgres.conf поменять параметр data_directory,
    ```

- напишите что и почему поменяли
    ```
    указал новый путь с данными
    data_directory = 'mnt/data/15/main/'
    ```

- попытайтесь запустить кластер
    ```shell
    sudo -u postgres pg_ctlcluster 15 main start
    ```

- напишите получилось или нет и почему
    ```
    Сейчас постгрес стартанул и можно подключиться
    ```

- зайдите через через psql и проверьте содержимое ранее созданной таблицы
    ```shell
    vbox@ubuntu:~$ sudo su postgres
    postgres@ubuntu:/home/vbox$ psql
    psql (15.7 (Ubuntu 15.7-1.pgdg20.04+1))
    Type "help" for help.

    postgres=# select * from t;
     c
    ---
     1
    (1 row)
    ```
