## Тема домашнего задания "Бэкапы Постгреса"

### Цель:
Используем современные решения для бэкапов

### Описание домашнего задания:

Делам бэкап Постгреса используя WAL-G или pg_probackup и восстанавливаемся на другом кластере

### Ход выполнения:

Использую готовую виртуальную машину с Debian 12 в VMWare Workstation, которую развернул для домашнего задания №1

На ВМ созданы два инстанса PostgreSQL **main** и **main2**. 

Скачаем и установим последнюю стабильную версию WAL-G

```bash
curl -L "https://github.com/wal-g/wal-g/releases/download/v3.0.3/wal-g-pg-ubuntu-20.04-amd64" -o "wal-g"
chmod ugo+x wal-g
sudo mv wal-g /usr/local/bin/

niker@debian12:~$ wal-g --version
wal-g version v3.0.3    3f88f3c 2024.08.08_17:53:40     PostgreSQL
```

Создадим папку для бэкапов и сделаем владельцем пользователя postgres
```bash
sudo mkdir /backup /backup/wal-g
chown -R postgres /backup/
```

Дальнейшие действия делаем из под пользователя *postgres*. 
Создадим файл конфигурации по-умолчанию в домашней папке пользователя

```bash
sudo su - postgres
nano .walg.json
```

```json
{
    "WALG_FILE_PREFIX": "/backup/wal-g",
    "WALG_COMPRESSION_METHOD": "brotli",
    "WALG_DELTA_MAX_STEPS": "5",
    "PGDATA": "/var/lib/postgresql/15/main",
    "PGHOST": "/var/run/postgresql/.s.PGSQL.5432"
}
```

Добавим в файл postgresql.auto.conf инстанса main следующие строки и перезапустим инстанс для применения настроек:

```bash
wal_level=replica
archive_mode=on
archive_command='/usr/local/bin/wal-g wal-push \"%p\" >> /var/log/postgresql/archive.log 2>&1'
archive_timeout=60
restore_command='/usr/local/bin/wal-g wal-fetch \"%f\" \"%p\" >> /var/log/postgresql/restore.log 2>&1'
```


Выполним бэкап инстанса, в папке с бэкапами появился первый бэкап
```bash
wal-g backup-push /var/lib/postgresql/15/main

postgres@debian12:~$ wal-g backup-list
INFO: 2025/01/04 16:05:10.018211 List backups from storages: [default]
backup_name                                              modified                  wal_file_name            storage_name
base_000000010000000000000002                            2025-01-04T14:56:11+03:00 000000010000000000000002 default
```
Создадим в main тестовую БД и таблицу, заполним её тестовыми данными:

```sql
CREATE DATABASE testdb;
\c testdb
CREATE TABLE test(i int);
insert into test(i) select * from generate_series(1, 10);

testdb=# select * from test;
 i  
----
  1
  2
  3
  4
  5
  6
  7
  8
  9
 10
(10 строк)
```

Сделаем бэкап ещё раз, у нас появилась Delta копия:
```bash
wal-g backup-push /var/lib/postgresql/15/main

postgres@debian12:~$ wal-g backup-list
INFO: 2025/01/04 16:05:10.018211 List backups from storages: [default]
backup_name                                              modified                  wal_file_name            storage_name
base_000000010000000000000002                            2025-01-04T14:56:11+03:00 000000010000000000000002 default
base_000000010000000000000004_D_000000010000000000000002 2025-01-04T15:59:24+03:00 000000010000000000000004 default
```

Восстановление 

Остановим **main2** и удалим файлы из папки с данными

```bash
pg_ctlcluster 15 main2 stop
rm -r /var/lib/postgresql/15/main2
```

Восстановим последний бэкап и запустим инстанс:

```bash
wal-g backup-fetch /var/lib/postgresql/15/main2 LATEST
touch /var/lib/postgresql/15/main2/recovery.signal

postgres@debian12:~$ pg_ctlcluster 15 main2 start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@15-main2
postgres@debian12:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory               Log file
15  main    5432 online postgres /var/lib/postgresql/15/main  /var/log/postgresql/postgresql-15-main.log
15  main2   5433 online postgres /var/lib/postgresql/15/main2 /var/log/postgresql/postgresql-15-main2.log
```

Подключаемся ко второму инстансу и проверяем, что данные у нас есть.

```sql
postgres@debian12:~$ psql -p 5433
psql (15.10 (Debian 15.10-0+deb12u1))
Введите "help", чтобы получить справку.

postgres=# \l
                                                  Список баз данных
    Имя    | Владелец | Кодировка | LC_COLLATE  |  LC_CTYPE   | локаль ICU | Провайдер локали |     Права доступа     
-----------+----------+-----------+-------------+-------------+------------+------------------+-----------------------
 hw1       | postgres | UTF8      | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc             | 
 postgres  | postgres | UTF8      | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc             | 
 template0 | postgres | UTF8      | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc             | =c/postgres          +
           |          |           |             |             |            |                  | postgres=CTc/postgres
 template1 | postgres | UTF8      | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc             | =c/postgres          +
           |          |           |             |             |            |                  | postgres=CTc/postgres
 testdb    | postgres | UTF8      | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc             | 
(5 строк)

postgres=# \c testdb
Вы подключены к базе данных "testdb" как пользователь "postgres".
testdb=# select * from test;
 i  
----
  1
  2
  3
  4
  5
  6
  7
  8
  9
 10
(10 строк)

testdb=# 
```

Бэкап восстановлен, данные есть. 