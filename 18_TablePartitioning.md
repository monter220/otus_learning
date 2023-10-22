- [x] Секционировать большую таблицу из демо базы flights
___
**Скачиваем с сайта https://postgrespro.com/docs/postgrespro/15/demodb-bookings-installation демо базу
`demo-small-en.zip`, извлекаем из архива и закидываем на наш сервер в папку `tmp`. Разворачиваем демо базу**
```commandline
~$ sudo -u postgres psql -f /tmp/demo-small-en-20170815.sql -U postgres
```
**Проверяем структуры БД `flights` чтобы понять как лучше провести дробление на секции**
```commandline
~$ sudo -u postgres psql -d demo -c " select * from bookings.flights limit 1;"
 flight_id | flight_no |  scheduled_departure   |   scheduled_arrival    | departure_airport | arrival_airport |  status   | aircraft_code | actual_departure | actual_arrival
-----------+-----------+------------------------+------------------------+-------------------+-----------------+-----------+---------------+------------------+----------------
      1185 | PG0134    | 2017-09-10 06:50:00+00 | 2017-09-10 11:55:00+00 | DME               | BTK             | Scheduled | 319           |                  |
(1 row)
```
**Считаю, что самым оптимальным вариантом для оценки кол-ва рейсов в месяц будет деление по 
дате отправки, для этого создадим новую версию таблицы `flights`, назовем ее  `flights_new`**

```commandline
~$ sudo -u postgres psql -d demo -c "CREATE TABLE bookings.flights_new (LIKE bookings.flights) PARTITION BY RANGE(scheduled_departure);"
CREATE TABLE
~$ sudo -u postgres psql -d demo -c "CREATE TABLE bookings.flights_new_201707 PARTITION OF bookings.flights_new FOR VALUES FROM ('2017-07-01'::timestamptz) TO ('2017-08-01'::timestamptz);"
CREATE TABLE
~$ sudo -u postgres psql -d demo -c "CREATE TABLE bookings.flights_new_201708 PARTITION OF bookings.flights_new FOR VALUES FROM ('2017-08-01'::timestamptz) TO ('2017-09-01'::timestamptz);"
CREATE TABLE
~$ sudo -u postgres psql -d demo -c "CREATE TABLE bookings.flights_new_201709 PARTITION OF bookings.flights_new FOR VALUES FROM ('2017-09-01'::timestamptz) TO ('2017-10-01'::timestamptz);"
CREATE TABLE
```
**Скопируем данные из таблицы первоисточника**
```commandline
~$ sudo -u postgres psql -d demo -c "INSERT INTO bookings.flights_new SELECT * FROM bookings.flights;"                                                                         
INSERT 0 33121
```
**Пришло время сравнения скорости запроса и точности обращения**

