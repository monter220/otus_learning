# Подготовка к сравнениям

*__Имеем ВМ с ОС Ubuntu 20.04 со следующими параметрами__*
```commandline
Number of CPUs - 8
RAM - 4 Gb
ROM - 50 Gb (в ходе работы расширено до 200 GB)
```
**Создаем новый кластер**
```commandline
~$ sudo -u postgres pg_createcluster 15 main
~$ sudo pg_ctlcluster 15 main start
```

**Создаем новую таблицу с первоначальными данными, 
которую будем использовать как контрольную**
```commandline
~$ sudo -u postgres psql -c "CREATE TABLE control
>  (
> trade_id integer PRIMARY KEY,
> order_id numeric(15),
> p numeric(18, 6),
> date timestamp,
> profit numeric(19, 2),
> instr_nm varchar(100),
> curr_c varchar(10),
> q numeric(18, 6),
> summ numeric(18, 2),
> summ_in_base_currency numeric(18, 2),
> home_currency text,
> operation varchar(10),
> auth_login text,
> instr_type int4,
> details jsonb
>  );"
```
**Заливаем данные в контрольную таблицу**
```commandline
~$ sudo -u postgres psql -c "COPY control
> (
> trade_id, order_id, p, date, profit, instr_nm, curr_c,
> q, summ, summ_in_base_currency, home_currency, operation, auth_login, instr_type, details
> )
> FROM '/tmp/src/source.csv'
> DELIMITER ','
> CSV HEADER;"
COPY 2200000
```

**Проверяем, что данные успешно попали и оцениваем скорость подсчета кол-ва строк**
```commandline
~$ sudo -u postgres psql -c"\timing" -c  "SELECT count(*) FROM control;"
Timing is on.
  count
---------
 2200000
(1 row)

Time: 795.645 ms

~$ sudo -u postgres psql -c "EXPLAIN ANALYZE SELECT count(*) FROM control;"
                                                                              QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=83446.18..83446.19 rows=1 width=8) (actual time=814.278..831.422 rows=1 loops=1)
   ->  Gather  (cost=83445.96..83446.17 rows=2 width=8) (actual time=813.900..831.399 rows=3 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Partial Aggregate  (cost=82445.96..82445.97 rows=1 width=8) (actual time=805.441..805.443 rows=1 loops=3)
               ->  Parallel Index Only Scan using control_pkey on control  (cost=0.43..80148.23 rows=919092 width=0) (actual time=0.302..711.057 rows=733333 loops=3)
                     Heap Fetches: 403326
 Planning Time: 21.420 ms
 Execution Time: 831.866 ms
(9 rows)
```
**Контрольная таблица готова**
___
**Создадим новую таблицу для настройки индексов и скопируем данные из контрольной таблицы**
```commandline
~$ sudo -u postgres psql -c"\timing" -c  "CREATE TABLE index (LIKE control);"
Timing is on.
CREATE TABLE
Time: 49.170 ms

~$ sudo -u postgres psql -c"\timing" -c  "INSERT INTO index SELECT * from control;"
Timing is on.
INSERT 0 2200000
Time: 32785.684 ms (00:32.786)
```

**Пока что разницы между таблицами нет, но теперь попробуем настроить 
индексы в соответствии с предполагаемыми вариантами выгрузок**

```commandline
~$ sudo -u postgres psql -c "\timing" -c "CREATE INDEX idx_order ON index (order_id);"
[sudo] password for monter:
Timing is on.
CREATE INDEX
Time: 6164.046 ms (00:06.164)

~$ sudo -u postgres psql -c "\timing" -c "CREATE INDEX idx_date ON index (date);"
Timing is on.
CREATE INDEX
Time: 2743.140 ms (00:02.743)

~$ sudo -u postgres psql -c "\timing" -c "CREATE INDEX idx_instr ON index (instr_nm);"
Timing is on.
CREATE INDEX
Time: 3880.149 ms (00:03.880)

~$ sudo -u postgres psql -c "\timing" -c "CREATE INDEX idx_auth ON index (auth_login);"
Timing is on.
CREATE INDEX
Time: 4356.358 ms (00:04.356)

~$ sudo -u postgres psql -c "\timing" -c "CREATE INDEX idx_auth_instr ON index (auth_login, instr_nm);"
Timing is on.
CREATE INDEX
Time: 6840.469 ms (00:06.840)

~$ sudo -u postgres psql -c "\timing" -c "CREATE INDEX idx_auth_trade ON index (auth_login, trade_id);"
Timing is on.
CREATE INDEX
Time: 7944.111 ms (00:07.944)

~$ sudo -u postgres psql -c "\timing" -c "CREATE INDEX idx_auth_data ON index (auth_login, date);"
Timing is on.
CREATE INDEX
Time: 6597.281 ms (00:06.597)

~$ sudo -u postgres psql -c "\timing" -c "CREATE INDEX idx_details_route ON index USING btree (((details ->> 'common_id'::text)));"
Timing is on.
CREATE INDEX
Time: 3688.931 ms (00:03.689)
```

