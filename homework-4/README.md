### Работа с базами данных, пользователями и правами

1. создайте новый кластер PostgresSQL 14
```shell
$ sudo docker run -d --name pg14 -p 5432:5432 -e POSTGRES_PASSWORD=postgres postgres:14
```

2. зайдите в созданный кластер под пользователем postgres
```shell
psql -U postgres -h localhost
```

3. создайте новую базу данных testdb
```sql
create database testdb;
```

4. зайдите в созданную базу данных под пользователем postgres
```shell
\c testdb
```

5. создайте новую схему testnm
```sql
create schema testnm;
```

6. создайте новую таблицу t1 с одной колонкой c1 типа integer
```sql
create table t1 (c1 integer);
```

7. вставьте строку со значением c1=1
```sql
insert into t1 values (1);
```

8. создайте новую роль readonly
```sql
create role readonly;
```

9. дайте новой роли право на подключение к базе данных testdb
```sql
grant connect on database testdb to readonly;
```

10. дайте новой роли право на использование схемы testnm
```sql
grant usage on schema testnm to readonly;
```

11. дайте новой роли право на select для всех таблиц схемы testnm
```sql
grant select on all tables in schema testnm to readonly;
```

12. создайте пользователя testread с паролем test123
```sql
create user testread with password 'test123';
```

13. дайте роль readonly пользователю testread
```sql
grant readonly to testread;
```
14. зайдите под пользователем testread в базу данных testdb
```sql
set role testread;
```

15. сделайте select * from t1;
```sql
testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
```

16. получилось? `(нет)`

17. напишите что именно произошло в тексте домашнего задания
```
нет прав для доступа к таблице t1
```

18. у вас есть идеи почему? ведь права то дали?
```
Мы дали права на схему, но таблица t1 по умолчанию была создана в схеме public, а у роли readonly
есть права только на чтение из схемы testnm
```

19. посмотрите на список таблиц
```
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)
```

21. а почему так получилось с таблицей?
```
Так как мы не указывали схему при создании, таблица была создана в public по умолчанию
```

22. вернитесь в базу данных testdb под пользователем postgres
```sql
set role postgres;
```

23. удалите таблицу t1
```sql
drop table t1;
```

24. создайте ее заново но уже с явным указанием имени схемы testnm
```sql
create table testnm.t1 (c1 integer);
```

25. вставьте строку со значением c1=1
```sql
insert into testnm.t1 values (1);
```

26. зайдите под пользователем testread в базу данных testdb
```sql
set role testread;
```

27. сделайте select * from testnm.t1;
```sql
testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
```

28. получилось? `(нет)`

29. есть идеи почему?
```
t1 была пересоздана, а grant select on all table распространяется на таблицы, которые есть
на момент исполнения, а не на все таблицы в принципе.
```

30. как сделать так чтобы такое больше не повторялось?
```sql
alter default privileges in schema testnm grant select on tables to readonly;
```

31. сделайте select * from testnm.t1;
```sql
testdb=> select * from testnm.t1 ;
ERROR:  permission denied for table t1
```

32. получилось? `(нет)`

33. есть идеи почему?
```sql
Стэйтмент atlter default privileges распространяется на все таблицы, созданные в будущем. Нужно
еще раз выдать привилегии на чтение

set role postgres;
grant select on all tables in schema testnm to readonly;
set role testread;
```

34. сделайте select * from testnm.t1;
```sql
select * from testnm.t1;
```

35. получилось? `(да)`

37. теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);
```sql
testdb=> create table t2 (c1 integer);
CREATE TABLE
testdb=> insert into t2 values (2);
INSERT 0 1
```

38. а как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?
```
¯\_ (ツ)_/¯
```

39. есть идеи как убрать эти права?
```sql
set role postgres;
revoke all on schema public from public;
grant usage on schema public to public;
grant select on all tables in schema public to public;
set role testread;
```

41. теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);
```sql
testdb=> create table t3(c1 integer);
ERROR:  permission denied for schema public

testdb=> insert into t2 values (2);
INSERT 0 1
```

42. расскажите что получилось и почему
```
Создавать таблицы в public теперь нет возможности, но по прежнему можно вставить значения в
таблицу t2. Чтобы это исправить нужно отозвать права непосредственно у пользователя testread

testdb=> revoke all on t2 from testread;
testdb=> grant select on t2 to testread;
```
