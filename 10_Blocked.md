- [x] Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, 
удерживаемых более 200 миллисекунд. 
Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.
___
*__Для подготовки удалил все предыдущие кластеры ПГ, создал новый и выполнил базовую настройку__*
```commandline
~$ sudo -u postgres pg_createcluster 15 main
~$ sudo pg_ctlcluster 15 main start
~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
~$ sudo -u postgres psql -c "ALTER SYSTEM SET deadlock_timeout TO 200"
ALTER SYSTEM
~$ sudo -u postgres psql -c "select pg_reload_conf()"
 pg_reload_conf
----------------
 t
(1 row)
~$ sudo -u postgres psql -c "show deadlock_timeout"
 deadlock_timeout
------------------
 200ms
(1 row)
~$ sudo -u postgres psql -c "create table test(id int primary key,text text)"  
CREATE TABLE
~$ sudo -u postgres psql -c "insert into test values (1, 'text-1')"
INSERT 0 1
~$ sudo -u postgres psql -c "insert into test values (2, 'text-2')"
INSERT 0 1
```
*__Открываем 2 сессии и пытаемся выполнить апдейты__*

| Сессия 1                                                       | Сессия 2                                                       |
|----------------------------------------------------------------|----------------------------------------------------------------|
| ~$ sudo -u postgres psql                                       | ~$ sudo -u postgres psql                                       |
| =# BEGIN;<br/>BEGIN                                            | =# BEGIN;<br/>BEGIN                                            |
| =*# SELECT text FROM test WHERE id = 2 FOR UPDATE;             | =*# SELECT text FROM test WHERE id = 1 FOR UPDATE;             |
| text<br/>--------<br/> text-2<br/>(1 row)                      | text<br/>--------<br/> text-1<br/>(1 row)                      |
|                                                                | =*# SELECT pg_sleep(10);                                       |
| =*# UPDATE test SET text = 'text from session 2' WHERE id = 1; | =*# UPDATE test SET text = 'text from session 1' WHERE id = 2; |
| ERROR:  deadlock detected                                      | UPDATE 1                                                       |
| =*# commit;<br/>ROLLBACK                                       | =*# commit;<br/>COMMIT                                         |
 
*__Cессия 1 оборвалась с ошибкой__*

```commandline
ERROR:  deadlock detected
DETAIL:  Process 337768 waits for ShareLock on transaction 740; blocked by process 337602.
Process 337602 waits for ShareLock on transaction 741; blocked by process 337768.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "test"
```
*__При проверке содержимого таблицы видим, что изменения есть только от сессии 2__*
```commandline
=# select * from test;
 id |        text
----+---------------------
  1 | text-1
  2 | text from session 1
(2 rows)
```
*__В логе видим__*
```commandline
~$ sudo cat /var/log/postgresql/postgresql-15-main.log | grep "deadlock detected" -A 10
2023-09-06 17:56:03.251 UTC [337768] postgres@postgres ERROR:  deadlock detected
2023-09-06 17:56:03.251 UTC [337768] postgres@postgres DETAIL:  Process 337768 waits for ShareLock on transaction 740; blocked by process 337602.
        Process 337602 waits for ShareLock on transaction 741; blocked by process 337768.
        Process 337768: UPDATE test SET text = 'text from session 2' WHERE id = 1;
        Process 337602: UPDATE test SET text = 'text from session 1' WHERE id = 2;
2023-09-06 17:56:03.251 UTC [337768] postgres@postgres HINT:  See server log for query details.
2023-09-06 17:56:03.251 UTC [337768] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "test"
2023-09-06 17:56:03.251 UTC [337768] postgres@postgres STATEMENT:  UPDATE test SET text = 'text from session 2' WHERE id = 1;
2023-09-06 17:58:00.972 UTC [337768] postgres@postgres ERROR:  current transaction is aborted, commands ignored until end of transaction block
```
___
- [x] Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. 
Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. 
Пришлите список блокировок и объясните, что значит каждая.
___
*__Для подготовки удалил таблицу, созданную для предыдущего шага и создал новую__*
```commandline
~$ sudo -u postgres psql -c "drop table test"
DROP TABLE
~$ sudo -u postgres psql -c "create table test(id int primary key,text text)"
CREATE TABLE
~$ sudo -u postgres psql -c "insert into test values (1, 'text-1')"
INSERT 0 1
```
*__Переходим к воспроизведению ситуации. Открываем 3 сеанса и выполняем во всех 3х__*
```commandline
~$ sudo -u postgres psql
=# BEGIN;
BEGIN
=*# SELECT pg_backend_pid() as pid, txid_current() as tid;
=*# UPDATE test SET text = 'text from session 1(2,3)' WHERE id = 1;
```
| Сессия | pid      | tid |
|--------|----------|-----|
| 1      | 377755   | 745 |
| 2      | 378847   | 748 |
| 3      | 378171   | 747 |

