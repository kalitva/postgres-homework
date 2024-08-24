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
    join payment p on p.customer_id = c.customer_id
    ```

2. Реализовать левостороннее (или правостороннее) соединение двух или более таблиц
    ```sql
    select 
        f.film_id,
        f.title,
        i.inventory_id 
    from film f
    left join inventory i on i.film_id = f.film_id
    ```

3. Реализовать кросс соединение двух или более таблиц
    ```sql
    select 
        f.film_id,
        f.title,
        l.name
    from film f, language l
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
