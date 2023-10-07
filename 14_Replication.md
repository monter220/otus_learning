**Имеем 4 ВМ**

| ВМ_name | ip              |
|---------|-----------------|
| pgotus  | 192.168.100.2   |
| pgotus2 | 192.168.100.181 |
| pgotus3 | 192.168.100.182 |
| pgotus4 | 192.168.100.183 |

**На всех ВМ выполняем**
```commandline
~$ sudo apt update 
~$ sudo apt upgrade -y 
~$ sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' 
~$ wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - 
~$ sudo apt-get update 
~$ sudo apt-get -y install postgresql-15
~$ echo "listen_addresses = '*'" | sudo tee -a /etc/postgresql/15/main/postgresql.conf
~$ echo "wal_level = 'logical'" | sudo tee -a /etc/postgresql/15/main/postgresql.conf
~$ echo "host all all 0.0.0.0/0 md5" | sudo tee -a /etc/postgresql/15/main/pg_hba.conf
~$ echo "host replication all 0.0.0.0/0 md5" | sudo tee -a /etc/postgresql/15/main/pg_hba.conf
~$ sudo systemctl restart postgresql
~$ sudo -u postgres psql -c "CREATE USER demo SUPERUSER encrypted PASSWORD '123'"
```
- [x] На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение.
___
**Минуя повторение выполняем команды на всех 3 ВМ**
```commandline
~$ sudo -u postgres psql -c "CREATE DATABASE demo"
CREATE DATABASE
~$ sudo -u postgres psql demo -c "CREATE TABLE test1 (m text)"
CREATE TABLE
~$ sudo -u postgres psql demo -c "CREATE TABLE test2 (m text)"
CREATE TABLE
```
___
- [x] Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2.
___
**Выполняем**
```commandline
@pgotus:~$ sudo -u postgres psql demo -c "CREATE PUBLICATION test1_pub FOR TABLE test1"
CREATE PUBLICATION
@pgotus2:~$ sudo -u postgres psql demo -c "CREATE SUBSCRIPTION test_sub CONNECTION 'host=192.168.100.2 port=5432 user=demo password=123 dbname=demo' PUBLICATION test1_pub WITH (copy_data = true)"
NOTICE:  created replication slot "test_sub" on publisher
CREATE SUBSCRIPTION
```
___
- [x] Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1.
___
**Выполняем команды**
```commandline
pgotus2:~$ sudo -u postgres psql demo -c "CREATE PUBLICATION test2_pub FOR TABLE test2"
CREATE PUBLICATION
pgotus:~$ sudo -u postgres psql demo -c "CREATE SUBSCRIPTION test2_sub CONNECTION 'host=192.168.100.181 port=5432 user=demo password=123 dbname=demo' PUBLICATION test2_pub WITH (copy_data = true)"
NOTICE:  created replication slot "test2_sub" on publisher
CREATE SUBSCRIPTION
```
**Проверяем, что все прошло успешно**
```commandline
pgotus:~$ sudo -u postgres psql demo -c "INSERT INTO test1 VALUES ('test1')"
INSERT 0 1
pgotus2:~$ sudo -u postgres psql demo -c "INSERT INTO test1 VALUES ('test2')"
INSERT 0 1
pgotus2:~$ sudo -u postgres psql demo -c "select * from test1"
   m
-------
 test1
 test2
(2 rows)
```
**Очевидно наблюдаем, что запутался в нейминге, но потратил минут 20 на осознание, в результате имеем
на первой ВМ в таблице test1 1 запись, а в таблице test2 ВМ2 2 записи...Считаю для данной лабы не критичным и проверяю обратную связь**
```commandline
pgotus2:~$ sudo -u postgres psql demo -c "INSERT INTO test2 VALUES ('test2')"
INSERT 0 1
pgotus:~$ sudo -u postgres psql demo -c "select * from test1"
   m
-------
 test1
(1 row)
```
___
- [x] 3 ВМ использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ).
___
**Выполняем**
```commandline
pgotus3:~$ sudo -u postgres psql demo -c "CREATE SUBSCRIPTION test_sub_for_hw9c CONNECTION 'host=192.168.100.2 port=5432 user=demo password=123 dbname=demo' PUBLICATION test1_pub WITH (copy_data = true)"
NOTICE:  created replication slot "test_sub_for_hw9c" on publisher
CREATE SUBSCRIPTION
pgotus3:~$ sudo -u postgres psql demo -c "CREATE SUBSCRIPTION test2_sub_for_hw9c CONNECTION 'host=192.168.100.181 port=5432 user=demo password=123 dbname=demo' PUBLICATION test2_pub WITH (copy_data = true)"
NOTICE:  created replication slot "test2_sub_for_hw9c" on publisher
CREATE SUBSCRIPTION
```
**Проверяем**
```commandline
pgotus3:~$ sudo -u postgres psql demo -c "select * from test1"
   m
-------
 test1
(1 row)

pgotus3:~$ sudo -u postgres psql demo -c "select * from test2"
   m
-------
 test2
(1 row)
```

**Итого мы получили перекрестную подписку между ВМ pgotus и pgotus2**

| на кого подписан | кто подписан  |
|------------------|---------------|
| pgotus.test1     | pgotus2.test2 |
| pgotus2.test1    | pgotus.test2  |

**И получили ВМ pgotus3, которая выполняем роль реплики для выше указанных таблиц**

| на кого подписан | кто подписан  |
|------------------|---------------|
| pgotus.test1     | pgotus3.test1 |
| pgotus2.test1    | pgotus3.test2 |

___

- [x] реализовать горячее реплицирование для высокой доступности на 4ВМ. 
Источником должна выступать ВМ №3. Написать с какими проблемами столкнулись.
___
**Выполняем**
```commandline
pgotus4:~$ sudo systemctl stop postgresql
pgotus4:~$ sudo -u postgres rm -rf /var/lib/postgresql/15/main/*
pgotus4:~$ sudo -u postgres pg_basebackup --host=192.168.100.182 --port=5432 --username=demo --pgdata=/var/lib/postgresql/15/main/ --progress --write-recovery-conf --create-slot --slot=replical
Password:
30477/30477 kB (100%), 1/1 tablespace
pgotus4:~$ sudo -u postgres psql demo -c "select * from test1"
   m
-------
 test1
(1 row)
```
**На данном этапе мы уже видим, что данные стали отображаться, но для полной уверенности попробуем добавить что-то на ВЬ pdotus в таблицу test1**
```commandline
pgotus:~$ sudo -u postgres psql demo -c "INSERT INTO test1 VALUES ('ZZZZZZZZZZZZ')"
INSERT 0 1
pgotus4:~$ sudo -u postgres psql demo -c "select * from test1"
      m
--------------
 test1
 ZZZZZZZZZZZZ
(2 rows)
```
___