___
*__Проверяем блокировки__*
```commandline
~$ SELECT blocked_locks.pid AS blocked_pid,
         blocked_activity.usename  AS blocked_user,
         blocking_locks.pid     AS blocking_pid,
         blocking_activity.usename AS blocking_user,
         blocked_activity.query    AS blocked_statement,
         blocking_activity.query   AS current_statement_in_blocking_process,
         blocked_activity.application_name AS blocked_application,
         blocking_activity.application_name AS blocking_application
   FROM  pg_catalog.pg_locks         blocked_locks
    JOIN pg_catalog.pg_stat_activity blocked_activity  ON blocked_activity.pid = blocked_locks.pid
    JOIN pg_catalog.pg_locks         blocking_locks 
        ON blocking_locks.locktype = blocked_locks.locktype
        AND blocking_locks.DATABASE IS NOT DISTINCT FROM blocked_locks.DATABASE
        AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
        AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
        AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
        AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
        AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
        AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
        AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
        AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
        AND blocking_locks.pid != blocked_locks.pid
 
    JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
   WHERE NOT blocked_locks.GRANTED;
```
| blocked_pid | blocked_user | blocking_pid | blocking_user |                     blocked_statement                      |           current_statement_in_blocking_process            | blocked_application | blocking_application |
|-------------|--------------|--------------|---------------|------------------------------------------------------------|------------------------------------------------------------|---------------------|----------------------|
|      378171 | postgres     |       377755 | postgres      | UPDATE test SET text = 'text from session 3' WHERE id = 1; | UPDATE test SET text = 'text from session 1' WHERE id = 1; | psql                | psql                 |
|      378847 | postgres     |       378171 | postgres      | UPDATE test SET text = 'text from session 2' WHERE id = 1; | UPDATE test SET text = 'text from session 3' WHERE id = 1; | psql                | psql                 |

