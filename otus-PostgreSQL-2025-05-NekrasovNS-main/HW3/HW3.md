# Домашняя работа № 3

**1. Создал виртуальную машину на базе ПО VirtualBox, установил ОС Linux Ubuntu 22.04 с дефолтными настройками системы, установил СУБД PostgresSQL 16 на свежеустановленую ОС.**

Выполнил команду проверки работоспособности кластера: ```sudo -u postgres pg_lsclusters```

```
root@pgdbvip:~# sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log```
```

**2. Создал произвольную таблицу с произвольным содержимым и останавливаем кластер postgres:**

```
postgres=# create table otus(c1 text);
CREATE TABLE
postgres=# insert into otus values ('20');
INSERT 0 1
```

Останавливаем работу кластера:

```
root@pgdbvip:~# systemctl stop postgresql@16-main.service
root@pgdbvip:~# sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 down   postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```

**3. Создаю и добавляю дополнительный диск к виртуальной машине в размере 15ГБ, так же инициализирую его в системе.**


Средствами ПО VirtualBox создаю диск на 15ГБ и примонтирую его к созданной ВМ.

Монтируем средствами fdisk раздел /dev/sdb:

```
sda                         8:0    0   15G  0 disk
├─sda1                      8:1    0    1M  0 part
├─sda2                      8:2    0  1,8G  0 part /boot
└─sda3                      8:3    0 13,2G  0 part
  └─ubuntu--vg-ubuntu--lv 253:0    0   10G  0 lvm  /
sdb                         8:16   0   15G  0 disk
└─sdb1                      8:17   0   15G  0 part
```

Создем файловую систему формата ext4 на диске, путем его форматирования:

```
root@pgdbvip:~# mkfs.ext4 /dev/sdb1
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 3931904 4k blocks and 983040 inodes
Filesystem UUID: 5102d8c9-95aa-47e9-bef5-152c531242c6
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done
```
Создаем директорию /backup
```
root@pgdbvip:~# mkdir /backup
```

Монтируем диск в директорию /backup командой ```mount -t ext4 /dev/sdb1 /backup```

```
root@pgdbvip:~# mount -t ext4 /dev/sdb1 /backup

```
Добавляем UUID диска в /etc/fstab(ввести UUID), чтобы после перезагрузки VM диск был примонтирован.


**4. Перенес каталог postgres в примонтированный раздел на отдельном диске командой ```mv /var/lib/postgresql/16 /backup```:**


Попытался запустить кластер, вышла ошибка:

```
root@pgdbvip:/backup/16/main# systemctl start postgresql@16-main.service
Job for postgresql@16-main.service failed because the service did not take the steps required by its unit configuration.
See "systemctl status postgresql@16-main.service" and "journalctl -xeu postgresql@16-main.service" for details.
```
**В конфигурационном файле /etc/postgresql/16/main/postgresql.conf нужно изменить параметр ```data_directory='/backup/16/main'```**
**Меняем права на директорию и владельца:**
```root@pgdbvip:/backup/16/main# sudo chown -R postgres:postgres /backup/16```
```root@pgdbvip:/backup/16/main# sudo chmod -R 700 /backup/16```

```sudo systemctl start postgresql@16-main```


```
root@pgdbvip:/backup/16/main# systemctl status postgresql@16-main.service
● postgresql@16-main.service - PostgreSQL Cluster 16-main
     Loaded: loaded (/lib/systemd/system/postgresql@.service; enabled-runtime; vendor preset: enabled)
     Active: active (running) since Mon 2025-11-03 19:03:55 UTC; 3s ago
    Process: 3285 ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 16-main start (code=exited, status=0/SUCCESS)
   Main PID: 3290 (postgres)
      Tasks: 6 (limit: 2220)
     Memory: 22.9M
        CPU: 111ms
     CGroup: /system.slice/system-postgresql.slice/postgresql@16-main.service
             ├─3290 /usr/lib/postgresql/16/bin/postgres -D /backup/16/main -c config_file=/etc/postgresql/16/main/postgresql.conf
             ├─3291 "postgres: 16/main: checkpointer " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ""
             ├─3292 "postgres: 16/main: background writer " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ""
             ├─3294 "postgres: 16/main: walwriter " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "">
             ├─3295 "postgres: 16/main: autovacuum launcher " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ""
             └─3296 "postgres: 16/main: logical replication launcher
```

**5. Через консоль psql проверяем содержимое ранее созданной таблицы otus:**

Делаем запрос: 

```
postgres=# select * from otus;
 c1
----
 20
(1 строка)

```

Видим данные остались, аномалий нет.