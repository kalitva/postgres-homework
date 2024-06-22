### Установка и настройка PostgteSQL в контейнере Docker

- создать ВМ с Ubuntu 20.04/22.04 или развернуть докер любым удобным способом
```
Установлена Ubuntu 22.04
```

- поставить на нем Docker Engine
```shell
sudo apt update
sudo apt install docker.io
```

- сделать каталог /var/lib/postgres
```shell
sudo mkdir /var/lib/postgresql/
```

- развернуть контейнер с PostgreSQL 15 смонтировав в него /var/lib/postgresql
```shell
sudo docker network create pg-homework
sudo docker run -d  \
  --name pg-server \
  --network pg-homework \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=postgres \
  -e PGDATA=/var/lib/postgresql/data/pgdata \
  -v /var/lib/postgresql/:/var/lib/postgresql/data \
  postgres:15
```

- развернуть контейнер с клиентом postgres
```shell
sudo docker run -d --name pg-client --network pg-homework -e POSTGRES_PASSWORD=postgres postgres:15
```

- подключится из контейнера с клиентом к контейнеру с сервером и сделать таблицу с парой строк
```shell
sudo docker exec -it pg-client bash
psql -h pg-server -U postgres

create table students (id integer primary key, name varchar not null);
insert into students values (1, 'vasya');
insert into students values (2, 'petya');
```

- подключится к контейнеру с сервером с ноутбука/компьютера извне инстансов GCP/ЯО/места установки докера
```shell
\q
exit

psql -h localhost -U postgres
```

- удалить контейнер с сервером
```shell
sudo docker container stop pg-server
sudo docker container rm pg-server
```

- создать его заново
```shell
sudo docker run -d  \
  --name pg-server \
  --network pg-homework \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=postgres \
  -e PGDATA=/var/lib/postgresql/data/pgdata \
  -v /opt/pg-data/:/var/lib/postgresql/data \
  postgres:15
```

- подключится снова из контейнера с клиентом к контейнеру с сервером

```shell
sudo docker exec -it pg-client bash
psql -h pg-server -U postgres
```
- проверить, что данные остались на месте

```sql
select * from students;
 id | name
----+-------
  1 | vasya
  2 | petya
(2 rows)
```