**На всякий случай проверим, что простой счетчик работает не хуже чем в контрольной таблице**
```commandline
~$ sudo -u postgres psql -c"\timing" -c  "SELECT count(*) FROM index;"
Timing is on.
  count
---------
 2200000
(1 row)

Time: 261.837 ms
```
___
**Переходим к созданию варианта таблицы с использованием `Table Partitioning`.**
**Самым оптимальным вариантом деления будет деление по полю `data`**
```commandline
~$ sudo -u postgres psql -c "\timing" -c "CREATE TABLE part (LIKE control) PARTITION BY RANGE(date);"
Timing is on.
CREATE TABLE
Time: 7.643 ms
~$ sudo -u postgres psql -c "\timing" -c "CREATE TABLE part_202207 PARTITION OF part FOR VALUES FROM ('2022-07-01'::timestamptz) TO ('2022-10-01'::timestamptz);"
Timing is on.
CREATE TABLE
Time: 39.186 ms
~$ sudo -u postgres psql -c "\timing" -c "CREATE TABLE part_202210 PARTITION OF part FOR VALUES FROM ('2022-10-01'::timestamptz) TO ('2023-01-01'::timestamptz);"
Timing is on.
CREATE TABLE
Time: 15.969 ms
~$ sudo -u postgres psql -c "\timing" -c "CREATE TABLE part_202301 PARTITION OF part FOR VALUES FROM ('2023-01-01'::timestamptz) TO ('2023-04-01'::timestamptz);"
Timing is on.
CREATE TABLE
Time: 11.207 ms
~$ sudo -u postgres psql -c "\timing" -c "CREATE TABLE part_202304 PARTITION OF part FOR VALUES FROM ('2023-04-01'::timestamptz) TO ('2023-07-01'::timestamptz);"
Timing is on.
CREATE TABLE
Time: 11.341 ms
~$ sudo -u postgres psql -c "\timing" -c "CREATE TABLE part_202307 PARTITION OF part FOR VALUES FROM ('2023-07-01'::timestamptz) TO ('2023-10-01'::timestamptz);"
Timing is on.
CREATE TABLE
Time: 10.857 ms
~$ sudo -u postgres psql -c "\timing" -c "CREATE TABLE part_202310 PARTITION OF part FOR VALUES FROM ('2023-10-01'::timestamptz) TO ('2024-01-01'::timestamptz);"
Timing is on.
CREATE TABLE
Time: 14.154 ms
```
**Наполним таблицу данными**
```commandline
~$ sudo -u postgres psql -c"\timing" -c  "INSERT INTO part SELECT * from control;"
Timing is on.
INSERT 0 2200000
Time: 32785.684 ms (00:32.786)
```
**Прогоним счетчик**
```commandline
~$ sudo -u postgres psql -c"\timing" -c  "SELECT count(*) FROM part;"
Timing is on.
  count
---------
 2200000
(1 row)

Time: 692.446 ms
```
______
**Попробуем совместить предыдущие 2 эксперимента в таблице part_index**
```commandline
~$ sudo -u postgres psql -c "\timing" -c "CREATE TABLE part_index (LIKE control) PARTITION BY RANGE(date);"
Timing is on.
CREATE TABLE
Time: 7.234 ms
~$ sudo -u postgres psql -c "\timing" -c "CREATE TABLE part_index_202207 PARTITION OF part_index FOR VALUES FROM ('2022-07-01'::timestamptz) TO ('2022-10-01'::timestamptz);"
Timing is on.
CREATE TABLE
Time: 28.362 ms
~$ sudo -u postgres psql -c "\timing" -c "CREATE TABLE part_index_202210 PARTITION OF part_index FOR VALUES FROM ('2022-10-01'::timestamptz) TO ('2023-01-01'::timestamptz);"
Timing is on.
CREATE TABLE
Time: 11.437 ms
~$ sudo -u postgres psql -c "\timing" -c "CREATE TABLE part_index_202301 PARTITION OF part_index FOR VALUES FROM ('2023-01-01'::timestamptz) TO ('2023-04-01'::timestamptz);"
Timing is on.
CREATE TABLE
Time: 10.777 ms
~$ sudo -u postgres psql -c "\timing" -c "CREATE TABLE part_index_202304 PARTITION OF part_index FOR VALUES FROM ('2023-04-01'::timestamptz) TO ('2023-07-01'::timestamptz);"
Timing is on.
CREATE TABLE
Time: 10.330 ms
~$ sudo -u postgres psql -c "\timing" -c "CREATE TABLE part_index_202307 PARTITION OF part_index FOR VALUES FROM ('2023-07-01'::timestamptz) TO ('2023-10-01'::timestamptz);"
Timing is on.
CREATE TABLE
Time: 39.940 ms
~$ sudo -u postgres psql -c "\timing" -c "CREATE TABLE part_index_202310 PARTITION OF part_index FOR VALUES FROM ('2023-10-01'::timestamptz) TO ('2024-01-01'::timestamptz);"
Timing is on.
CREATE TABLE
Time: 10.701 ms
~$ sudo -u postgres psql -c"\timing" -c  "INSERT INTO part_index SELECT * from control;"
Timing is on.
INSERT 0 2200000
Time: 33901.901 ms (00:33.902)
```
**Создадим такие же индексы, что и у таблицы `index`**
```commandline
~$ sudo -u postgres psql -c "\timing" -c "CREATE INDEX idx_order2 ON part_index (order_id);"
Timing is on.
CREATE INDEX
Time: 6316.841 ms (00:06.317)
~$ sudo -u postgres psql -c "\timing" -c "CREATE INDEX idx_date2 ON part_index (date);"
Timing is on.
CREATE INDEX
Time: 3588.840 ms (00:03.589)
~$ sudo -u postgres psql -c "\timing" -c "CREATE INDEX idx_instr2 ON part_index (instr_nm);"
Timing is on.
CREATE INDEX
Time: 3790.950 ms (00:03.791)
~$ sudo -u postgres psql -c "\timing" -c "CREATE INDEX idx_auth2 ON part_index (auth_login);"
Timing is on.
CREATE INDEX
Time: 4342.336 ms (00:04.342)
~$ sudo -u postgres psql -c "\timing" -c "CREATE INDEX idx_auth_instr2 ON part_index (auth_login, instr_nm);"
Timing is on.
CREATE INDEX
Time: 5986.771 ms (00:05.987)
~$ sudo -u postgres psql -c "\timing" -c "CREATE INDEX idx_auth_trade2 ON part_index (auth_login, trade_id);"
Timing is on.
CREATE INDEX
Time: 9032.235 ms (00:09.032)
~$ sudo -u postgres psql -c "\timing" -c "CREATE INDEX idx_auth_data2 ON part_index (auth_login, date);"
Timing is on.
CREATE INDEX
Time: 5243.903 ms (00:05.244)
~$ sudo -u postgres psql -c "\timing" -c "CREATE INDEX idx_details_route2 ON part_index USING btree (((details ->> 'common_id'::text)));"
Timing is on.
CREATE INDEX
Time: 3757.946 ms (00:03.758)
```
**Проверим что счетчик также работает**
```commandline
~$ sudo -u postgres psql -c"\timing" -c  "SELECT count(*) FROM part_index;"
Timing is on.
  count
---------
 2200000
(1 row)

Time: 215.834 ms
```

