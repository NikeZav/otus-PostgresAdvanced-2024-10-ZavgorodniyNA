## Тема домашнего задания "Установка и настройка PostgteSQL в контейнере Docker"

### Цель:
- развернуть ВМ ЯО/Аналоги
- установить туда докер
- установить PostgreSQL в Docker контейнере
- настроить контейнер для внешнего подключения

### Описание домашнего задания:

- сделать в ЯО/Аналоги инстанс с Ubuntu 20.04
- поставить на нем Docker Engine
- сделать каталог /var/lib/postgres
- развернуть контейнер с PostgreSQL 14 смонтировав в него /var/lib/postgres
- развернуть контейнер с клиентом postgres
- подключится из контейнера с клиентом к контейнеру с сервером и сделать таблицу с парой строк
- подключится к контейнеру с сервером с ноутбука/компьютера извне инстансов GCP/ЯО/Аналоги
- удалить контейнер с сервером
- создать его заново
- подключится снова из контейнера с клиентом к контейнеру с сервером проверить, что данные остались на месте

### Ход выполнения:

Использую готовую виртуальную машину с Debian 12 в VMWare Workstation, которую развернул для домашнего задания №1

Установим Docker Engine
```bash
sudo apt install docker

niker@debian12:~$ docker --version
Docker version 20.10.24+dfsg1, build 297e128
```
Для удобства добавим текущего пользователя в группу docker (для применения нужно будет переподключиться)
```bash
sudo usermod -aG docker niker
```
Создадим каталог /var/lib/postgres
```bash
sudo mkdir /var/lib/postgres
```
Скачаем образы Postgresql 14 и клиента и проверим, что образы скачались
```bash
docker pull postgres:14
docker pull alpine/psql

niker@debian12:~$ docker image ls
REPOSITORY    TAG         IMAGE ID       CREATED         SIZE
alpine/psql   latest      586923559407   6 days ago      12.7MB
postgres      14          365d502a72f3   6 weeks ago     422MB
```

Для сетевого взаимодействия между контейнерами создадим сеть и проверим, что она создалась
```bash
docker network create pg_net

niker@debian12:~$ docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
e6f928c8a014   bridge    bridge    local
a67087e5fdf2   host      host      local
dbaee1025be3   none      null      local
2ba2be8e7ee8   pg_net    bridge    local
```

Запустим контейнер c postgresql

```bash
docker run -e POSTGRES_PASSWORD=postgres -v /var/lib/postgres:/var/lib/postgresql/data -d --network pg_net -p 5433:5432 --name pg_server --rm 365d502a72f3

niker@debian12:~$ docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                                       NAMES
037e0cdc4981   365d502a72f3   "docker-entrypoint.s…"   6 seconds ago   Up 5 seconds   0.0.0.0:5433->5432/tcp, :::5433->5432/tcp   pg_server


```
Параметры запуска:
-e - задать переменную окружения, в данном случае обязательное требование для запуска контейнера впервые.
-v - монтирование внешней папки
-d - запуск в фоновом режиме
-p - соответствие внешнего:внутреннего порта
--network - имя сети в которой будет работать контейнер
--name - имя контейнера
--rm - удалить контейнер после остановки

Запустим контейнер с клиентом и подключимся к контейнеру с postgresql
```bash
niker@debian12:~$ docker run -ti --network pg_net --rm alpine/psql -h pg_server -U postgres -d postgres
Password for user postgres: 
psql (17.2, server 14.15 (Debian 14.15-1.pgdg120+1))
Type "help" for help.

postgres=# 
```

Создадим таблицу и добавим пару записей, проверим:
```sql
postgres=# create table persons(id serial, first_name text, second_name text); 
insert into persons(first_name, second_name) values('ivan', 'ivanov'); 
insert into persons(first_name, second_name) values('petr', 'petrov');
CREATE TABLE
INSERT 0 1
INSERT 0 1
postgres=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

postgres=# 
```
Выйдем из клиента

Теперь попробуем подключиться с текущей ВМ к pg_server. В данном образе доступ для внешних подключений уже разрешён, но если бы его не было, то можно было отредактировать файл /var/lib/postgres/pg_hba.conf и перезапустить контейнер docker restart pg_server, либо подключиться внутрь контейнера с помощью docker exec -it pg_server bash и уже внутри контейнера отредактировать файл pg_hba.conf

```bash
niker@debian12:~$ psql -h localhost -U postgres -p 5433
Пароль пользователя postgres: 
psql (15.10 (Debian 15.10-0+deb12u1), сервер 14.15 (Debian 14.15-1.pgdg120+1))
Введите "help", чтобы получить справку.

postgres=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 строки)

postgres=# 
```

Теперь остановим контейнер и т.к. был ключ --rm при запуске, проверим, что контейнер удалился
```bash
docker stop pg_server

niker@debian12:~$ docker ps -a
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
niker@debian12:~$ 
```
Ещё раз запустим, в этот раз без ключа *-e*, видим, что ID контейнера другой. 
```bash
niker@debian12:~$ docker run -v /var/lib/postgres:/var/lib/postgresql/data -d --network pg_net -p 5433:5432 --name pg_server --rm 365d502a72f3
20296ca66f43d6894fa9e7e2b7037ab9676c86726dbe6fa637292414b2e92362
niker@debian12:~$ docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS          PORTS                                       NAMES
20296ca66f43   365d502a72f3   "docker-entrypoint.s…"   14 seconds ago   Up 13 seconds   0.0.0.0:5433->5432/tcp, :::5433->5432/tcp   pg_server
niker@debian12:~$ 
```

Подключимся к контейнеру клиентом и посмотрим данные
```bash
niker@debian12:~$ psql -h localhost -U postgres -p 5433
Пароль пользователя postgres: 
psql (15.10 (Debian 15.10-0+deb12u1), сервер 14.15 (Debian 14.15-1.pgdg120+1))
Введите "help", чтобы получить справку.

postgres=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 строки)

postgres=# 
```

Наши данные на месте, значит данные были сохранены в нашем docker volume


