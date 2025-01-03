## Тема домашнего задания "Работа с уровнями изоляции транзакции в PostgreSQL"

#### Цель 
Научиться управлять уровнем изоляции транзации в PostgreSQL и понимать особенность работы уровней read commited и repeatable read

### Описание домашнего задания
- создать новый проект в Яндекс облако или на любых ВМ, например postgres2024-, где yyyymmdd год, месяц и день вашего рождения (имя проекта должно быть уникально)
- далее создать инстанс виртуальной машины с дефолтными параметрами - 1-2 ядра, 2-4Гб памяти, любой линукс, на курсе Ubuntu 100%
- добавить свой ssh ключ
- зайти удаленным ssh (первая сессия), не забывайте про ssh-add
- поставить PostgreSQL из пакетов apt install
- зайти вторым ssh (вторая сессия)
- запустить везде psql из под пользователя postgres
- выключить auto commit
- сделать в первой сессии новую таблицу и наполнить ее данными
  - create table persons(id serial, first_name text, second_name text);
  - insert into persons(first_name, second_name) values('ivan', 'ivanov');
  - insert into persons(first_name, second_name) values('petr', 'petrov');
  - commit;
- посмотреть текущий уровень изоляции: show transaction isolation level
- начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции
- в первой сессии добавить новую запись
  - insert into persons(first_name, second_name) values('sergey', 'sergeev');
- сделать select * from persons во второй сессии
- видите ли вы новую запись и если да то почему?
- завершить первую транзакцию - commit;
- сделать select * from persons во второй сессии
- видите ли вы новую запись и если да то почему?
- завершите транзакцию во второй сессии
- начать новые но уже repeatable read транзакции 
  - set transaction isolation level repeatable read;
- в первой сессии добавить новую запись
  - insert into persons(first_name, second_name) values('sveta', 'svetova');
- сделать select * from persons во второй сессии
- видите ли вы новую запись и если да то почему?
- завершить первую транзакцию - commit;
- сделать select * from persons во второй сессии
- видите ли вы новую запись и если да то почему?
- завершить вторую транзакцию
- сделать select * from persons во второй сессии
- видите ли вы новую запись и если да то почему?

### Ход выполнения:

Для удобства добавляем в конец файла /etc/hosts имя сервера для подключения:
```bash
sudo nano /etc/hosts

....
192.168.74.131  debian
```

Генерируем пару ключей

```bash
$ ssh-keygen -t ed25519
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/niker/.ssh/id_ed25519): 
Created directory '/home/niker/.ssh'.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/niker/.ssh/id_ed25519
Your public key has been saved in /home/niker/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:hsMAMdloiD/ci2DpBIiw6SQSZd1g6KGm5yZ3R99H+Y4 niker@fedora
The key's randomart image is:
+--[ED25519 256]--+
|Bo*Boo           |
|*=*+o .          |
|*B.o.            |
|B== .o .         |
|*o o .+ S    .   |
|..o . .o    o    |
| o   . . . . .   |
|. + . . . . ...  |
| + . .     .E..  |
+----[SHA256]-----+
```
Передаем публичный ключ на сервер

```bash
sh-copy-id -i ~/.ssh/id_ed25519.pub debian
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/niker/.ssh/id_ed25519.pub"
The authenticity of host 'debian (192.168.74.131)' can't be established.
ED25519 key fingerprint is SHA256:vBEEP5ComXCUGO8U0tCJcbghLywWi2q1gqO3YRkCseU.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
niker@debian's password: 

Number of key(s) added: 1

Now try logging into the machine, with: "ssh -i /home/niker/.ssh/id_ed25519 'debian'"
and check to make sure that only the key(s) you wanted were added.
```

Проверяем доступ по ключу

```bash
ssh -i ~/.ssh/id_ed25519 debian 
Enter passphrase for key '/home/niker/.ssh/id_ed25519': 
Linux debian12 6.1.0-28-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.119-1 (2024-11-22) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Thu Jan  2 12:44:31 2025 from 192.168.74.133
```

После успешной проверки отключаем доступ по паролю:

```bash
sudo sh -c 'echo "PasswordAuthentication no" >> /etc/ssh/sshd_config'
sudo systemctl restart sshd
```

Устанавливаем PostgreSQL через пакетный менеджер
```bash
sudo apt install postgresql
```

Открываем ещё одну ssh сессию в отдельной вкладке
```bash
ssh debian
```

В обоих ssh сессиях подключаемся как пользователь postgres и запускаем psql
```bash 
sudo su - postgres
psql

l (15.10 (Debian 15.10-0+deb12u1))
Введите "help", чтобы получить справку.

postgres=# 
```

Отключаем автокоммит в обоих сессиях

```pgsql
postgres=# \set AUTOCOMMIT OFF
postgres=# \echo :AUTOCOMMIT
OFF
```

Создадим тестовую БД и подключимся (в двух сессиях) к ней 
```pgsql
postgres=# CREATE DATABASE hw1;
CREATE DATABASE
postgres=# \c hw1
Вы подключены к базе данных "hw1" как пользователь "postgres".
hw1=# 
```

Создадим тестовую таблицу и добавим данные 
```pgsql
hw1=# create table persons(id serial, first_name text, second_name text);
insert into persons(first_name, second_name) values('ivan', 'ivanov');
insert into persons(first_name, second_name) values('petr', 'petrov');
commit;
CREATE TABLE
INSERT 0 1
INSERT 0 1
COMMIT
hw1=#
```

Проверим уровень изоляции
```pgsql
hw1=# show transaction isolation level;
 transaction_isolation 
-----------------------
 read committed
(1 строка)
hw1=*# commit;
COMMIT
hw1=# 
```

В первой сессии добавим новую запись не фиксируя транзакцию
```pgsql
hw1=# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
hw1=*# 
```

Во второй сессии сделаем выборку из таблицы persons
```pgsql
hw1=#  select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 строки)
hw1=*# 
```
Во второй сессии нет данных, добавленных в первой сессии, т.к. при уровне изоляции read committed видны только зафиксированные (commit) изменения.

Завершаем транзакцию в первой сессии (commit;) и проверям выборку во второй сессии
```pgsql
hw1=#  select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 строки)
hw1=*# 
```

После того как мы зафиксировали транзакцию в первой сессии, данные попали в выборку во второй сессии. Уровень изоляции read committed отработал корректно, данные при повторной выборки появились после фиксации

Начнём в сессиях транзакции с уровнем repeatable read.

```pgsql
set transaction isolation level repeatable read;
hw1=# set transaction isolation level repeatable read;
SET
hw1=*# 
```

В первой сессии вставляем данные
```pgsql
hw1=*# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
hw1=*# 
```

Во второй сессии делаем выборку из таблицы
```pgsql
hw1=*#  select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 строки)

hw1=*# 
```
Новые данные не видны, т.к. при уровне транзакции repeatable read, аналогично read commited отсутствует эффект "грязного чтения", до фиксации данные в выборку не попадут.

Завершаем транзакцию в первой сессии и делаем выборку во второй:
```pgsql
hw1=*#  select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 строки)

hw1=*# 
```
После фиксации данные не попали в выборку, т.к. при уровне транзакции repeatable read гарантируется отсутствие эффектов "Неповторяемое чтение" и "Фантомное чтение", т.е. исключает возможность чтения изменений в рамках одной транзакции.

После завершения транзакции во второй сессии при повторной выборке новые данные есть. Уровень транзакции repeatable read отработал корректно.

```pgsql
hw1=*# commit;
COMMIT
hw1=#  select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 строки)

hw1=*# 
```


