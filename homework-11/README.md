### Работа с индексами

Создал таблицу и заполнил сгенерированными данными
```sql
create table songs (
    id bigint,
    name varchar,
    artist varchar,
    genre varchar,
    lyrics tsvector
);
```

- Создать индекс к какой-либо из таблиц вашей БД
    ```sql
    create index idx_songs_id on songs (id);
    ```
    По дефолту создается btree индекс    
    <br />

- Прислать текстом результат команды explain, в которой используется данный индекс
    ```sql
    explain analyze select * from songs where id = 111;
                                                          QUERY PLAN
    ----------------------------------------------------------------------------------------------------------------------
     Index Scan using idx_songs_id on songs  (cost=0.28..8.29 rows=1 width=241) (actual time=0.023..0.023 rows=0 loops=1)
       Index Cond: (id = 111)
     Planning Time: 0.104 ms
     Execution Time: 0.047 ms
   (4 rows)
    ```
    Индекс срабатывает только на оператор `=`, я полагал, что будет также сработать на условия
    с `>` и `<`...  
    <br />

- Реализовать индекс для полнотекстового поиска
    ```sql
    create index idx_song_lyrics on songs using gin (lyrics);
    ```
    Для корректной работы gin индекса поле должно бы определенного типа (в данном случае использую
    `tsvector`). Также для выборки по условию нужен особый синтаксис
    ```sql
    explain select * from songs where lyrics @@ to_tsquery('lorem');

                                          QUERY PLAN
    -------------------------------------------------------------------------------
     Bitmap Heap Scan on songs  (cost=12.29..28.54 rows=5 width=322)
       Recheck Cond: (lyrics @@ to_tsquery('lorem'::text))
       ->  Bitmap Index Scan on idx_song_lyrics  (cost=0.00..12.29 rows=5 width=0)
             Index Cond: (lyrics @@ to_tsquery('lorem'::text))
    (4 rows)
    ```

- Реализовать индекс на часть таблицы или индекс на поле с функцией
    ```sql
    create index idx_songs_genre on songs (lower(genre));
    ```
    Чтобы индекс сработал, нужно в условии использовать функцию
    ```sql
    explain select * from songs where lower(genre) = 'rock';
                                      QUERY PLAN
    ------------------------------------------------------------------------------
     Bitmap Heap Scan on songs  (cost=4.19..19.21 rows=5 width=322)
       Recheck Cond: (lower((genre)::text) = 'rock'::text)
       ->  Bitmap Index Scan on idx_songs_genre  (cost=0.00..4.19 rows=5 width=0)
             Index Cond: (lower((genre)::text) = 'rock'::text)
    (4 rows)
    ```
- Создать индекс на несколько полей
    ```sql
    create index idx_songs_name_artist on songs (name, artist);

    explain  select * from songs where name = 'ABC' and artist = 'John';
                                         QUERY PLAN                                      
    -------------------------------------------------------------------------------------
     Index Scan using idx_songs_name_artist on songs  (cost=0.28..8.29 rows=1 width=322)
       Index Cond: (((name)::text = 'ABC'::text) AND ((artist)::text = 'John'::text))
    (2 rows)
    ```
    Опять же, думал, что индекс будет срабатывать при условиях типа `where name like 'hello%'`, но
    работает только с оператором `=`