**Для начала выполним оценочный запрос с прогревом кеша основной таблицы(которая без дробления)**
```commandline
~$ sudo -u postgres psql -d demo -c "EXPLAIN ANALYZE SELECT count(*) FROM bookings.flights WHERE departure_airport between '2017-08-01' AND '2017-08-15' AND departure_airport='DME';"
                                                                      QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=971.62..971.63 rows=1 width=8) (actual time=30.535..30.537 rows=1 loops=1)
   ->  Seq Scan on flights  (cost=0.00..971.62 rows=1 width=0) (actual time=30.454..30.455 rows=0 loops=1)
         Filter: ((departure_airport >= '2017-08-01'::bpchar) AND (departure_airport <= '2017-08-15'::bpchar) AND (departure_airport = 'DME'::bpchar))
         Rows Removed by Filter: 33121
 Planning Time: 2.746 ms
 Execution Time: 31.148 ms

~$ sudo -u postgres psql -d demo -c "EXPLAIN ANALYZE SELECT count(*) FROM bookings.flights WHERE departure_airport between '2017-08-01' AND '2017-08-15' AND departure_airport='DME';"
                                                                      QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=971.62..971.63 rows=1 width=8) (actual time=25.037..25.039 rows=1 loops=1)
   ->  Seq Scan on flights  (cost=0.00..971.62 rows=1 width=0) (actual time=25.029..25.029 rows=0 loops=1)
         Filter: ((departure_airport >= '2017-08-01'::bpchar) AND (departure_airport <= '2017-08-15'::bpchar) AND (departure_airport = 'DME'::bpchar))
         Rows Removed by Filter: 33121
 Planning Time: 1.456 ms
 Execution Time: 25.303 ms

~$ sudo -u postgres psql -d demo -c "EXPLAIN ANALYZE SELECT count(*) FROM bookings.flights WHERE departure_airport between '2017-08-01' AND '2017-08-15' AND departure_airport='DME';"
                                                                      QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=971.62..971.63 rows=1 width=8) (actual time=24.512..24.514 rows=1 loops=1)
   ->  Seq Scan on flights  (cost=0.00..971.62 rows=1 width=0) (actual time=24.504..24.504 rows=0 loops=1)
         Filter: ((departure_airport >= '2017-08-01'::bpchar) AND (departure_airport <= '2017-08-15'::bpchar) AND (departure_airport = 'DME'::bpchar))
         Rows Removed by Filter: 33121
 Planning Time: 1.442 ms
 Execution Time: 24.767 ms
```
**В таблице без дроблений мы получили скорость выполнения запроса**
```commandline
Execution Time: 31.148 ms
Execution Time: 25.303 ms
Execution Time: 24.767 ms
```
**Так же видим, что обращение идет к полной таблице `Scan on flights`**


**Теперь повторим для обновленной таблицы этот запрос**
```commandline
~$ sudo -u postgres psql -d demo -c "EXPLAIN ANALYZE SELECT count(*) FROM bookings.flights_new WHERE departure_airport between '2017-08-01' AND '2017-08-15' AND departure_airport='DME';"
                                        QUERY PLAN
------------------------------------------------------------------------------------------
 Aggregate  (cost=0.00..0.01 rows=1 width=8) (actual time=0.019..0.020 rows=1 loops=1)
   ->  Result  (cost=0.00..0.00 rows=0 width=0) (actual time=0.003..0.003 rows=0 loops=1)
         One-Time Filter: false
 Planning Time: 2.632 ms
 Execution Time: 0.123 ms

~$ sudo -u postgres psql -d demo -c "EXPLAIN ANALYZE SELECT count(*) FROM bookings.flights_new WHERE departure_airport between '2017-08-01' AND '2017-08-15' AND departure_airport='DME';"
                                        QUERY PLAN
------------------------------------------------------------------------------------------
 Aggregate  (cost=0.00..0.01 rows=1 width=8) (actual time=0.010..0.011 rows=1 loops=1)
   ->  Result  (cost=0.00..0.00 rows=0 width=0) (actual time=0.002..0.003 rows=0 loops=1)
         One-Time Filter: false
 Planning Time: 1.752 ms
 Execution Time: 0.089 ms

~$ sudo -u postgres psql -d demo -c "EXPLAIN ANALYZE SELECT count(*) FROM bookings.flights_new WHERE departure_airport between '2017-08-01' AND '2017-08-15' AND departure_airport='DME';"
                                        QUERY PLAN
------------------------------------------------------------------------------------------
 Aggregate  (cost=0.00..0.01 rows=1 width=8) (actual time=0.011..0.013 rows=1 loops=1)
   ->  Result  (cost=0.00..0.00 rows=0 width=0) (actual time=0.002..0.003 rows=0 loops=1)
         One-Time Filter: false
 Planning Time: 2.104 ms
 Execution Time: 0.146 ms
```
**Видим, что теперь скорость выросла более чем в 100 раз**
```commandline
Execution Time: 0.123 ms
Execution Time: 0.089 ms
Execution Time: 0.146 ms
```

**Видим очевидное преимущество, но с учетом информации из леции понимаем, что процесс настройки 
использования подобной схемы (Таблицы с использованием метода секционирования) усложняется и требует детальной 
проработки вопроса создания требуемых индексов и т.д.**
___