______
**Установим MySQL**
```commandline
~$ sudo apt install -y mysql-server
~$ sudo mysql_secure_installation
~$ sudo mysqladmin -p -u root version
Enter password:
mysqladmin  Ver 8.0.35-0ubuntu0.20.04.1 for Linux on x86_64 ((Ubuntu))
Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Server version          8.0.35-0ubuntu0.20.04.1
Protocol version        10
Connection              Localhost via UNIX socket
UNIX socket             /var/run/mysqld/mysqld.sock
Uptime:                 2 min 17 sec

Threads: 2  Questions: 8  Slow queries: 0  Opens: 131  Flush tables: 3  Open tables: 50  Queries per second avg: 0.058
```

**Создадим таблицу и наполним ее данными из csv**
```commandline
~$ sudo mysql -u root -e "create database demo"
~$ sudo mysql -u root -D demo -e "create table mysql (
p numeric(18, 6),
> trade_id integer PRIMARY KEY,
> order_id numeric(15),
> p numeric(18, 6),
> date timestamp,
auth_login text,
> profit numeric(19, 2),
)"> instr_nm varchar(100),
> curr_c varchar(10),
> q numeric(18, 6),
> summ numeric(18, 2),
> summ_in_base_currency numeric(18, 2),
> home_currency text,
> operation varchar(10),
> auth_login text,
> instr_type int4,
> details json
> )"
~$ sudo mysql -vv -u root -D demo -e "SHOW VARIABLES LIKE 'secure_file_priv'"
--------------
SHOW VARIABLES LIKE 'secure_file_priv'
--------------

+------------------+-----------------------+
| Variable_name    | Value                 |
+------------------+-----------------------+
| secure_file_priv | /var/lib/mysql-files/ |
+------------------+-----------------------+
1 row in set (0.03 sec)

Bye
~$ sudo mv /tmp/src/source.csv /var/lib/mysql-files/
~$ sudo mysql -vv -u root -D demo -e "LOAD DATA INFILE '/var/lib/mysql-files/source.csv' IGNORE
> INTO TABLE mysql
> COLUMNS TERMINATED BY ','
> LINES TERMINATED BY '\n'
> (
> @trade_id,
> @order_id,
> @p,
> @date,
> @profit,
> @instr_nm,
> @curr_c,
> @q,
> @summ,
> @summ_in_base_currency,
> @home_currency,
> @operation,
> @auth_login,
> @instr_type)
> SET
> trade_id = NULLIF(@trade_id, ''),
> order_id = NULLIF(@order_id, ''),
> p = NULLIF(@p, ''),
> date = NULLIF(@date, ''),
> profit = NULLIF(@profit, ''),
> instr_nm = NULLIF(@instr_nm, ''),
> curr_c = NULLIF(@curr_c, ''),
> q = NULLIF(@q, ''),
> summ = NULLIF(@summ, ''),
> summ_in_base_currency = NULLIF(@summ_in_base_currency, ''),
> home_currency = NULLIF(@home_currency, ''),
> operation = NULLIF(@operation, ''),
> auth_login = NULLIF(@auth_login, ''),
> instr_type = NULLIF(@instr_type, '')
> "
--------------
LOAD DATA INFILE '/var/lib/mysql-files/source.csv' IGNORE
INTO TABLE mysql
COLUMNS TERMINATED BY ','
LINES TERMINATED BY '\n'
(
@trade_id,
@order_id,
@p,
@date,
@profit,
@instr_nm,
@curr_c,
@q,
@summ,
@summ_in_base_currency,
@home_currency,
@operation,
@auth_login,
@instr_type
)
SET
trade_id = NULLIF(@trade_id, ''),
order_id = NULLIF(@order_id, ''),
p = NULLIF(@p, ''),
date = NULLIF(@date, ''),
profit = NULLIF(@profit, ''),
instr_nm = NULLIF(@instr_nm, ''),
curr_c = NULLIF(@curr_c, ''),
q = NULLIF(@q, ''),
summ = NULLIF(@summ, ''),
summ_in_base_currency = NULLIF(@summ_in_base_currency, ''),
home_currency = NULLIF(@home_currency, ''),
operation = NULLIF(@operation, ''),
auth_login = NULLIF(@auth_login, ''),
instr_type = NULLIF(@instr_type, '')
--------------


Query OK, 2200001 rows affected
Records: 2200001  

Bye
```
**Пришлось отказаться от заполнения `json` поля, так как победить ошибку заполнения не вышло**
**А так же обнаружено, что при заполнении данных было залито +1 дополнительная строка**
**Проверим что данные попали в таблицу**
```commandline
~$ sudo mysql -vv -u root -D demo -e "SELECT count(*) FROM mysql"
--------------
SELECT count(*) FROM mysql
--------------

+----------+
| count(*) |
+----------+
|  2200001 |
+----------+
1 row in set (0.73 sec)

Bye
```

