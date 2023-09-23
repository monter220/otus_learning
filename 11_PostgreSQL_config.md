- [x] настроить кластер PostgreSQL 15 на максимальную производительность не
обращая внимание на возможные проблемы с надежностью в случае
аварийной перезагрузки виртуальной машины
___

*__Имеем ВМ с ОС Ubuntu 20.04 со следующими параметрами__*
```commandline
Number of CPUs - 8
RAM - 4 Gb
ROM - 50 Gb
```

*__Создал кластер и выполнил базовый тест производительности__*

```commandline
~$ sudo -u postgres pg_createcluster 15 main
~$ sudo pg_ctlcluster 15 main start
~$ sudo -u postgres pgbench -i postgres
~$ sudo -u postgres pgbench -c 50 -j 2 -P 10 -T 60 postgres
pgbench (15.4 (Ubuntu 15.4-1.pgdg20.04+1))
starting vacuum...end.
progress: 10.0 s, 676.5 tps, lat 71.172 ms stddev 76.834, 0 failed
progress: 20.0 s, 703.7 tps, lat 70.985 ms stddev 81.957, 0 failed
progress: 30.0 s, 697.7 tps, lat 71.709 ms stddev 84.290, 0 failed
progress: 40.0 s, 699.1 tps, lat 71.653 ms stddev 72.560, 0 failed
progress: 50.0 s, 707.8 tps, lat 70.576 ms stddev 82.384, 0 failed
progress: 60.0 s, 687.0 tps, lat 72.805 ms stddev 91.295, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 41767
number of failed transactions: 0 (0.000%)
latency average = 71.526 ms
latency stddev = 81.803 ms
initial connection time = 282.464 ms
tps = 698.577137 (without initial connection time)
```

*__Определяем рекомендуемые параметры от портала https://pgtune.leopard.in.ua/__*

```commandline
# DB Version: 15
# OS Type: linux
# DB Type: desktop
# Total Memory (RAM): 4 GB
# CPUs num: 8
# Data Storage: hdd

shared_buffers = 256MB
effective_cache_size = 1GB
maintenance_work_mem = 256MB
wal_buffers = 7864kB
```
*__Устанавливаем рекомендуемые параметры и повторяем тест__*

```commandline
~$ sudo -u postgres pgbench -c 50 -j 2 -P 10 -T 60 postgres
pgbench (15.4 (Ubuntu 15.4-1.pgdg20.04+1))
starting vacuum...end.
progress: 10.0 s, 685.3 tps, lat 70.982 ms stddev 79.028, 0 failed
progress: 20.0 s, 667.0 tps, lat 73.876 ms stddev 83.966, 0 failed
progress: 30.0 s, 691.0 tps, lat 72.969 ms stddev 88.064, 0 failed
progress: 40.0 s, 690.9 tps, lat 72.736 ms stddev 89.628, 0 failed
progress: 50.0 s, 633.3 tps, lat 79.006 ms stddev 87.599, 0 failed
progress: 60.0 s, 684.3 tps, lat 73.111 ms stddev 83.759, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 40567
number of failed transactions: 0 (0.000%)
latency average = 73.753 ms
latency stddev = 85.432 ms
initial connection time = 197.169 ms
tps = 677.491738 (without initial connection time)
```

*__На рекомендуемых параметрах tps незначительно уменьшилось__*

*__Выставляем безумные параметры, пробуем повторить тест и надеемся, что ВМ не упадет__*
```commandline
shared_buffers = 2GB
effective_cache_size = 3GB
maintenance_work_mem = 1GB
wal_buffers = 256MB

~$ sudo -u postgres pgbench -c 50 -j 2 -P 10 -T 60 postgres
pgbench (15.4 (Ubuntu 15.4-1.pgdg20.04+1))
starting vacuum...end.
progress: 10.0 s, 668.4 tps, lat 72.525 ms stddev 74.281, 0 failed
progress: 20.0 s, 684.4 tps, lat 73.069 ms stddev 82.977, 0 failed
progress: 30.0 s, 693.9 tps, lat 72.265 ms stddev 78.940, 0 failed
progress: 40.0 s, 692.2 tps, lat 72.193 ms stddev 85.897, 0 failed
progress: 50.0 s, 687.6 tps, lat 72.526 ms stddev 82.486, 0 failed
progress: 60.0 s, 685.9 tps, lat 72.979 ms stddev 81.940, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 41174
number of failed transactions: 0 (0.000%)
latency average = 72.645 ms
latency stddev = 81.224 ms
initial connection time = 215.852 ms
tps = 687.777060 (without initial connection time)
```

