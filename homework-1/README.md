### Работа с уровнями изоляции транзакции в PostgreSQL

```
Открыл в терминале две сессии на локально установленной postgres
```

- сделать в первой сессии новую таблицу и наполнить ее данными

```sql
create table persons(id serial, first_name text, second_name text);   │
insert into persons(first_name, second_name) values('ivan', 'ivanov');│
insert into persons(first_name, second_name) values('petr', 'petrov');│
CREATE TABLE                                                          │
INSERT 0 1                                                            │
INSERT 0 1                                                            │
```

- посмотреть текущий уровень изоляции: show transaction isolation level
```sql
show transaction isolation level;                                     │
transaction_isolation                                                 │
-----------------------                                               │
 read committed                                                       │
(1 row)                                                               │
```
- начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции

```sql
begin;                                                                │
                                                                      │ begin;
```
- в первой сессии добавить новую запись
```sql
insert into persons(first_name, second_name)                          │
values('sergey', 'sergeev');                                          │
```
- сделать select from persons во второй сессии
```sql
                                                                      │ select * from persons;
                                                                      │  id | first_name | second_name
                                                                      │ ----+------------+-------------
                                                                      │   1 | ivan       | ivanov
                                                                      │   2 | petr       | petrov
                                                                      │ (2 rows)
                                                                      │ -- не видно записи из другой транзакции
                                                                      │ -- так как уровень read committed
                                                                      │ -- позволяет читать только зафиксированные
                                                                      │ -- изменения
```
- завершить первую транзакцию
```sql
commit;                                                               │
```
- сделать select from persons во второй сессии
```sql
                                                                      │ select * from persons;
                                                                      │  id | first_name | second_name
                                                                      │ ----+------------+-------------
                                                                      │   1 | ivan       | ivanov
                                                                      │   2 | petr       | petrov
                                                                      │   4 | sergey     | sergeev
                                                                      │  (3 rows)
                                                                      │ -- сейчас в другой транзакции изменения
                                                                      │ -- зафиксированы и здесь видна новая
                                                                      │ -- запись
```
- завершите транзакцию во второй сессии
```sql
                                                                      │ commit;
```
- начать новые но уже repeatable read транзации
```sql
begin;                                                                │
set transaction isolation level repeatable read;                      │
                                                                      │ begin;
                                                                      │ set transaction isolation level repeatable read;
```
- в первой сессии добавить новую запись
```sql
insert into persons(first_name, second_name)                          │
 values('sveta', 'svetova');                                          │
```
- сделать select * from persons во второй сессии
```sql
                                                                      │ select * from persons;
                                                                      │  id | first_name | second_name
                                                                      │ ----+------------+-------------
                                                                      │   1 | ivan       | ivanov
                                                                      │   2 | petr       | petrov
                                                                      │   4 | sergey     | sergeev
                                                                      │ (3 rows)
                                                                      │ -- не видно новой записи так как
                                                                      │ -- так как repeatable read также
                                                                      │ -- не позволяет читать незафиксированные
                                                                      │ -- изменения
```
- завершить первую транзакцию - commit;
```sql
commit;                                                               │
```
- сделать select from persons во второй сессии
```sql
                                                                      │ select * from persons;
                                                                      │  id | first_name | second_name
                                                                      │ ----+------------+-------------
                                                                      │   1 | ivan       | ivanov
                                                                      │   2 | petr       | petrov
                                                                      │   4 | sergey     | sergeev
                                                                      │ (3 rows)
                                                                      │ -- по прежнему не видно новой записи
                                                                      │ -- так как repeatable read читает
                                                                      │ -- только те данные, которые были
                                                                      │ -- до начала транзакции
```
- завершить вторую транзакцию
```sql
                                                                      │ commit;
```
- сделать select * from persons во второй сессии
```sql
                                                                      │ select * from persons;
                                                                      │  id | first_name | second_name
                                                                      │ ----+------------+-------------
                                                                      │   1 | ivan       | ivanov
                                                                      │   2 | petr       | petrov
                                                                      │   4 | sergey     | sergeev
                                                                      │   5 | sveta      | svetova
                                                                      │ (4 rows)
                                                                      │ -- видно новую запсиь так как мы уже
                                                                      │ -- вне транзакции
```