# Сравнение вариантов настройки

**Список запросов по типам**

- Счетчик всех полей таблицы с прогретым кешем:
```commandline
~$ sudo mysql -vv -u root -D demo -e "SELECT count(*) FROM mysql"
~$ sudo -u postgres psql -c "\timing" -c "SELECT count(*) FROM control;"
~$ sudo -u postgres psql -c "\timing" -c "SELECT count(*) FROM index;"
~$ sudo -u postgres psql -c "\timing" -c "SELECT count(*) FROM part;"
~$ sudo -u postgres psql -c "\timing" -c "SELECT count(*) FROM part_index;"
```

- Счетчик за диапазон времени:
```
~$ sudo mysql -vv -u root -D demo -e "SELECT count(*) FROM mysql WHERE date BETWEEN '2023-01-01' AND '2023-01-03'"
~$ sudo -u postgres psql -c "\timing" -c "SELECT count(*) FROM control WHERE date BETWEEN '2023-01-01' AND '2023-01-03';"
~$ sudo -u postgres psql -c "\timing" -c "SELECT count(*) FROM index WHERE date BETWEEN '2023-01-01' AND '2023-01-03';"
~$ sudo -u postgres psql -c "\timing" -c "SELECT count(*) FROM part WHERE date BETWEEN '2023-01-01' AND '2023-01-03';"
~$ sudo -u postgres psql -c "\timing" -c "SELECT count(*) FROM part_index WHERE date BETWEEN '2023-01-01' AND '2023-01-03';"
```
- Счетчик по `auth_login` за диапазон времени:
```commandline
~$ sudo mysql -vv -u root -D demo -e "SELECT count(trade_id), auth_login FROM mysql WHERE date BETWEEN '2023-01-01' AND '2023-01-03' GROUP BY auth_login"
~$ sudo -u postgres psql -c "\timing" -c "SELECT count(trade_id), auth_login FROM control WHERE date BETWEEN '2023-01-01' AND '2023-01-03' GROUP BY auth_login;"
~$ sudo -u postgres psql -c "\timing" -c "SELECT count(trade_id), auth_login FROM index WHERE date BETWEEN '2023-01-01' AND '2023-01-03' GROUP BY auth_login;"
~$ sudo -u postgres psql -c "\timing" -c "SELECT count(trade_id), auth_login FROM part WHERE date BETWEEN '2023-01-01' AND '2023-01-03' GROUP BY auth_login;"
~$ sudo -u postgres psql -c "\timing" -c "SELECT count(trade_id), auth_login FROM part_index WHERE date BETWEEN '2023-01-01' AND '2023-01-03' GROUP BY auth_login;"
```
- Подсчет суммы `p` за диапазон времени с сортировкой по 'auth_login':
```commandline
~$ sudo mysql -vv -u root -D demo -e "SELECT sum(p), auth_login FROM mysql WHERE date BETWEEN '2022-10-01' AND '2023-01-03' GROUP BY auth_login ORDER BY auth_login"
~$ sudo -u postgres psql -c "\timing" -c "SELECT sum(p), auth_login FROM control WHERE date BETWEEN '2023-01-01' AND '2023-01-03' GROUP BY auth_login ORDER BY auth_login;"
~$ sudo -u postgres psql -c "\timing" -c "SELECT sum(p), auth_login FROM index WHERE date BETWEEN '2023-01-01' AND '2023-01-03' GROUP BY auth_login ORDER BY auth_login;"
~$ sudo -u postgres psql -c "\timing" -c "SELECT sum(p), auth_login FROM part WHERE date BETWEEN '2023-01-01' AND '2023-01-03' GROUP BY auth_login ORDER BY auth_login;"
~$ sudo -u postgres psql -c "\timing" -c "SELECT sum(p), auth_login FROM part_index WHERE date BETWEEN '2023-01-01' AND '2023-01-03' GROUP BY auth_login ORDER BY auth_login;"
```
- Отбор 3 последних по времени строк из диапазона, которые соответствуют фильтру по `json` полю:
```commandline
~$ sudo -u postgres psql -c "\timing" -c "SELECT * FROM control WHERE date BETWEEN '2023-01-01' AND '2023-07-10' AND details->>'route' = 'NYSE' ORDER BY date DESC LIMIT 3;"
~$ sudo -u postgres psql -c "\timing" -c "SELECT * FROM index WHERE date BETWEEN '2023-01-01' AND '2023-07-10' AND details->>'route' = 'NYSE' ORDER BY date DESC LIMIT 3;"
~$ sudo -u postgres psql -c "\timing" -c "SELECT * FROM part WHERE date BETWEEN '2023-01-01' AND '2023-07-10' AND details->>'route' = 'NYSE' ORDER BY date DESC LIMIT 3;"
~$ sudo -u postgres psql -c "\timing" -c "SELECT * FROM part_index WHERE date BETWEEN '2023-01-01' AND '2023-07-10' AND details->>'route' = 'NYSE' ORDER BY date DESC LIMIT 3;"
```
- Добавить 1 строку в таблицу:
```commandline
~$ sudo -u postgres psql -c "\timing" -c "COPY control 
(trade_id, order_id, p, date, profit, instr_nm, curr_c, q, summ, 
summ_in_base_currency, home_currency, operation, auth_login, instr_type, 
details) FROM '/tmp/src/source_1.csv' DELIMITER ',' CSV HEADER;"

~$ sudo mv /tmp/src/1.csv /var/lib/mysql-files/
~$ sudo mysql -vv -u root -D demo -e "LOAD DATA INFILE '/var/lib/mysql-files/1.csv' IGNORE
> INTO TABLE mysql
> COLUMNS TERMINATED BY ','
> LINES TERMINATED BY '\n'
> (
> @trade_id,
> @order_id,
> @p,
> @date,
> @profit,
> @instr_nm,
> @curr_c,
> @q,
> @summ,
> @summ_in_base_currency,
> @home_currency,
> @operation,
> @auth_login,
> @instr_type)
> SET
> trade_id = NULLIF(@trade_id, ''),
> order_id = NULLIF(@order_id, ''),
> p = NULLIF(@p, ''),
> date = NULLIF(@date, ''),
> profit = NULLIF(@profit, ''),
> instr_nm = NULLIF(@instr_nm, ''),
> curr_c = NULLIF(@curr_c, ''),
> q = NULLIF(@q, ''),
> summ = NULLIF(@summ, ''),
> summ_in_base_currency = NULLIF(@summ_in_base_currency, ''),
> home_currency = NULLIF(@home_currency, ''),
> operation = NULLIF(@operation, ''),
> auth_login = NULLIF(@auth_login, ''),
> instr_type = NULLIF(@instr_type, '')"
```
- Добавить 10 X 1М строк в таблицу:
```commandline
control
Time: 31782.646 ms
Time: 32448.302 ms
Time: 32627.663 ms
Time: 37667.014 ms
Time: 32481.144 ms
Time: 32216.617 ms
Time: 24223.711 ms
Time: 27545.951 ms
Time: 35967.137 ms
Time: 33624.751 ms

index
Time: 270842.332 ms
Time: 175856.255 ms
Time: 152046.423 ms
Time: 212180.499 ms
Time: 245293.222 ms
Time: 262121.047 ms
Time: 276140.767 ms
Time: 285973.545 ms
Time: 298005.596 ms
Time: 421374.469 ms

part
Time: 35032.689 ms
Time: 26400.056 ms
Time: 30830.141 ms
Time: 27347.640 ms
Time: 32409.562 ms
Time: 33574.147 ms
Time: 26721.791 ms
Time: 28913.319 ms
Time: 33911.640 ms
Time: 35732.633 ms

part_index
Time: 105696.205 ms
Time: 115576.918 ms
Time: 113724.877 ms
Time: 182248.183 ms
Time: 203560.716 ms
Time: 237579.736 ms
Time: 147584.335 ms
Time: 129076.260 ms
Time: 187445.087 ms
Time: 198171.445 ms

mysql
48.16 sec
48.19 sec
50.03 sec
46.55 sec
47.40 sec
48.09 sec
47.10 sec
50.08 sec
53.87 sec
66.74 sec
```

