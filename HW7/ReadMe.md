## Тема домашнего задания "Кластер Patroni"

### Цель:
Развернуть HA кластер

### Описание домашнего задания:

- Создаем 3 ВМ для etcd + 3 ВМ для Patroni +1 HA proxy (при проблемах можно на 1 хосте развернуть)
- Инициализируем кластер
- Проверяем отказоустойсивость
- *настраиваем бэкапы через wal-g или pg_probackup

### Ход выполнения:

Создадим следующие ВМ на базе Debian 12
| IP             | Name     |
| -------------- | -------- |
| 192.168.74.142 | etcd1    |
| 192.168.74.148 | etcd2    |
| 192.168.74.147 | etcd3    |
| 192.168.74.146 | patroni1 |
| 192.168.74.145 | patroni2 |
| 192.168.74.144 | patroni3 |
| 192.168.74.143 | haproxy1 |

Добавим в файл /etc/hosts сразу имена серверов
```bash
sudo nano /etc/hosts

...
192.168.74.142 etcd1
192.168.74.148 etcd2
192.168.74.147 etcd3
192.168.74.146 patroni1
192.168.74.145 patroni2
192.168.74.144 patroni3
192.168.74.143 haproxy1 
```

### Установка и настройка ETCD

Устанавливаем etcd на сервера etcd*
```bash
sudo apt install etcd-server etcd-client

niker@etcd1:~$ etcd --version
etcd Version: 3.4.23
Git SHA: Not provided (use ./build instead of go build)
Go Version: go1.19.8
Go OS/Arch: linux/amd64
```

Настраиваем конфигурацию
```bash
sudo nano /etc/defaults/etcd

## Use environment to override, for example: ETCD_NAME=default
ETCD_NAME="etcd1"  #меняем на каждом сервере
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://etcd1:2379"  #меняем на каждом сервере
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://etcd1:2380"  #меняем на каждом сервере
ETCD_INITIAL_CLUSTER_TOKEN="etcd_cluster"
ETCD_INITIAL_CLUSTER="etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_ENABLE_V2="true"

sudo systemctl start etcd
```

После успешного старта проверяем, что кластер etcd успешно запустился

```bash
niker@etcd1:~$ etcdctl endpoint status --cluster -w table
+-------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|     ENDPOINT      |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+-------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| http://etcd1:2379 |  b527e3f018dd770 |  3.4.23 |   20 kB |     false |      false |       720 |         19 |                 19 |        |
| http://etcd3:2379 | cef988610ab31962 |  3.4.23 |   20 kB |      true |      false |       720 |         19 |                 19 |        |
| http://etcd2:2379 | ff6ccee9c34eff04 |  3.4.23 |   37 kB |     false |      false |       720 |         19 |                 19 |        |
+-------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

Далее на etcd* меняем в /etc/defaults/etcd параметр и перезапускаем etcd
```bash
ETCD_INITIAL_CLUSTER_STATE="existing"

sudo systemctl restart etcd
```
### Установка и настройка PostgreSQL и Patroni

Устанавливаем на серверах PostgreSQL и проверяем, что PostgreSQL запустился
```bash
for i in {1..3}; do vm=patroni$i && ssh $vm 'sudo apt -y install postgresql patroni rsync' & done;

for i in {1..3}; do vm=patroni$i && ssh $vm 'hostname; pg_lsclusters' & done;
```

Создаём файл конфигурации для Patroni на одном из хостов и заполняем его
```
sudo nano /etc/patroni/config.yml

scope: patroni
name: patroni3
restapi:
  listen: 192.168.74.144:8008
  connect_address: 192.168.74.144:8008
etcd:
  hosts: etcd1:2379,etcd2:2379,etcd3:2379
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      parameters:
  initdb:
  - encoding: UTF8
  - data-checksums
  pg_hba:
  - host replication replicator 192.0.0.0/8 md5
  - host all all 192.0.0.0/8 md5
  users:
    admin:
      password: postgres
      options:
        - createrole
        - createdb
postgresql:
  listen: 127.0.0.1, 192.168.74.144:5432
  connect_address: 192.168.74.144:5432
  data_dir: /var/lib/postgresql/15/main
  config_dir: /var/lib/postgresql/15/main
  bin_dir: /usr/lib/postgresql/15/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: rep-pass_321
    superuser:
      username: admin
      password: postgres
    rewind:
      username: rewind_user
      password: rewind
  parameters:
    unix_socket_directories: '/tmp/'
tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```
Добавляем переменную окружения на серверах patroni, чтобы не дописывать путь к конфигурационному файлу (-c /etc/patroni/config.yml)
export PATRONICTL_CONFIG_FILE=/etc/patroni/config.yml
сразу добавляем переменную для всех пользователей создав файл, где прописываем её в файл и копируем на остальные хосты с patroni, так же копируем файл конфигурации

```
sudo nano /etc/profile.d/alluser_env.sh
sudo rsync --rsync-path="sudo rsync" /etc/profile.d/alluser_env.sh niker@patroni1:/etc/profile.d/
sudo rsync --rsync-path="sudo rsync" /etc/profile.d/alluser_env.sh niker@patroni2:/etc/profile.d/
sudo rsync --rsync-path="sudo rsync" /etc/patroni/config.yml niker@patroni1:/etc/patroni/
sudo rsync --rsync-path="sudo rsync" /etc/patroni/config.yml niker@patroni2:/etc/patroni/
```

На всех хостах останавливаем службы postgresql и patroni и очищаем папку /var/lib/postgresql/15/main/
```bash
for i in {1..3}; do vm=patroni$i && ssh $vm 'sudo systemctl stop postgresql; sudo systemctl stop patroni' & done;
for i in {1..3}; do vm=patroni$i && ssh $vm 'sudo rm -R /var/lib/postgresql/15/main/*' & done;
```

Запускаем patroni на patroni3. При запуске происходит первоначальная инициализация кластера, проверяем:
```
postgres@patroni3:~$ patronictl list
+ Cluster: patroni ---------+--------+---------+----+-----------+
| Member   | Host           | Role   | State   | TL | Lag in MB |
+----------+----------------+--------+---------+----+-----------+
| patroni3 | 192.168.74.144 | Leader | running |  3 |           |
+----------+----------------+--------+---------+----+-----------+
```
Запускаем Patroni на остальных узлах. Проверяем состояние кластера
```
niker@patroni3:~$ patronictl list
+ Cluster: patroni ---------+---------+---------+----+-----------+
| Member   | Host           | Role    | State   | TL | Lag in MB |
+----------+----------------+---------+---------+----+-----------+
| patroni1 | 192.168.74.146 | Replica | running |  5 |         0 |
| patroni2 | 192.168.74.145 | Replica | running |  5 |         0 |
| patroni3 | 192.168.74.144 | Leader  | running |  5 |           |
+----------+----------------+---------+---------+----+-----------+
```

Пробуем штатное переключение 

```
niker@patroni3:~$ patronictl switchover
Current cluster topology
+ Cluster: patroni ---------+---------+---------+----+-----------+
| Member   | Host           | Role    | State   | TL | Lag in MB |
+----------+----------------+---------+---------+----+-----------+
| patroni1 | 192.168.74.146 | Replica | running |  5 |         0 |
| patroni2 | 192.168.74.145 | Replica | running |  5 |         0 |
| patroni3 | 192.168.74.144 | Leader  | running |  5 |           |
+----------+----------------+---------+---------+----+-----------+
Primary [patroni3]: 
Candidate ['patroni1', 'patroni2'] []: patroni1
When should the switchover take place (e.g. 2025-02-09T20:20 )  [now]: 
Are you sure you want to switchover cluster patroni, demoting current leader patroni3? [y/N]: y
2025-02-09 19:20:15.30606 Successfully switched over to "patroni1"

+ Cluster: patroni ---------+---------+---------+----+-----------+
| Member   | Host           | Role    | State   | TL | Lag in MB |
+----------+----------------+---------+---------+----+-----------+
| patroni1 | 192.168.74.146 | Leader  | running |  6 |           |
| patroni2 | 192.168.74.145 | Replica | running |  6 |         0 |
| patroni3 | 192.168.74.144 | Replica | running |  6 |         0 |
+----------+----------------+---------+---------+----+-----------+
```

Останавливаем patroni на лидере

```
niker@patroni3:~$ patronictl list
+ Cluster: patroni ---------+---------+---------+----+-----------+
| Member   | Host           | Role    | State   | TL | Lag in MB |
+----------+----------------+---------+---------+----+-----------+
| patroni1 | 192.168.74.146 | Replica | stopped |    |   unknown |
| patroni2 | 192.168.74.145 | Replica | running |  7 |         0 |
| patroni3 | 192.168.74.144 | Leader  | running |  7 |           |
+----------+----------------+---------+---------+----+-----------+
```

Запускаем
```
niker@patroni3:~$ patronictl list
+ Cluster: patroni ---------+---------+---------+----+-----------+
| Member   | Host           | Role    | State   | TL | Lag in MB |
+----------+----------------+---------+---------+----+-----------+
| patroni1 | 192.168.74.146 | Replica | running |  7 |         0 |
| patroni2 | 192.168.74.145 | Replica | running |  7 |         0 |
| patroni3 | 192.168.74.144 | Leader  | running |  7 |           |
+----------+----------------+---------+---------+----+-----------+
```

### Настраиваем HAProxy
Подключаемся к haproxy1 и запускаем установку
```
sudo apt -y install haproxy
```

Сделаем бэкап оригинального файла конфигурации и сделаем свою конфигурацию
```
sudo cp -p /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg-orig
sudo nano /etc/haproxy/haproxy.cfg

global
    maxconn 100
 
defaults
    log global
    mode tcp
    retries 2
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s
 
listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /
 
listen postgres
    bind *:5000
    option httpchk OPTIONS /master
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server patroni1 patroni1:5432 maxconn 100 check port 8008
    server patroni2 patroni2:5432 maxconn 100 check port 8008
    server patroni3 patroni3:5432 maxconn 100 check port 8008

```
Проверим файл конфигурации и перезапустим службу haproxy
```
niker@haproxy1:~$ sudo haproxy -c -V -f /etc/haproxy/haproxy.cfg
Configuration file is valid

sudo systemctl restart haproxy
```

Проверяем подключение через HAProxy:
```
niker@patroni3:~$ psql -h haproxy1 -p 5000 -U admin -d postgres
Пароль пользователя admin: 
psql (15.10 (Debian 15.10-0+deb12u1))
Введите "help", чтобы получить справку.

postgres=# \l
                                               Список баз данных
    Имя    | Владелец | Кодировка | LC_COLLATE  |  LC_CTYPE   | локаль ICU | Провайдер локали |  Права доступа  
-----------+----------+-----------+-------------+-------------+------------+------------------+-----------------
 postgres  | admin    | UTF8      | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc             | 
 template0 | admin    | UTF8      | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc             | =c/admin       +
           |          |           |             |             |            |                  | admin=CTc/admin
 template1 | admin    | UTF8      | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc             | =c/admin       +
           |          |           |             |             |            |                  | admin=CTc/admin
(3 строки)

postgres=# 

```