*__Видим, что 377755 блокирует 378171, а та в свою очередь блокирует 378847__*
___
*__Для получения списка блокировок и объяснения значений выполним запрос__*
```commandline
SELECT 
row_number() over(ORDER BY pid, virtualxid, transactionid::text::bigint) as n,
CASE
WHEN locktype = 'relation' THEN 'отношение'
WHEN locktype = 'extend' THEN 'расширение отношения'
WHEN locktype = 'frozenid' THEN 'замороженный идентификатор'
WHEN locktype = 'page' THEN 'страница'
WHEN locktype = 'tuple' THEN 'кортеж'
WHEN locktype = 'transactionid' THEN 'идентификатор транзакции'
WHEN locktype = 'virtualxid' THEN 'виртуальный идентификатор'
WHEN locktype = 'object' THEN 'объект'
WHEN locktype = 'userlock' THEN 'пользовательская блокировка'
WHEN locktype = 'advisory' THEN 'рекомендательная'
END AS locktype,

relation::regclass,

CASE WHEN page IS NOT NULL AND tuple IS NOT NULL THEN (select text from test m where m.ctid::text = '(' || page || ',' || tuple || ')' limit 1) ELSE NULL END AS row, 

virtualxid, transactionid, virtualtransaction, 

pid, 
CASE WHEN pid = 377755 THEN 'Сессия 1' WHEN pid = 378847 THEN 'Сессия 2' WHEN pid = 378171 THEN 'Сессия 3' END AS session, 

mode, 

CASE WHEN granted = true THEN 'блокировка получена' ELSE 'блокировка ожидается' END AS granted,
CASE WHEN fastpath = true THEN 'блокировка получена по короткому пути' ELSE 'блокировка получена через основную таблицу блокировок' END AS fastpath 
FROM pg_locks WHERE pid in (377755, 378847,378171) 
ORDER BY pid, virtualxid, transactionid::text::bigint;
```
| n   | locktype                  | relation  | row    | virtualxid | transactionid | virtualtransaction |  pid   | session  |       mode       |       granted        | fastpath                                              |
|-----|---------------------------|-----------|--------|------------|---------------|--------------------|--------|----------|------------------|----------------------|-------------------------------------------------------|
| 1   | виртуальный идентификатор |           |        | 3/5087     |               | 3/5087             | 377755 | Сессия 1 | ExclusiveLock    | блокировка получена  | блокировка получена по короткому пути                 |
| 2   | идентификатор транзакции  |           |        |            | 745           | 3/5087             | 377755 | Сессия 1 | ExclusiveLock    | блокировка получена  | блокировка получена через основную таблицу блокировок |
| 3   | отношение                 | test      |        |            |               | 3/5087             | 377755 | Сессия 1 | RowExclusiveLock | блокировка получена  | блокировка получена по короткому пути                 |
| 4   | отношение                 | test_pkey |        |            |               | 3/5087             | 377755 | Сессия 1 | RowExclusiveLock | блокировка получена  | блокировка получена по короткому пути                 |
| 5   | виртуальный идентификатор |           |        | 4/50       |               | 4/50               | 378171 | Сессия 3 | ExclusiveLock    | блокировка получена  | блокировка получена по короткому пути                 |
| 6   | идентификатор транзакции  |           |        |            | 745           | 4/50               | 378171 | Сессия 3 | ShareLock        | блокировка ожидается | блокировка получена через основную таблицу блокировок |
| 7   | идентификатор транзакции  |           |        |            | 747           | 4/50               | 378171 | Сессия 3 | ExclusiveLock    | блокировка получена  | блокировка получена через основную таблицу блокировок |
| 8   | отношение                 | test      |        |            |               | 4/50               | 378171 | Сессия 3 | RowExclusiveLock | блокировка получена  | блокировка получена по короткому пути                 |
| 9   | отношение                 | test_pkey |        |            |               | 4/50               | 378171 | Сессия 3 | RowExclusiveLock | блокировка получена  | блокировка получена по короткому пути                 |
| 10  | кортеж                    | test      | text-1 |            |               | 4/50               | 378171 | Сессия 3 | ExclusiveLock    | блокировка получена  | блокировка получена через основную таблицу блокировок |
| 11  | виртуальный идентификатор |           |        | 5/98       |               | 5/98               | 378847 | Сессия 2 | ExclusiveLock    | блокировка получена  | блокировка получена по короткому пути                 |
| 12  | идентификатор транзакции  |           |        |            | 748           | 5/98               | 378847 | Сессия 2 | ExclusiveLock    | блокировка получена  | блокировка получена через основную таблицу блокировок |
| 13  | кортеж                    | test      | text-1 |            |               | 5/98               | 378847 | Сессия 2 | ExclusiveLock    | блокировка ожидается | блокировка получена через основную таблицу блокировок |
| 14  | отношение                 | test      |        |            |               | 5/98               | 378847 | Сессия 2 | RowExclusiveLock | блокировка получена  | блокировка получена по короткому пути                 |
| 15  | отношение                 | test_pkey |        |            |               | 5/98               | 378847 | Сессия 2 | RowExclusiveLock | блокировка получена  | блокировка получена по короткому пути                 |

- каждый сеанс держит эксклюзивные (exclusive lock) блокировки на номера своих транзакций (transactionid - 2, 7, 12 строки) и виртуальной транзакции (virtualxid - 1, 5, 11 - строки)
- первый сеанс выставил эксклюзивную блокировку строки для ключа и самой строки, строки 3, 4
- оставшиеся два запроса ожидают блокировки, но выставили эксклюзивную блокировку на ключ и строку (строки - 8, 9 и 14, 15)
- так же оставшиеся два сеанса повесили экслоюзивную блокировку на сам кортеж, т.к. хотят обновить именно его, а он уже обновлен в первом сеансе (строки 10 и 13)
- оставшаяся блокировка share lock в 6 строке вызванна тем что мы пытаемся обновить ту же строку что и в первом сеансе у которого уже выставлена эксклюзивная блокировка строки
___
- [x] Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?
___
*__Для подготовки удалил таблицу, созданную для предыдущего шага и создал новую__*
```commandline
~$ sudo -u postgres psql -c "drop table test"
DROP TABLE
~$ sudo -u postgres psql -c "create table test(id int primary key,text text)"
CREATE TABLE
~$ sudo -u postgres psql -c "insert into test values (1, 'text-1')"
INSERT 0 1
~$ sudo -u postgres psql -c "insert into test values (2, 'text-2')"
INSERT 0 1
~$ sudo -u postgres psql -c "insert into test values (3, 'text-3')"
INSERT 0 1
```
*__Создаем взаимоблокировку 3х транзакций__*

