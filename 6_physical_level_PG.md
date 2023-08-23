- [x] создайте виртуальную машину c Ubuntu 20.04/22.04 LTS в GCE/ЯО/Virtual Box/докере
___
*__Использую виртуальную машину__*

___
- [x] поставьте на нее PostgreSQL 15 через sudo apt
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
- [x] проверьте что кластер запущен через sudo -u postgres pg_lsclusters
___
```commandline
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

___
- [x] зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым
___
*__Воспользовался командами__*
```commandline
sudo su postgres
psql
create table test(c1 text);
insert into test values('1');
```
*__Убедился, что таблица создана и вышел из среды__*
```commandline
select * from test;
 c1
----
 1
(1 row)
\q
```

___
- [x] остановите postgres например через sudo -u postgres pg_ctlcluster 15 main stop
- [x] создайте новый диск к ВМ размером 10GB
- [x] добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk
___
*__Так как я использую виртуалку, потребовалось перезагрузить ВМ__*

___
- [x] проинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на актуальное, в вашем случае это скорее всего будет /dev/sdb - https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux
- [x] перезагрузите инстанс и убедитесь, что диск остается примонтированным (если не так смотрим в сторону fstab)
- [x] сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/
___
*__Выполнил команду__*
```commandline
sudo chown -R postgres:postgres /mnt/data/
```

___
- [x] перенесите содержимое /var/lib/postgres/15 в /mnt/data - mv /var/lib/postgresql/15/mnt/data
___
*__Выполнил команду__*
```commandline
sudo mv /var/lib/postgresql/15 /mnt/data
```

___
- [x] попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start
___
*__При попытке выполнить команду получил ошибку__*
```commandline
sudo -u postgres pg_ctlcluster 15 main start
Error: /var/lib/postgresql/15/main is not accessible or does not exist
```

___
- [x] найти конфигурационный параметр в файлах раположенных в /etc/postgresql/15/main который надо поменять и поменяйте его
___
*__в файле `/etc/postgresql/15/main/postgresql.conf` 
заменил значение у параметра `data_directory` на `/mnt/data/15/main`.
Так как перенесли данные в эту папку.__*

___
- [x] попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start
___
*__При попытке выполнить команду ошибки больше нет (только уведомление, что запустили не как сервис)__*
```commandline
sudo -u postgres pg_ctlcluster 15 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
sudo systemctl start postgresql@15-main
Cluster is already running.

```

___
- [x] зайдите через через psql и проверьте содержимое ранее созданной таблицы
___
*__Убедился, что таблица на месте и вышел из среды__*
```commandline
select * from test;
 c1
----
 1
(1 row)
\q
```

___
- [x] задание со звездочкой *: не удаляя существующий инстанс ВМ сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgres, перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.
___
## *__Для выполнения задачи выполнил:__*
- Отключил от первой ВМ диск
- Подключил ко второй ВМ диск
- командой `sudo lsblk` убедился, что диск подключен
```commandline
sudo lsblk
sdb                         8:16   0   10G  0 disk
└─sdb1                      8:17   0   10G  0 part
```
- командой `sudo mkdir -p /mnt/data` создал директорию для дальнейшего монтирования раздела
- командой `sudo mount -o defaults /dev/sdb1 /mnt/data` примонтировал раздел
- установил postgres омандами
```commandline
sudo apt update 
sudo apt upgrade -y 
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' 
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - 
sudo apt-get update 
sudo apt-get -y install postgresql-15
```
- проверил что кластер запущен в базовой директории
```commandline
sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```
- остановил кластер
```commandline
sudo -u postgres pg_ctlcluster 15 main stop
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@15-main
```
- в файле `/etc/postgresql/15/main/postgresql.conf` заменил значение у параметра `data_directory` на `/mnt/data/15/main`
- выдал права пользователя postgres на папку `sudo chown -R postgres:postgres /mnt/data/`
- Запустил кластер
```commandline
sudo -u postgres pg_ctlcluster 15 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@15-main
```
- Убедился что база также перехала на новый сервер
```commandline
sudo su postgres
psql
select * from test;
 c1
----
 1
(1 row)
```
- Порадовался, что получилось

___