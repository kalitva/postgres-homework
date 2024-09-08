### Триггеры, поддержка заполнения витрин

- Для начала заполним таблицу продаж
```sql
insert into goods_sum_mart
select g.goods_name, sum(g.goods_price * s.sales_qty) from sales s
join goods g using (goods_id)
group by g.goods_name
```

- Создадим unique constraint для имени товара таблицы отчета, он пригодится для тригерной функции
```sql
alter table goods_sum_mart add constraint goods_name_unq unique (goods_name)
```

- Создадим триггер, который будет обновлять отчет при продажах
```sql
create function update_goods_sum_mart_on_new_sales()
    returns trigger
    language plpgsql
as 
$body$
declare
    v_goods_name varchar(63);
    v_goods_price numeric(12, 2);
begin 
    select g.goods_name, g.goods_price
    into v_goods_name, v_goods_price
    from goods g
    where g.goods_id = new.goods_id;

    insert into goods_sum_mart 
    values (v_goods_name, (new.sales_qty * v_goods_price))
    on conflict (goods_name) do update 
    set sum_sale = goods_sum_mart.sum_sale + (NEW.sales_qty * v_goods_price);

    return new;
end
$body$;

create trigger update_goods_sum_mart_on_new_sales_trigger
after insert on sales
for each row execute procedure update_goods_sum_mart_on_new_sales();
```

- Создадим триггер, который будет обновлять отчет, если имя товара обновится
```sql
create function update_goods_sum_mart_goods_name()
    returns trigger
    language plpgsql
as 
$body$
begin
    update goods_sum_mart
    set goods_name = new.goods_name
    where goods_name = old.goods_name;

    return new;
end
$body$;

create trigger update_goods_sum_mart_goods_name_trigger
after update on goods
for each row execute procedure update_goods_sum_mart_goods_name();
```

- Триггер, который пересчитывает отчет, если продажа была отменена
```sql
create function update_goods_sum_mart_sum_sale()
    returns trigger
    language plpgsql
as
$body$
declare
    v_goods_name varchar(63);
    v_goods_price numeric(12, 2);
    v_sales_qty integer;
begin 
    select g.goods_name, g.goods_price, s.sales_qty
    into v_goods_name, v_goods_price, v_sales_qty
    from goods g
    join sales s using(goods_id)
    where g.goods_id = old.goods_id;

    update goods_sum_mart
    set sum_sale = goods_sum_mart.sum_sale - (v_goods_price * v_sales_qty)
    where goods_name = v_goods_name;

    return old;
end
$body$;

create trigger update_goods_sum_mart_sum_sale_trigger
after delete on sales
for each row execute procedure update_goods_sum_mart_sum_sale();
```
Здесь у нас получается могут быть несоответствие - если перед удалением продажи цена товара
будет изменена, то значение поля sum_sale будет неверным. Чтобы это исправить, можно например
хранить в `goods_sum_mart` цену проданного товара, а лучше сохранять цену, по которой продан товар
в sales

- Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" 
(кроме производительности)?

\- В отчете-таблице хранится актуальная история продаж. Например, цена товара может измениться, 
а товар может быть удален - в обеих случаях отчет-таблица будет хранить актульные данные, тогда 
как отчет-запрос будет вычислять статиску по текущей цене и без удаленных товаров
