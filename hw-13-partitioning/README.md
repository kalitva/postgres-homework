### Секционирование таблицы

```sql
-- Начнем транзакцию
begin;

-- Удалим индекс, далее его пересоздадим
alter table flights drop constraint flights_flight_no_scheduled_departure_key;

-- Удалим primary key, также удалятся foreign key, которые на него ссылаются. Foreign key 
-- восстановить не получится, primary key будет добавлен, но уже для каждой партиции отдельно
alter table flights drop constraint flights_pkey cascade;

-- Создадим новую таблицу для секционирования flights_tmp
create table flights_tmp (like flights including all) partition by range (scheduled_departure);

-- Восстановим unique constraint
alter table flights_tmp 
add constraint flights_flight_no_scheduled_departure_key unique (flight_no, scheduled_departure);

-- Создадим процедуру, которая будет создавать партиции для каждого месяца, год принимается
-- как параметр
create procedure partition_flights_by_months(year int) language plpgsql as
$body$
declare 
    month integer;
    date_from timestamp with time zone;
    table_name text;
begin
    for month in 1..12
    loop
        select concat(year, '-', lpad(month::text, 2, '0'), '-01 00:00')::timestamp at time zone 'UTC' into date_from;
        select format('flights_%s_%s', year, month) into table_name;
        execute format(
            'create table %s partition of flights_tmp for values from (''%s'') to (''%s'')',
            table_name,
            date_from,
            date_from + interval '1 month'
        );
        execute format('alter table %s add primary key (flight_id)', table_name);
    end loop;
end;
$body$;

-- Исполним процедуру для 2016 года
call partition_flights_by_months(2016);

-- Наполним партиционированную таблицу данными
insert into flights_tmp select * from flights;

-- Переименуем
alter table flights rename to flights_legacy;
alter table flights_tmp rename to flights;

-- Закоммитим транзакцию
commit;
```