*__В процессе данных экспериментов не удалось увидеть улучшений, даже замечены просадки tps. Пробуем еще менять параметры__*

```commandline
shared_buffers = 3GB
huge_pages = off

~$ sudo -u postgres pgbench -c 50 -j 2 -P 10 -T 60 postgres
pgbench (15.4 (Ubuntu 15.4-1.pgdg20.04+1))
starting vacuum...end.
progress: 10.0 s, 673.5 tps, lat 72.243 ms stddev 76.775, 0 failed
progress: 20.0 s, 694.2 tps, lat 71.965 ms stddev 81.564, 0 failed
progress: 30.0 s, 701.4 tps, lat 71.247 ms stddev 83.773, 0 failed
progress: 40.0 s, 682.2 tps, lat 73.060 ms stddev 83.573, 0 failed
progress: 50.0 s, 676.7 tps, lat 74.079 ms stddev 84.211, 0 failed
progress: 60.0 s, 702.5 tps, lat 71.165 ms stddev 78.929, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 41355
number of failed transactions: 0 (0.000%)
latency average = 72.329 ms
latency stddev = 81.558 ms
initial connection time = 208.971 ms
tps = 690.796994 (without initial connection time)
```

```commandline
max_wal_size = 8GB
min_wal_size = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 2GB
wal_buffers = 1GB

~$ sudo -u postgres pgbench -c 50 -j 2 -P 10 -T 60 postgres
pgbench (15.4 (Ubuntu 15.4-1.pgdg20.04+1))
starting vacuum...end.
progress: 10.0 s, 719.3 tps, lat 67.681 ms stddev 79.045, 0 failed
progress: 20.0 s, 681.3 tps, lat 73.126 ms stddev 80.423, 0 failed
progress: 30.0 s, 707.7 tps, lat 70.809 ms stddev 76.586, 0 failed
progress: 40.0 s, 703.7 tps, lat 71.020 ms stddev 72.079, 0 failed
progress: 50.0 s, 701.0 tps, lat 71.351 ms stddev 82.837, 0 failed
progress: 60.0 s, 692.5 tps, lat 71.902 ms stddev 82.976, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 42104
number of failed transactions: 0 (0.000%)
latency average = 71.059 ms
latency stddev = 79.357 ms
initial connection time = 199.985 ms
tps = 703.128187 (without initial connection time)
```

```commandline
synchronous_commit = off

~$ sudo -u postgres pgbench -c 50 -j 2 -P 10 -T 60 postgres
pgbench (15.4 (Ubuntu 15.4-1.pgdg20.04+1))
starting vacuum...end.
progress: 10.0 s, 1157.3 tps, lat 42.197 ms stddev 55.092, 0 failed
progress: 20.0 s, 1126.7 tps, lat 44.179 ms stddev 51.629, 0 failed
progress: 30.0 s, 1091.2 tps, lat 45.921 ms stddev 58.040, 0 failed
progress: 40.0 s, 1113.5 tps, lat 44.928 ms stddev 54.059, 0 failed
progress: 50.0 s, 1122.2 tps, lat 44.370 ms stddev 52.565, 0 failed
progress: 60.0 s, 1139.3 tps, lat 43.891 ms stddev 59.650, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 67551
number of failed transactions: 0 (0.000%)
latency average = 44.275 ms
latency stddev = 55.309 ms
initial connection time = 201.574 ms
tps = 1128.493496 (without initial connection time)
```

*__Таким образом мы видим, что значительное влияние на производительность оказывает параметр `synchronous_commit`__*
*__Параметр `synchronous_commit` используется для обеспечения того, что фиксация транзакции будет ожидать записи 
WAL на диск, прежде чем вернуть клиенту статус успешного завершения. Выставленный в эксперименте параметр указывает, что 
транзакция фиксируется очень быстро, потому что она не будет ожидать сброса файла WAL, но надежность будет поставлена под угрозу. 
В случае сбоя сервера данные могут быть потеряны, даже если клиент получил сообщение об успешном завершении фиксации транзакции.__*