# Подготовка к сравнениям

*__Имеем ВМ с ОС Ubuntu 20.04 со следующими параметрами__*
```commandline
Number of CPUs - 8
RAM - 4 Gb
ROM - 50 Gb
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
> FROM '/tmp/src/2307_2310.csv'
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
~$ sudo mysql -vv -u root -D demo -e "LOAD DATA INFILE '/var/lib/mysql-files/2207_2210.csv' IGNORE
@profit,
@instr_nm,
> INTO TABLE mysql
)
> COLUMNS TERMINATED BY ','
> LINES TERMINATED BY '\n'
> (
curr_c = NULLIF(@curr_c, ''),
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
> @instr_type
instr_type = NULLIF(@instr_type, '')
"> )
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

- Счетчик всех полей таблицы с прогретым кешем
```commandline
~$ sudo mysql -vv -u root -D demo -e "SELECT count(*) FROM mysql"
~$ sudo -u postgres psql -c "\timing" -c "SELECT count(*) FROM control;"
~$ sudo -u postgres psql -c "\timing" -c "SELECT count(*) FROM index;"
~$ sudo -u postgres psql -c "\timing" -c "SELECT count(*) FROM part;"
~$ sudo -u postgres psql -c "\timing" -c "SELECT count(*) FROM part_index;"
```

- Счетчик за диапазон времени
```
~$ sudo mysql -vv -u root -D demo -e "SELECT count(*) FROM mysql WHERE date BETWEEN '2023-01-01' AND '2023-01-03'"
~$ sudo -u postgres psql -c "\timing" -c "SELECT count(*) FROM control WHERE date BETWEEN '2023-01-01' AND '2023-01-03';"
~$ sudo -u postgres psql -c "\timing" -c "SELECT count(*) FROM index WHERE date BETWEEN '2023-01-01' AND '2023-01-03';"
~$ sudo -u postgres psql -c "\timing" -c "SELECT count(*) FROM part WHERE date BETWEEN '2023-01-01' AND '2023-01-03';"
~$ sudo -u postgres psql -c "\timing" -c "SELECT count(*) FROM part_index WHERE date BETWEEN '2023-01-01' AND '2023-01-03';"
```
- Счетчик по `auth_login` за диапазон времени
```commandline
~$ sudo mysql -vv -u root -D demo -e "SELECT count(trade_id), auth_login FROM mysql WHERE date BETWEEN '2023-01-01' AND '2023-01-03' GROUP BY auth_login"
~$ sudo -u postgres psql -c "\timing" -c "SELECT count(trade_id), auth_login FROM control WHERE date BETWEEN '2023-01-01' AND '2023-01-03' GROUP BY auth_login;"
~$ sudo -u postgres psql -c "\timing" -c "SELECT count(trade_id), auth_login FROM index WHERE date BETWEEN '2023-01-01' AND '2023-01-03' GROUP BY auth_login;"
~$ sudo -u postgres psql -c "\timing" -c "SELECT count(trade_id), auth_login FROM part WHERE date BETWEEN '2023-01-01' AND '2023-01-03' GROUP BY auth_login;"
~$ sudo -u postgres psql -c "\timing" -c "SELECT count(trade_id), auth_login FROM part_index WHERE date BETWEEN '2023-01-01' AND '2023-01-03' GROUP BY auth_login;"
```
- Подсчет суммы `p` за диапазон времени с сортировкой по 'auth_login'
```commandline
~$ sudo mysql -vv -u root -D demo -e "SELECT sum(p), auth_login FROM mysql WHERE date BETWEEN '2022-10-01' AND '2023-01-03' GROUP BY auth_login ORDER BY auth_login"
~$ sudo -u postgres psql -c "\timing" -c "SELECT sum(p), auth_login FROM control WHERE date BETWEEN '2023-01-01' AND '2023-01-03' GROUP BY auth_login ORDER BY auth_login;"
~$ sudo -u postgres psql -c "\timing" -c "SELECT sum(p), auth_login FROM index WHERE date BETWEEN '2023-01-01' AND '2023-01-03' GROUP BY auth_login ORDER BY auth_login;"
~$ sudo -u postgres psql -c "\timing" -c "SELECT sum(p), auth_login FROM part WHERE date BETWEEN '2023-01-01' AND '2023-01-03' GROUP BY auth_login ORDER BY auth_login;"
~$ sudo -u postgres psql -c "\timing" -c "SELECT sum(p), auth_login FROM part_index WHERE date BETWEEN '2023-01-01' AND '2023-01-03' GROUP BY auth_login ORDER BY auth_login;"
```
- Отбор 3 последних по времени строк из диапазона, которые соответствуют фильтру по `json` полю
```commandline
~$ sudo -u postgres psql -c "\timing" -c "SELECT * FROM control WHERE date BETWEEN '2023-01-01' AND '2023-07-10' AND details->>'route' = 'NYSE' ORDER BY date DESC LIMIT 3;"
~$ sudo -u postgres psql -c "\timing" -c "SELECT * FROM index WHERE date BETWEEN '2023-01-01' AND '2023-07-10' AND details->>'route' = 'NYSE' ORDER BY date DESC LIMIT 3;"
~$ sudo -u postgres psql -c "\timing" -c "SELECT * FROM part WHERE date BETWEEN '2023-01-01' AND '2023-07-10' AND details->>'route' = 'NYSE' ORDER BY date DESC LIMIT 3;"
~$ sudo -u postgres psql -c "\timing" -c "SELECT * FROM part_index WHERE date BETWEEN '2023-01-01' AND '2023-07-10' AND details->>'route' = 'NYSE' ORDER BY date DESC LIMIT 3;"
```

| Действие                                                                                                | control          | index              | part             | part_index         | mysql    |
|---------------------------------------------------------------------------------------------------------|------------------|--------------------|------------------|--------------------|----------|
| Создание таблицы                                                                                        | Time: 49.170 ms  | Time: 42263.655 ms | Time: 110.357 ms | Time: 42178.503 ms | NULL     | 
| Счетчик всех полей <br/>таблицы с прогретым кешем                                                       | Time: 742.144 ms | Time: 182.679 ms   | Time: 688.087 ms | Time: 223.575 ms   | 0.87 sec | 
| Счетчик за диапазон времени                                                                             | Time: 698.439 ms | Time: 5.167 ms     | Time: 346.827 ms | Time: 6.442 ms     | 2.91 sec | 
| Счетчик по `auth_login`<br/> за диапазон времени                                                        | Time: 722.046 ms | Time: 6.714 ms     | Time: 335.059 ms | Time: 8.658 ms     | 6.01 sec | 
| Подсчет суммы `p` <br/>за диапазон времени с <br/>сортировкой по 'auth_login'                           | Time: 764.044 ms | Time: 7.739 ms     | Time: 340.620 ms | Time: 8.935 ms     | 8.58 sec | 
| Отбор 3 последних по <br/>времени строк из диапазона, которые <br/>соответствуют фильтру по `json` полю | Time: 881.520 ms | Time: 6.623 ms     | Time: 725.881 ms | Time: 12.012 ms    | NULL     | 

___
___
# Вывод

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