- Поиск максимальных профитов по заданым условиям
  - 1 Вариант запроса

```commandline
~$ sudo -u postgres psql -c "\timing" -c "SELECT DISTINCT ON (route) route, cnt, profit, summ, order_id , auth_login
FROM (
SELECT count(trade_id) AS cnt, SUM(profit) AS profit, SUM(summ) AS summ,
order_id , auth_login, details->>'route' AS route
FROM control
WHERE TRUE
AND date BETWEEN '2023-01-01' AND '2023-04-01'
AND details->>'route' IS NOT NULL
GROUP BY order_id , auth_login, details->>'route'
HAVING count(trade_id) > 5) AS tmp
ORDER BY route, profit DESC;"
```
- Поиск максимальных профитов по заданым условиям
  - 2 вариант запроса 
```
~$ sudo mysql -vv -u root -D demo -e "SELECT DISTINCT auth_login, cnt, profit, summ, order_id
FROM (
SELECT count(trade_id) AS cnt, SUM(profit) AS profit, SUM(summ) AS summ,
order_id , auth_login
FROM mysql
WHERE TRUE
AND date BETWEEN '2023-01-01' AND '2023-04-01'
GROUP BY order_id , auth_login
HAVING count(trade_id) > 100) AS tmp
ORDER BY auth_login, profit DESC"

~$ sudo -u postgres psql -c "\timing" -c "SELECT DISTINCT ON (auth_login) auth_login, cnt, profit, summ, order_id
FROM (
SELECT count(trade_id) AS cnt, SUM(profit) AS profit, SUM(summ) AS summ,
order_id , auth_login
FROM control
WHERE TRUE
AND date BETWEEN '2023-01-01' AND '2023-04-01'
GROUP BY order_id , auth_login
HAVING count(trade_id) > 100) AS tmp
ORDER BY auth_login, profit DESC;"
```