| Сессия 1                                                                                                        | Сессия 2                                                                                                        | Сессия 3                                                                                                        |
|-----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| =# BEGIN;<br/> BEGIN                                                                                            | =# BEGIN;<br/> BEGIN                                                                                            | =# BEGIN;<br/> BEGIN                                                                                            |
| =*# SELECT pg_backend_pid() as pid, txid_current() as tid;<br/>  pid / tid<br/>--------/-----<br/> 380092 / 757 | =*# SELECT pg_backend_pid() as pid, txid_current() as tid;<br/>  pid / tid<br/>--------/-----<br/> 380117 / 758 | =*# SELECT pg_backend_pid() as pid, txid_current() as tid;<br/>  pid / tid<br/>--------/-----<br/> 380126 / 759 |
| =*# SELECT text FROM test WHERE id = 1 FOR UPDATE;                                                              | =*# SELECT text FROM test WHERE id = 2 FOR UPDATE;                                                              | =*# SELECT text FROM test WHERE id = 3 FOR UPDATE;                                                              |
| =*# UPDATE test SET text = 'text from session 1' WHERE id = 2;                                                  | =*# UPDATE test SET text = 'text from session 2' WHERE id = 3;                                                  | =*# UPDATE test SET text = 'text from session 3' WHERE id = 1;                                                  |
|                                                                                                                 | UPDATE 1                                                                                                        | ERROR:  deadlock detected                                                                                       |

*__В результате имеем:__*
 - первый сеанс висит на апдейте
 - второй сеанс обновил строку
 - третий сеанс вылетел с ошибкой
```commandline
ERROR:  deadlock detected
DETAIL:  Process 380126 waits for ShareLock on transaction 757; blocked by process 380092.
Process 380092 waits for ShareLock on transaction 758; blocked by process 380117.
Process 380117 waits for ShareLock on transaction 759; blocked by process 380126.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "test"
```
*__В логе видим__*
```commandline
~$ sudo cat /var/log/postgresql/postgresql-15-main.log | grep "deadlock detected" -A 10
2023-09-07 16:13:35.677 UTC [380126] postgres@postgres ERROR:  deadlock detected
2023-09-07 16:13:35.677 UTC [380126] postgres@postgres DETAIL:  Process 380126 waits for ShareLock on transaction 757; blocked by process 380092.
        Process 380092 waits for ShareLock on transaction 758; blocked by process 380117.
        Process 380117 waits for ShareLock on transaction 759; blocked by process 380126.
        Process 380126: UPDATE test SET text = 'text from session 3' WHERE id = 1;
        Process 380092: UPDATE test SET text = 'text from session 1' WHERE id = 2;
        Process 380117: UPDATE test SET text = 'text from session 2' WHERE id = 3;
2023-09-07 16:13:35.677 UTC [380126] postgres@postgres HINT:  See server log for query details.
2023-09-07 16:13:35.677 UTC [380126] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "test"
2023-09-07 16:13:35.677 UTC [380126] postgres@postgres STATEMENT:  UPDATE test SET text = 'text from session 3' WHERE id = 1;
2023-09-07 16:17:08.264 UTC [379967] LOG:  checkpoint starting: time
```
*__В логе мы видим:__* 
 - процесс № 380126 (третий сеанс) - deadlock
 
*__Детали нам говорят о том что:__*
 - третий сенас ждал первого и второго, сеанс два при этом ждал третьего (образовалось кольцо)
 - В логе приведены запросы, но из-за того что блокировку мы вызвали ранее по ним можно только сказать что мы пытались обновить, но не причину блокировки
___
- [x] Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
___
*__Для подготовки удалил таблицу, созданную для предыдущего шага и создал новую__*
```commandline
~$ sudo -u postgres psql -c "drop table test"
DROP TABLE
~$ sudo -u postgres psql -c "create table test(id integer primary key generated always as identity, n float)"
CREATE TABLE
~$ sudo -u postgres psql -c "insert into test(n) select random() from generate_series(1,1000000)"
INSERT 0 1000000
```
*__Пытаемся проверить гипотезу, что возможна блокировка__*

| Сессия 1                                                                          | Сессия 2                                                                       |
|-----------------------------------------------------------------------------------|--------------------------------------------------------------------------------|
| =# BEGIN;<br/>BEGIN                                                               | =# BEGIN;<br/>BEGIN                                                            | 
| =*# UPDATE test SET n = (select id from test order by id asc limit 1 for update); | UPDATE test SET n = (select id from test order by id desc limit 1 for update); | 
| ERROR:  deadlock detected                                                         | UPDATE 1000000                                                                 | 

*__Первая сессия завершилась ошибкой:__*
```commandline
ERROR:  deadlock detected
DETAIL:  Process 380092 waits for ShareLock on transaction 765; blocked by process 380117.
Process 380117 waits for ShareLock on transaction 764; blocked by process 380092.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (5405,75) in relation "test"
```

*__Для симуляции случайной взаимоблокировке двумя апдейтами мы в первой сессии обновляем строки по возрастанию id, а во второй по убыванию.
В начале выполнение 2х запросов начинается без каких-либо аномалий, но при пересечении id происходит deadlock__*

___