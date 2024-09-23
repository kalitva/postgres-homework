### Работа с join'ами, статистикой

Используется бд с сайта postgresqltutorial - 
https://www.postgresqltutorial.com/postgresql-getting-started/postgresql-sample-database/

1. Реализовать прямое соединение двух или более таблиц
    ```sql
    select
        c.customer_id,
        c.first_name,
        c.last_name,
        p.amount,
        p.payment_date
    from customer c
    join payment p using (customer_id)
    ```
    Запрашиваем пользователя и информацию о платежах. Будет Использоваться hash join, так как 
    выборка достаточно большая
    ```sql
                                        QUERY PLAN
    --------------------------------------------------------------------------
     Hash Join  (cost=22.48..315.02 rows=14596 width=31)
       Hash Cond: (p.customer_id = c.customer_id)
       ->  Seq Scan on payment p  (cost=0.00..253.96 rows=14596 width=16)
       ->  Hash  (cost=14.99..14.99 rows=599 width=17)
             ->  Seq Scan on customer c  (cost=0.00..14.99 rows=599 width=17)
    ```

2. Реализовать левостороннее (или правостороннее) соединение двух или более таблиц
    ```sql
    select 
        f.film_id,
        f.title,
        i.inventory_id 
    from film f
    left join inventory i using (film_id)
    ```
    Выбираем фильм и инвентарный номер, если имеется. Похожий план как и для запроса выше
    ```sql
                                  QUERY PLAN                               
    -----------------------------------------------------------------------
     Hash Right Join  (cost=110.50..193.39 rows=4581 width=23)
       Hash Cond: (i.film_id = f.film_id)
       ->  Seq Scan on inventory i  (cost=0.00..70.81 rows=4581 width=6)
       ->  Hash  (cost=98.00..98.00 rows=1000 width=19)
             ->  Seq Scan on film f  (cost=0.00..98.00 rows=1000 width=19)
    ```

3. Реализовать кросс соединение двух или более таблиц
    ```sql
    select 
        f.film_id,
        f.title,
        l.name
    from film f, language l
    ```
    Выбираем фильмы со всеми возможными языками. Используется Nested Loop, так как соединяются 
    каждая строка с каждой, нет необходимости использовать хэш
    ```sql
                                  QUERY PLAN                               
    -----------------------------------------------------------------------
     Nested Loop  (cost=0.00..174.07 rows=6000 width=103)
       ->  Seq Scan on film f  (cost=0.00..98.00 rows=1000 width=19)
       ->  Materialize  (cost=0.00..1.09 rows=6 width=84)
             ->  Seq Scan on language l  (cost=0.00..1.06 rows=6 width=84)
    ```

4. Реализовать полное соединение двух или более таблиц
    ```sql
    select
        s.first_name || ' ' ||  s.last_name as staff,
        c.first_name || ' ' ||  c.last_name as customer,
        coalesce(s.email, c.email) as email
    from staff s
    full join customer c using (email)
    ```
    Объединим клиентов и сотрудников по email. Снова используется Hash Join
    ```sql
                                 QUERY PLAN                              
    ---------------------------------------------------------------------
     Hash Full Join  (cost=1.04..24.29 rows=599 width=182)
       Hash Cond: ((c.email)::text = (s.email)::text)
       ->  Seq Scan on customer c  (cost=0.00..14.99 rows=599 width=45)
       ->  Hash  (cost=1.02..1.02 rows=2 width=334)
             ->  Seq Scan on staff s  (cost=0.00..1.02 rows=2 width=334)
    ```

5. Реализовать запрос, в котором будут использованы разные типы соединений
    ```sql
    select 
        f.title, 
        a.first_name || ' ' ||  a.last_name as actor,
        f.description,
        i.store_id
    from film f
    left join inventory i using (film_id)
    join film_actor fa using (film_id) 
    join actor a using (actor_id)
    ```
    Запросим актеров и (через промежуточную таблицу) фильмы с инвентарным номером. Используются
    Hash Join'ы
    ```sql
     Hash Join  (cost=257.15..779.52 rows=25021 width=143)
       Hash Cond: (fa.film_id = f.film_id)
       ->  Hash Join  (cost=6.50..105.76 rows=5462 width=15)
             Hash Cond: (fa.actor_id = a.actor_id)
             ->  Seq Scan on film_actor fa  (cost=0.00..84.62 rows=5462 width=4)
             ->  Hash  (cost=4.00..4.00 rows=200 width=17)
                   ->  Seq Scan on actor a  (cost=0.00..4.00 rows=200 width=17)
       ->  Hash  (cost=193.39..193.39 rows=4581 width=115)
             ->  Hash Right Join  (cost=110.50..193.39 rows=4581 width=115)
                   Hash Cond: (i.film_id = f.film_id)
                   ->  Seq Scan on inventory i  (cost=0.00..70.81 rows=4581 width=4)
                   ->  Hash  (cost=98.00..98.00 rows=1000 width=113)
                         ->  Seq Scan on film f  (cost=0.00..98.00 rows=1000 width=113)
    ```