- Определение основного веса таблицы
```commandline
~$ sudo -u postgres psql -c "\timing" -c "
SELECT nspname  '.'  relname AS "relation", 
pg_size_pretty(pg_relation_size(C.oid)) AS "size" 
FROM pg_class C 
LEFT JOIN pg_namespace N ON (N.oid = C.relnamespace) 
WHERE nspname = 'public' 
ORDER BY pg_relation_size(C.oid) DESC;"

~$ sudo mysql -vv -u root -D demo -e "
SELECT table_name , round(((data_length + index_length) / 1024 / 1024 ), 2) 
FROM information_schema.TABLES WHERE table_schema = 'demo'"
```
- Определение веса вспомогательных данных таблицы(партиции, индексы...)

| Действие                                                                                                | control             | index                | part                | part_index           | mysql      |
|---------------------------------------------------------------------------------------------------------|---------------------|----------------------|---------------------|----------------------|------------|
| Создание таблицы                                                                                        | Time: 49.170 ms     | Time: 42263.655 ms   | Time: 110.357 ms    | Time: 42178.503 ms   | NULL       | 
| Счетчик всех полей <br/>таблицы с прогретым кешем                                                       | Time: 742.144 ms    | Time: 182.679 ms     | Time: 688.087 ms    | Time: 223.575 ms     | 0.87 sec   | 
| Счетчик за диапазон времени                                                                             | Time: 698.439 ms    | Time: 5.167 ms       | Time: 346.827 ms    | Time: 6.442 ms       | 2.91 sec   | 
| Счетчик по `auth_login`<br/> за диапазон времени                                                        | Time: 722.046 ms    | Time: 6.714 ms       | Time: 335.059 ms    | Time: 8.658 ms       | 6.01 sec   | 
| Подсчет суммы `p` <br/>за диапазон времени с <br/>сортировкой по 'auth_login'                           | Time: 764.044 ms    | Time: 7.739 ms       | Time: 340.620 ms    | Time: 8.935 ms       | 8.58 sec   | 
| Отбор 3 последних по <br/>времени строк из диапазона, которые <br/>соответствуют фильтру по `json` полю | Time: 881.520 ms    | Time: 6.623 ms       | Time: 725.881 ms    | Time: 12.012 ms      | NULL       |
| Добавить 1 строку в таблицу                                                                             | Time: 139.130 ms    | Time: 218.795 ms     | Time: 52.206 ms     | Time: 151.351 ms     | 0.32 sec   |
| Добавить 1М строку в таблицу                                                                            | Time: 35851.870 ms  | Time: 129344.546 ms  | Time: 28165.354 ms  | Time: 71446.436 ms   | 51.08 sec  |
| Добавить 10 X 1М строку в таблицу                                                                       | Time: 320584.936 ms | Time: 2599834.155 ms | Time: 310873.618 ms | Time: 1620663,762 ms | 506.21 sec |
| Счетчик за диапазон времени<br/> после добавления данных                                                | Time: 64901.457 ms  | Time: 168.572 ms     | Time: 3916.888 ms   | Time: 112.853 ms     | 76.37 sec  |
| Добавление еще 1М записей после <br/>доведения кол-ва строк в таблице > 25M                             | Time: 59178.928 ms  | Time: 1463059.311 ms | Time: 45389.109 ms  | Time: 300124.521 ms  | 56.83 sec  |
| Счетчик всех полей<br/> после добавления данных                                                         | Time: 84690.141 ms  | Time: 88025.080 ms   | Time: 55602.968 ms  | Time: 67252.267 ms   | 80.33 sec  |
| Счетчик за диапазон времени<br/> после добавления данных                                                | Time: 84082.855 ms  | Time: 5.038 ms       | Time: 643.404 ms    | Time: 5.912 ms       | 150.26 sec |
| Поиск максимальных профитов по задданым условиям<br/>1 вариант                                          | Time: 88289.460 ms  | Time: 94023.912 ms   | Time: 4515.378 ms   | Time: 3722.837 ms    | NULL       |
| Поиск максимальных профитов по задданым условиям<br/>2 вариант                                          | Time: 90084.894 ms  | Time: 98440.353 ms   | Time: 6932.736 ms   | Time: 5815.889 ms    | 204.58 sec |
| Определение основного веса таблицы                                                                      | 11 GB               | 11 GB                | 0                   | 0                    | 6216.94 MB |
| Определение веса вспомогательных данных таблицы(партиции, индексы...)                                   | 0                   | 5398 MB              | 10625 MB            | 15925 MB             | 0          |

