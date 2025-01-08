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
for i in {1..3}; do vm=patroni$i && ssh $vm 'sudo apt -y install postgresql' & done;

for i in {1..3}; do vm=patroni$i && ssh $vm 'hostname; pg_lsclusters' & done;
```



