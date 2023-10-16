___
**Создаем новый кластер и работаем в нем**
```commandline
~$ sudo -u postgres pg_createcluster 15 main
~$ sudo pg_ctlcluster 15 main start
```

**Создаем базу и наполняем ее данными**
```commandline
~$ sudo -u postgres psql -c "create database otus;"
CREATE DATABASE
~$ sudo -u postgres psql -c "create table test as 
select generate_series as id
	, generate_series::text || (random() * 10)::text as col2 
    , (array['Yes', 'No', 'Maybe'])[floor(random() * 3 + 1)] as is_okay
from generate_series(1, 50000);"
SELECT 50000
```
**Проверяем explain до создания индекса**
```commandline
~$ sudo -u postgres psql -c "explain select id from test where id = 1;"

                       QUERY PLAN
--------------------------------------------------------
 Seq Scan on test  (cost=0.00..789.94 rows=163 width=4)
   Filter: (id = 1)
(2 rows)

```
___
- [x] Создать индекс к какой-либо из таблиц вашей БД
___
**Выполняем**
```commandline
~$ sudo -u postgres psql -c "create index idx_test_id on test(id);"
CREATE INDEX
```
___
- [x] Прислать текстом результат команды explain, в которой используется данный индекс
___
**Выполняем**
```commandline
~$ sudo -u postgres psql -c "explain select id from test where id = 1;"

                                 QUERY PLAN
-----------------------------------------------------------------------------
 Index Only Scan using idx_test_id on test  (cost=0.29..4.31 rows=1 width=4)
   Index Cond: (id = 1)
(2 rows)

```
___

- [x] Реализовать индекс для полнотекстового поиска
___
**Удалил предыдущую таблицу, так как она нам больше не 
подходит, создадим новую и наполним подходящими данными**
```commandline
~$ sudo -u postgres psql -c "drop table test;"                    
DROP TABLE
~$ sudo -u postgres psql -c "create table orders (id int,user_id int,order_date date,status text,some_text text);"
CREATE TABLE
~$ sudo -u postgres psql -c "insert into orders(id, user_id, order_date, status, some_text)
select generate_series, (random() * 70), date'2019-01-01' + (random() * 300)::int as order_date
        , (array['returned', 'completed', 'placed', 'shipped'])[(random() * 4)::int]
        , concat_ws(' ', (array['go', 'space', 'sun', 'London'])[(random() * 5)::int]
            , (array['the', 'capital', 'of', 'Great', 'Britain'])[(random() * 6)::int]
            , (array['some', 'another', 'example', 'with', 'words'])[(random() * 6)::int]
            )
from generate_series(100001, 1000000);"
INSERT 0 900000
~$ sudo -u postgres psql -c "alter table orders add column some_text_lexeme tsvector;"
ALTER TABLE
~$ sudo -u postgres psql -c "update orders set some_text_lexeme = to_tsvector(some_text);"
UPDATE 900000
```
**Таблица создана и заполнена, перед созданием индекса попробуем проверить как будет выполняться запрос сейчас**
```commandline
~$ sudo -u postgres psql -c "explain select some_text, to_tsvector(some_text) @@ to_tsquery('britains') from orders;"
                                   QUERY PLAN
--------------------------------------------------------------------------------
 Gather  (cost=1000.00..708543.15 rows=2200388 width=15)
   Workers Planned: 2
   ->  Parallel Seq Scan on orders  (cost=0.00..487504.35 rows=916828 width=15)
 JIT:
   Functions: 2
   Options: Inlining true, Optimization true, Expressions true, Deforming true
(6 rows)
```
**Создаем индекс для полнотекстового поиска и повторяем запрос**
```commandline
~$ sudo -u postgres psql -c "CREATE INDEX search_index_ord ON orders USING GIN (some_text_lexeme);"
CREATE INDEX
```
**Теперь индекс создан и пробуем снова провести поиск и увидеть разницу**
```commandline
~$ sudo -u postgres psql -c "explain select some_text, to_tsvector(some_text) @@ to_tsquery('britains') from orders;"

                                   QUERY PLAN                                   
---------------------------------------------------------------------------------
 Gather  (cost=1000.00..300817.50 rows=900000 width=15)
   Workers Planned: 2
   ->  Parallel Seq Scan on orders  (cost=0.00..209817.50 rows=375000 width=15)
 JIT:
   Functions: 2
   Options: Inlining false, Optimization false, Expressions true, Deforming true
(6 rows)
```
**Колличество строк при втором запуске explain сократилос с `2200388` до `900000`**
___
- [x] Реализовать индекс на часть таблицы 
___
**Пересоздаем таблицу и наполняем данными**
```commandline
~$ sudo -u postgres psql -c "drop table orders;" 
DROP TABLE
~$ sudo -u postgres psql -c "create table test as 
select generate_series as id
	, generate_series::text || (random() * 10)::text as col2 
    , (array['Yes', 'No', 'Maybe'])[floor(random() * 3 + 1)] as is_okay
from generate_series(1, 50000);"
SELECT 50000
```
**Для получения данных для сравнения посмотрим результат explain до создания индекса**
```commandline
~$ sudo -u postgres psql -c "explain select * from test where id < 50;"
                       QUERY PLAN
---------------------------------------------------------
 Seq Scan on test  (cost=0.00..1008.00 rows=47 width=31)
   Filter: (id < 50)
(2 rows) 
```
**Создаем индекс на часть таблицы и повторяем explain**
```commandline
~$ sudo -u postgres psql -c "create index idx_test_id_100 on test(id) where id < 100;"
CREATE INDEX
~$  sudo -u postgres psql -c "explain select * from test where id < 50;"
                                  QUERY PLAN
------------------------------------------------------------------------------
 Index Scan using idx_test_id_100 on test  (cost=0.14..8.96 rows=47 width=31)
   Index Cond: (id < 50)
(2 rows)
```
___
- [x] Создать индекс на несколько полей
___
**Пересоздаем таблицу и наполняем данными**
```commandline
~$ sudo -u postgres psql -c "drop table test;"                    
DROP TABLE
~$ sudo -u postgres psql -c "create table test as
select generate_series as id
	, generate_series::text || (random() * 10)::text as col2
    , (array['Yes', 'No', 'Maybe'])[floor(random() * 3 + 1)] as is_okay
from generate_series(1, 50000);" 
SELECT 50000
```
**Для получения данных для сравнения посмотрим результат explain до создания индекса**
```commandline
~$ sudo -u postgres psql -c "explain select * from test where id = 1 and is_okay = 'True';"
                       QUERY PLAN
--------------------------------------------------------
 Seq Scan on test  (cost=0.00..1133.00 rows=1 width=31)
   Filter: ((id = 1) AND (is_okay = 'True'::text))
(2 rows) 
```
**Создаем индекс на часть таблицы и повторяем explain**
```commandline
~$ sudo -u postgres psql -c "create index idx_test_id_is_okay on test(id, is_okay);"
CREATE INDEX
~$ sudo -u postgres psql -c "explain select * from test where id = 1 and is_okay = 'True';"

                                   QUERY PLAN                                   
---------------------------------------------------------------------------------
 Index Scan using idx_test_id_is_okay on test  (cost=0.29..6.06 rows=1 width=31)
   Index Cond: ((id = 1) AND (is_okay = 'True'::text))
(2 rows)

```
