- [x] создать ВМ с Ubuntu 20.04/22.04 или развернуть докер любым удобным способом
___
*__Использовал ВМ с Ubuntu 20.04__*
___
- [x] поставить на нем Docker Engine
___
*__Использовал список команд__*
```commandline
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh 
rm get-docker.sh 
sudo usermod -aG docker $USER && newgrp docker
```
___
- [x] сделать каталог /var/lib/postgres
- [x] развернуть контейнер с PostgreSQL 15 смонтировав в него /var/lib/postgresql
___
*__Для развертывания использовал команду__*
```commandline
sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
```
*__При выполнении команды получил сообщение, что локально найти нужный контейнер не удалось и произошло скачивание из библиотеки контейнеров__*
```commandline
Unable to find image 'postgres:15' locally
15: Pulling from library/postgres
```
___
- [x] развернуть контейнер с клиентом postgres
___
*__Использовал команду__*
```commandline
sudo docker run -it --rm --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres
```
___
- [x] подключится из контейнера с клиентом к контейнеру с сервером и сделать
таблицу с парой строк
___
*__Использовал команды из урока и предыдущего задания__*
```commandline
psql -h localhost -U postgres -d postgres
create table persons(id serial, first_name text, second_name text); 
insert into persons(first_name, second_name) values('ivan', 'ivanov'); 
insert into persons(first_name, second_name) values('petr', 'petrov'); 
commit;
```
___
- [x] подключится к контейнеру с сервером с ноутбука/компьютера извне инстансов GCP/ЯО/места установки докера
___
*__Использовал команду из урока__*
```commandline
psql -p 5432 -U postgres -h _IP-адрес_ -d postgres -W
```
___
- [x] удалить контейнер с сервером
___
*__Для удаление использовал команду__*
```commandline
sudo docker stop
sudo docker rm
```
___
- [x] создать его заново
- [x] подключится снова из контейнера с клиентом к контейнеру с сервером
- [x] проверить, что данные остались на месте
___
*__Данные на месте__*
- [x] оставляйте в ЛК ДЗ комментарии что и как вы делали и как боролись с проблемами
___
*__Сложности возникли только с настройкой подключения для удаленного клиента ПГ(но не критичные, нашел нужные настройки даже в лекции)__*