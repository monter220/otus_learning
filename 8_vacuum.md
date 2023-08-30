- [x] Создать инстанс ВМ с 2 ядрами и 4 Гб ОЗУ и SSD 10GB
- [x] Установить на него PostgreSQL 15 с дефолтными настройками
___
*__Использовал команды__*
```commandline
sudo apt update 
sudo apt upgrade -y 
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' 
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - 
sudo apt-get update 
sudo apt-get -y install postgresql-15
```

---
- [x] Создать БД для тестов: выполнить pgbench -i postgres
___
*__Использовал команды__*
```commandline
sudo su postgres
pgbench -i postgres
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.35 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.86 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.60 s, vacuum 0.11 s, primary keys 0.14 s).
```
___
- [x] Запустить pgbench -c8 -P 6 -T 60 -U postgres postgres
___
```commandline
pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (15.4 (Ubuntu 15.4-1.pgdg20.04+1))
starting vacuum...end.
progress: 6.0 s, 822.2 tps, lat 9.613 ms stddev 6.115, 0 failed
progress: 12.0 s, 874.6 tps, lat 9.147 ms stddev 4.709, 0 failed
progress: 18.0 s, 882.3 tps, lat 9.059 ms stddev 4.894, 0 failed
progress: 24.0 s, 870.5 tps, lat 9.188 ms stddev 4.684, 0 failed
progress: 30.0 s, 840.0 tps, lat 9.521 ms stddev 5.982, 0 failed
progress: 36.0 s, 876.5 tps, lat 9.120 ms stddev 4.747, 0 failed
progress: 42.0 s, 877.0 tps, lat 9.120 ms stddev 4.713, 0 failed
```
___
- [x] Применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла
___
*__Внес правки в файл `/etc/postgresql/15/main/postgresql.conf` согласно приложенному файлу__*
___
- [x] Протестировать заново. Что изменилось и почему?
___
```commandline
pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (15.4 (Ubuntu 15.4-1.pgdg20.04+1))
starting vacuum...end.
progress: 6.0 s, 841.0 tps, lat 9.411 ms stddev 4.966, 0 failed
progress: 12.0 s, 847.5 tps, lat 9.154 ms stddev 5.187, 0 failed
progress: 18.0 s, 711.5 tps, lat 11.575 ms stddev 37.105, 0 failed
progress: 24.0 s, 895.5 tps, lat 8.929 ms stddev 4.675, 0 failed
progress: 30.0 s, 871.8 tps, lat 9.170 ms stddev 4.974, 0 failed
progress: 36.0 s, 872.1 tps, lat 9.172 ms stddev 4.647, 0 failed
```
*__В силу не изменения БД изменений не нашел__*
___
- [x] Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк
___
*__Выполнил__*
```commandline
postgres=# CREATE TABLE student(
postgres(#   id serial,
postgres(#   fio char(100)
postgres(# ) WITH (autovacuum_enabled = on);
CREATE TABLE
postgres=# INSERT INTO student(fio) SELECT 'noname' FROM generate_series(1,1000000);
INSERT 0 1000000
```
___
- [x] Посмотреть размер файла с таблицей
___
*__Размер таблицы - `135MB` __*
___
- [x] 5 раз обновить все строчки и добавить к каждой строчке любой символ
___
*__Выполнил__*
```commandline
postgres=# update student set fio ='q';
UPDATE 1000000
postgres=# update student set fio ='qq';
UPDATE 1000000
postgres=# update student set fio ='qqw';
UPDATE 1000000
postgres=# update student set fio ='qqww';
UPDATE 1000000
postgres=# update student set fio ='qqwwe';
UPDATE 1000000
```
___
- [x] Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум
___
```commandline
SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'student';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 student |    1000000 |    3999710 |    399 | 2023-08-30 16:09:42.999578+00
(1 row)
```
___
- [x] Подождать некоторое время, проверяя, пришел ли автовакуум
___
```commandline
SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'student';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 student |    1000000 |          0 |      0 | 2023-08-30 16:10:45.469762+00
(1 row)
```
___
- [x] 5 раз обновить все строчки и добавить к каждой строчке любой символ
___
```commandline
postgres=# update student set fio ='q';                                                    UPDATE 1000000
postgres=# update student set fio ='qq';
UPDATE 1000000
postgres=# update student set fio ='qqq';
UPDATE 1000000
postgres=# update student set fio ='qqw';
UPDATE 1000000
postgres=# update student set fio ='qqwwe';
UPDATE 1000000
```
___
- [x] Посмотреть размер файла с таблицей
___
```commandline
postgres=# SELECT pg_size_pretty(pg_total_relation_size('student'));                        pg_size_pretty
----------------
 808 MB
(1 row)
```
___
- [x] Отключить Автовакуум на конкретной таблице
___
```commandline
postgres=# ALTER TABLE student set (autovacuum_enabled = off);                             
ALTER TABLE
```
___
- [x] 10 раз обновить все строчки и добавить к каждой строчке любой символ
___
*__Выполнил через процедуру, написанную для задания со звездочкой__*
___
- [x] Посмотреть размер файла с таблицей. Объясните полученный результат. Не забудьте включить автовакуум
___
*__Таблица разрослась(с учетом попыток написания процедуры, которые привели к неоднократному запуску апдейтов и получению более 30 МЛН мусорных записей) 
до  `4440 MB`__*
```commandline
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::foat "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'student';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 student |    1000000 |   31954060 |   3195 | 2023-08-30 16:10:45.469762+00
(1 row)
```
*__После включения автовакуума потребовалось 3,5 минуты на вычищение мусора__*
- [x] Задание со *: Написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в искомой таблице.
Не забыть вывести номер шага цикла.
___
*__Написал с указанием кол-ва итераций при вызове__*
```commandline
postgres=# CREATE or replace FUNCTION loop(count INT) RETURNS integer AS $$
BEGIN
FOR i IN 1..count LOOP
update student set fio = 'q'; 
RAISE NOTICE 'LOOP: %',  i;
END LOOP;
END;
$$ LANGUAGE plpgsql;
CREATE FUNCTION
postgres=# select loop(10);
NOTICE:  LOOP: 1
NOTICE:  LOOP: 2
NOTICE:  LOOP: 3
NOTICE:  LOOP: 4
NOTICE:  LOOP: 5
NOTICE:  LOOP: 6
```
___