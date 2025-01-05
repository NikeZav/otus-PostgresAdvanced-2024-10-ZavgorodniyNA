## Тема домашнего задания "Настройка дисков для Постгреса"

### Цель:
- создавать дополнительный диск для уже существующей виртуальной машины, размечать его и делать на нем файловую систему
- переносить содержимое базы данных PostgreSQL на дополнительный диск
- переносить содержимое БД PostgreSQL между виртуальными машинами

### Описание домашнего задания:

- создайте виртуальную машину c Ubuntu 20.04 LTS (bionic) в GCE типа e2-medium в default VPC в любом регионе и зоне, например us-central1-a или ЯО/VirtualBox
- поставьте на нее PostgreSQL 15 через sudo apt
- проверьте что кластер запущен через sudo -u postgres pg_lsclusters
- зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым
- остановите postgres например через sudo -u postgres pg_ctlcluster 15 main stop
- создайте новый standard persistent диск GKE через Compute Engine -> Disks в том же регионе и зоне что GCE инстанс размером например 10GB  или аналог в другом облаке/виртуализации
- добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk
- проинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на актуальное, в вашем случае это скорее всего будет /dev/sdb - https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux
- перезагрузите инстанс и убедитесь, что диск остается примонтированным (если не так смотрим в сторону fstab)
- сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/
- перенесите содержимое /var/lib/postgresql/15 в /mnt/data - mv /var/lib/postgresql/15 /mnt/data
- попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start
- напишите получилось или нет и почему
- задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/14/main который надо поменять и поменяйте его
- напишите что и почему поменяли
- попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start
- напишите получилось или нет и почему
- зайдите через через psql и проверьте содержимое ранее созданной таблицы
- задание со звездочкой *: не удаляя существующий GCE инстанс/ЯО сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgresql, перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.

### Ход выполнения:

Использую готовую виртуальную машину с Debian 12 в VMWare Workstation, которую разворачивал для предыдущих ДЗ

На ВМ установлен PostgreSQL 15 из пакетов (sudo apt install postgresql) 

Проверяем, что кластер запущен
```bash
niker@debian12:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory               Log file
15  main    5432 online postgres /var/lib/postgresql/15/main  /var/log/postgresql/postgresql-15-main.log
```

Подключаемся к базе и создаём тестовую таблицу
```bash
sudo -u postgres psql

postgres=# create table test (c1 text);
CREATE TABLE
postgres=# insert into test values ('1');
INSERT 0 1
postgres=# \dt
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 public | test | table | postgres
(1 row)

postgres=# select * from test ;
 c1 
----
 1
(1 row)

postgres=# 
```

Добавляю диск 10Gb (SCSI) в настройках ВМ VmWare Workstation

Проверяем именование нужного диска. В нашем случае - sdb
```bash
niker@debian12:~$ lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                       8:0    0   20G  0 disk 
├─sda1                    8:1    0  487M  0 part /boot
├─sda2                    8:2    0    1K  0 part 
└─sda5                    8:5    0 19,5G  0 part 
  ├─debian12--vg-root   254:0    0 18,6G  0 lvm  /
  └─debian12--vg-swap_1 254:1    0  980M  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk 
sr0                      11:0    1 1024M  0 rom  
```

Создаём раздел и форматируем новый диск
```bash
sudo parted /dev/sdb mklabel gpt
sudo parted -a optimal -s /dev/sdb mkpart primary ext4  0% 100%
sudo mkfs.ext4 /dev/sdb1
```

Проверяем
```bash
niker@debian12:~$ sudo parted /dev/sdb print
Model: VMware, VMware Virtual S (scsi)
Disk /dev/sdb: 10,7GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name     Flags
 1      1049kB  10,7GB  10,7GB  ext4         primary
```

Создаём каталог монтирования, узнаём UUID правим fstab и перезагружаемся для проверки автомонтирования
```bash
sudo mkdir /mnt/data

niker@debian12:~$ sudo blkid /dev/sdb1
/dev/sdb1: UUID="875ffa37-1c2b-403f-b021-6b755aab9397" BLOCK_SIZE="4096" TYPE="ext4" PARTLABEL="primary" PARTUUID="d55f09e1-14c8-43ba-8aba-6d4bf6a9703c"

sudo nano /etc/fstab

---
UUID=875ffa37-1c2b-403f-b021-6b755aab9397       /mnt/data       ext4    defaults        0       2
---
```
После перезагрузки проверяем, что диск примонтировался
```bash
niker@debian12:~$ df -hT
Файловая система              Тип      Размер Использовано  Дост Использовано% Cмонтировано в
udev                          devtmpfs   943M            0  943M            0% /dev
tmpfs                         tmpfs      194M         796K  193M            1% /run
/dev/mapper/debian12--vg-root ext4        19G         4,0G   14G           24% /
tmpfs                         tmpfs      967M         2,1M  965M            1% /dev/shm
tmpfs                         tmpfs      5,0M            0  5,0M            0% /run/lock
/dev/sda1                     ext2       455M         201M  230M           47% /boot
/dev/sdb1                     ext4       9,8G          24K  9,3G            1% /mnt/data
tmpfs                         tmpfs      194M            0  194M            0% /run/user/1000
```

Останавливаем инстанс *main*
```bash
sudo systemctl stop postgresql@15-main
```
Делаем пользователя postgres владельцем /mnt/data и переносим содержимое /var/lib/postgresql/15 в /mnt/data
```bash
sudo chown -R postgres:postgres /mnt/data/
sudo mv /var/lib/postgresql/15 /mnt/data
```

Пробуем запустить кластер - возникает ошибка, т.к. каталога с данными нет, мы его перенесли в другую папку
```bash
niker@debian12:~$ sudo -u postgres pg_ctlcluster 15 main start
Error: /var/lib/postgresql/15/main is not accessible or does not exist
```
Чтобы это исправить нужно в файле postgresql.conf поправить параметр data_directory на новый путь
```bash
niker@debian12:~$ sudo nano /etc/postgresql/15/main/postgresql.conf 
niker@debian12:~$ sudo -u postgres pg_ctlcluster 15 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@15-main
Removed stale pid file.
niker@debian12:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory    Log file
15  main    5432 online postgres /mnt/data/15/main /var/log/postgresql/postgresql-15-main.log
niker@debian12:~$ 
```
Кластер запустился, подключаемся и проверяем, что наша таблица есть.
```bash
niker@debian12:~$ sudo -u postgres psql
could not change directory to "/home/niker": Отказано в доступе
psql (15.10 (Debian 15.10-0+deb12u1))
Type "help" for help.

postgres=# \dt
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 public | test | table | postgres
(1 row)

postgres=# select * from test;
 c1 
----
 1
(1 row)

postgres=# 
```