___
___

### Первичные выводы на основе первых 6 проверок

**Из собранных данных получаем, что при своей сложности в настройке и продумывании 
вариант с грамотно настроенными индексами оказывается самым выигрышным**

___

**Итоговый рейтинг вариантов настройки**

- По скорости разворачивания таблицы:
  - Без каких-либо настроек (control)
  - Секционированная таблица (part)
  - Таблица с настроенными индексами и секционирование (part_index) *Что немного странно, полагаю, что тут оказала влияние сама ВМ*
  - Таблица с настроенными индексами (index)
  - MySQL таблица (mysql) *так как не удалось настроить и заполнить всеми данными, а так же при импорте из csv загрузилась лишняя строка*
- По скорости работы:
  - Таблица с настроенными индексами (index)
  - Таблица с настроенными индексами и секционирование (part_index)
  - Секционированная таблица (part)
  - Без каких-либо настроек (control)
  - MySQL таблица (mysql)

___
___

# Выводы на основе всех экспериментов

При увеличении кол-ва записей в таблице в процессе работы было выявлено:
- Таблица на MySQL из коробки не дает каких-либо преимуществ перед PG даже при отсутствии в таблице PG доп надстроек
- Использование индексации при существовании необходимости добавлять данные в случайные области таблицы ведет к падению скорости работы с таблицей даже по сравнению с контрольной таблицей
- Использование секционированной таблицы даем максимальные преимущества при необходимости добавлять данные в случайные области таблицы
- Если в таблицу в процессе работы планируется добавлять данные не случайным образом, а строго по возрастанию ID, то использование секционированной таблицы с организованными индексами имеет лучшие показатели скорости предоставления данных (в ходе экспериментов случайным образом вышло, что в временной блок, из которого извлекаись данные, оказался менее загруженным на этапе добавлении данных)
- Если таблица подразумевает разовые внесения данных и огромное кол-во извлечений данных, то индексы ускоряют работу многократно
- По мере добавления данных выяснилось, что сами индексы начинают занимать очень много пространства (5398 MB для таблицы `index` к концы всех `insert`)

___