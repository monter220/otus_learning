- [x] Создаем БД, схему и в ней таблицу. Заполним таблицы автосгенерированными 100 записями. 
Под линукс пользователем Postgres создадим каталог для бэкапов
___
*__Вполняем подготовительные операции__*
```commandline
~$ sudo -u postgres pg_createcluster 15 main
~$ sudo pg_ctlcluster 15 main start
~$ sudo mkdir /srv/otus
~$ sudo chown postgres /srv/otus/
/srv$ ls -l
total 4
drwxr-xr-x 2 postgres root 4096 Sep 24 09:51 otus
~$ sudo -u postgres psql
postgres=# create database otus;
CREATE DATABASE
postgres=# \c otus
You are now connected to database "otus" as user "postgres".
otus=#
otus=# create table test(id integer primary key generated always as identity, n float);
otus=# insert into test(n) select random() from generate_series(1,100);
INSERT 0 100
```
___
- [x] Сделаем логический бэкап используя утилиту COPY
___
*__Проведем логическое коипирование в файл командой__*
```commandline
otus=# \copy test to '/srv/otus/test.sql' with delimiter ',';
COPY 100
```
___
- [x] Восстановим в 2 таблицу данные из бэкапа.
___
*__Проведем восстановление из файла командой__*
```commandline
otus=# \copy test2 from '/srv/otus/test.sql' with delimiter ',';
ERROR:  relation "test2" does not exist
```
*__повторяем восстановление предварительно создав таблицу__*
```commandline
otus=# create table test2(id integer primary key generated always as identity, n float);
CREATE TABLE
otus=# \copy test2 from '/srv/otus/test.sql' with delimiter ',';
COPY 100
```
___
- [x] Используя утилиту pg_dump создадим бэкап в кастомном сжатом формате двух таблиц
___
*__Создадим бэкап командой__*
```commandline
~$ sudo -u postgres pg_dump -d otus --create -U postgres -Fc > /srv/otus/arh.gz
```
___
- [x] Используя утилиту pg_restore восстановим в новую БД только вторую таблицу!
___
*__Создадим новую БД и восстановим только test2__*
```commandline
~$ sudo -u postgres psql -c  "drop database otus2"                           
DROP DATABASE
~$ sudo -u postgres psql -c  "CREATE DATABASE otus"
CREATE DATABASE
~$ sudo -u postgres pg_restore -d otus --table=test2 -U postgres /srv/otus/arh.gz
~$ sudo -u postgres psql
postgres=# \c otus
otus=# \dt
         List of relations
 Schema | Name  | Type  |  Owner
--------+-------+-------+----------
 public | test2 | table | postgres
(1 row)
```
___