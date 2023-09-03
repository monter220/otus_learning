- [x] Настройте выполнение контрольной точки раз в 30 секунд.
___
*__В настройках выставил `checkpoint_timeout = 30s` и перезапустил кластер__*
___
- [x] 10 минут c помощью утилиты pgbench подавайте нагрузку.
___
*__Выполнил__*
```commandline
sudo -u postgres pgbench -i postgres
sudo -u postgres pgbench -c8 -P 60 -T 600 -U postgres postgres
```
___
- [x] Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.
___
*__До__*
```commandline
17M     /var/lib/postgresql/15/main/pg_wal
```
*__После__*
```commandline
81M     /var/lib/postgresql/15/main/pg_wal
```
*__На одну контрольную точку приходится в среднем `12mb`__*

___
- [x] Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?
___
*__Согласно отчету полученному командой:__*
```commandline
sudo -u postgres psql -c "select * from pg_stat_bgwriter;"
```
*__Получаем, что по статистике среднее время выполнения контрольной точки - 120 секунд,
а пытаемся делать точки каждые 25 секунд, то есть время выполнения 1 КТ накладывается на время старта выполнения следующих.
А следовательно при такой нагрузке нет смысла так часто делать контрольные точки__*
___
- [x] Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.
___
*__По умолчанию имеем `synchronous_commit = on` следовательно выполняем команду для проверки с включенным синхронным режимом__*
```commandline
sudo -u postgres pgbench -P 1 -T 10 postgres
pgbench (15.4 (Ubuntu 15.4-1.pgdg20.04+1))
starting vacuum...end.
progress: 1.0 s, 358.9 tps, lat 2.761 ms stddev 0.286, 0 failed
progress: 2.0 s, 370.0 tps, lat 2.701 ms stddev 0.146, 0 failed
progress: 3.0 s, 423.0 tps, lat 2.367 ms stddev 0.125, 0 failed
progress: 4.0 s, 387.0 tps, lat 2.583 ms stddev 0.153, 0 failed
progress: 5.0 s, 400.0 tps, lat 2.498 ms stddev 0.108, 0 failed
progress: 6.0 s, 381.0 tps, lat 2.619 ms stddev 0.170, 0 failed
progress: 7.0 s, 380.0 tps, lat 2.630 ms stddev 0.128, 0 failed
progress: 8.0 s, 422.0 tps, lat 2.370 ms stddev 0.252, 0 failed
progress: 9.0 s, 391.0 tps, lat 2.559 ms stddev 0.174, 0 failed
progress: 10.0 s, 419.1 tps, lat 2.382 ms stddev 0.130, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 3933
number of failed transactions: 0 (0.000%)
latency average = 2.540 ms
latency stddev = 0.220 ms
initial connection time = 6.859 ms
tps = 393.542540 (without initial connection time)
```
*__Меняем на асинхронный режим и повторяем эксперимент__*
```commandline
echo "synchronous_commit = off" | sudo tee -a /etc/postgresql/15/main/postgresql.conf
synchronous_commit = off
sudo systemctl restart postgresql

sudo -u postgres pgbench -P 1 -T 10 postgres                                pgbench (15.4 (Ubuntu 15.4-1.pgdg20.04+1))
starting vacuum...end.
progress: 1.0 s, 453.0 tps, lat 2.188 ms stddev 0.216, 0 failed
progress: 2.0 s, 460.0 tps, lat 2.171 ms stddev 0.089, 0 failed
progress: 3.0 s, 526.0 tps, lat 1.901 ms stddev 0.227, 0 failed
progress: 4.0 s, 530.0 tps, lat 1.888 ms stddev 0.253, 0 failed
progress: 5.0 s, 450.0 tps, lat 2.218 ms stddev 0.215, 0 failed
progress: 6.0 s, 497.0 tps, lat 2.012 ms stddev 0.242, 0 failed
progress: 7.0 s, 598.0 tps, lat 1.672 ms stddev 0.139, 0 failed
progress: 8.0 s, 539.0 tps, lat 1.854 ms stddev 0.272, 0 failed
progress: 9.0 s, 739.0 tps, lat 1.353 ms stddev 0.206, 0 failed
progress: 10.0 s, 583.0 tps, lat 1.714 ms stddev 0.413, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 5376
number of failed transactions: 0 (0.000%)
latency average = 1.858 ms
latency stddev = 0.362 ms
initial connection time = 7.062 ms
tps = 537.876522 (without initial connection time)
```
*__Критической разницы на таком эксперименте увидеть не удалось, однако
разница имеется. Теперь видим более эффективное сбрасывание на диск, то есть вместо того что бы писать на диск каждое изменение, 
мы делаем это отдельным процессом, по расписанию.__*
___

- [x] Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. 
Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. 
Что и почему произошло? как проигнорировать ошибку и продолжить работу?
___
*__Выполнил__*
```commandline
sudo pg_createcluster 15 test -p 5433
sudo pg_ctlcluster 15 test start
sudo -u postgres psql -p 5433 -c "create table messages(message text)"
CREATE TABLE
sudo -u postgres psql -p 5433 -c "insert into messages (message) values ('hello')"
INSERT 0 1
sudo -u postgres psql -p 5433 -c "insert into messages (message) values ('hello2')"
INSERT 0 1
sudo -u postgres psql -p 5433 -c "SELECT pg_relation_filepath('messages');"
 pg_relation_filepath
----------------------
 base/5/16384
(1 row)
sudo pg_ctlcluster 15 test stop


sudo dd if=/dev/zero of=/var/lib/postgresql/15/test/base/5/16384 oflag=dsync conv=notrunc bs=1 count=8
8+0 records in
8+0 records out
8 bytes copied, 0.00401568 s, 2.0 kB/s

sudo pg_ctlcluster 15 test start

sudo -u postgres psql -p 5433 -c "select * from messages"
WARNING:  page verification failed, calculated checksum 40176 but expected 64197
ERROR:  invalid page in block 0 of relation base/13427/16384
```
*__`data checksums` гарантирует целостность на уровне байтов в файлах и ругается о том что файл был поврежден__*

*__Для решения возникшей проблемы необходимо выставить настройку зануляющую поврежденные строки и выполнить полный вакуум__*
```
SET zero_damaged_pages = on;
vacuum full messages;
select * from messages;
```
*__При этом поврежденная строка - потеряна__*
___
