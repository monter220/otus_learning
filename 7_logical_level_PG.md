- [x] создайте новый кластер PostgresSQL 14
___
*__Для установки PG 14 использовал команды__*
```commandline
sudo apt update 
sudo apt upgrade -y 
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' 
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - 
sudo apt update 
sudo apt -y install postgresql-14
```
___
- [x] зайдите в созданный кластер под пользователем postgres
___
*__Поскольку теперь у нас 2 кластера для подключения к нужному использовал__*
```commandline
sudo pg_ctlcluster 14 main start
sudo su postgres
psql --port=5433
```
___
- [x] создайте новую базу данных testdb
___
*__Выполнил__*
```commandline
CREATE DATABASE testdb;
```
___
- [x] зайдите в созданную базу данных под пользователем postgres
___
*__Выполнил__*
```commandline
\c testdb
psql (15.4 (Ubuntu 15.4-1.pgdg20.04+1), server 14.9 (Ubuntu 14.9-1.pgdg20.04+1))
You are now connected to database "testdb" as user "postgres".
```
___
- [x] создайте новую схему testnm
___
*__Выполнил__*
```commandline
CREATE SCHEMA testnm;
```
___
- [x] создайте новую таблицу t1 с одной колонкой c1 типа integer
___
*__Выполнил__*
```commandline
CREATE TABLE t1(c1 integer);
```
___
- [x] вставьте строку со значением c1=1
___
*__Выполнил__*
```commandline
INSERT INTO t1 values(1);
```
___
- [x] создайте новую роль readonly
___
*__Выполнил__*
```commandline
CREATE role readonly;
```
___
- [x] дайте новой роли право на подключение к базе данных testdb
___
*__Выполнил__*
```commandline
grant connect on DATABASE testdb TO readonly;
```
___
- [x] дайте новой роли право на использование схемы testnm
___
*__Выполнил__*
```commandline
grant usage on SCHEMA testnm to readonly;
```
___
- [x] дайте новой роли право на select для всех таблиц схемы testnm
___
*__Выполнил__*
```commandline
grant SELECT on all TABLEs in SCHEMA testnm TO readonly;
```
___
- [x] создайте пользователя testread с паролем test123
___
*__Выполнил__*
```commandline
CREATE USER testread with password 'test123';
```
___
- [x] дайте роль readonly пользователю testread
___
*__Выполнил__*
```commandline
grant readonly TO testread;
```
___
- [x] зайдите под пользователем testread в базу данных testdb
___
*__Выполнил__*
```commandline
psql -h 127.0.0.1 -U testread -d testdb -W --port=5433
```
___
- [x] сделайте `select * from t1;`  получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)
___
*__Выполнил__*
```commandline
select * from t1;
ERROR:  permission denied for table t1
```
*__При создании таблицы не указывали в какой схеме создавать, поэтому она создана в схеме publik,
а к ней не давали права__*

___
- [x] вернитесь в базу данных testdb под пользователем postgres
___
*__Выполнил__*
```commandline
\q
 psql --port=5433 --dbname=testdb
```
___
- [x] удалите таблицу t1
___
*__Выполнил__*
```commandline
DROP TABLE t1;
```
___
- [x] создайте ее заново но уже с явным указанием имени схемы testnm
___
*__Выполнил__*
```commandline
CREATE TABLE testnm.t1(c1 integer);
```
___
- [x] вставьте строку со значением c1=1
___
*__Выполнил__*
```commandline
INSERT INTO testnm.t1 values(1);
```
___
- [x] зайдите под пользователем testread в базу данных testdb
___
*__Выполнил__*
```commandline
\q
psql -h 127.0.0.1 -U testread -d testdb -W --port=5433
```
___
- [x] сделайте `select * from t1;`  получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)
___
- [x] сделайте `select * from testnm.t1;`. Получилось?
___
*__Выполнил__*
```commandline
select * from testnm.t1;
ERROR:  permission denied for table t1
```
*__Выдача прав распространяется на уже созданные таблицы. 
Таблицу `testnm.t1` создали после выдачи прав, 
следовательно права на нее не распространились.
Для исправления необходимо выдать снова права пользователю на таблицу__*

*__Для этого необходимо выполнить__*
```commandline
\q
psql --port=5433 --dbname=testdb
grant SELECT on all TABLEs in SCHEMA testnm TO readonly;
```
*__Всем текущим таблицам будут выданы права на чтение, но новые таблицы будут недоступны.
Альтернативный способ__*
```commandline
\q
psql --port=5433 --dbname=testdb
ALTER default privileges in SCHEMA testnm grant SELECT on TABLES to readonly; 
DROP TABLE t1;
CREATE TABLE testnm.t1(c1 integer);
INSERT INTO testnm.t1 values(1);
```
*__Заклинание `ALTER default` выдаст права на созданные в дальнейшем таблицы__*
___
- [x] сделайте `select * from testnm.t1;`
___
*__Выполнил__*
```commandline
select * from testnm.t1;
 c1
----
  1
(1 row)
```
___
- [x] теперь попробуйте выполнить команду `create table t2(c1 integer); insert into t2 values (2);`
___
*__Ошибки доступа не возникло.__*

*__Поумолчанию используется схему `public` и при создании новой БД 
по умолчанию всем пользователям присваивается роль public, 
которая имеет досаточно прав на создание таблиц.
Для исправления такой неприятности необходимо внести правку командами:__*

```commandline
\q
psql --port=5433 --dbname=testdb
REVOKE CREATE on SCHEMA public FROM public;
REVOKE ALL on DATABASE testdb FROM public;
```
___
- [x] теперь попробуйте выполнить команду `create table t3(c1 integer); insert into t2 values (2);`
___
*__Выполнил__*
```commandline
\q
psql -h 127.0.0.1 -U testread -d testdb -W --port=5433
create table t3(c1 integer);
ERROR:  permission denied for schema public
LINE 1: create table t3(c1 integer);
```
*__Для создания новых таблиц теперь не хватает прав__*
___
