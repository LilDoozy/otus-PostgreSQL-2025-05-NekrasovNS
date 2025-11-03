# Домашняя работа № 3

**Шаг 1. Создал виртуальную машину на базе ПО VirtualBox, установил ОС Linux Ubuntu 22.04 с дефолтными настройками системы, установил СУБД PostgresSQL 16 на свежеустановленую ОС.**

Выполнил команду проверки работоспособности кластера: ```sudo -u postgres pg_lsclusters```

```
student@studentPC:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```

**Шаг 2. Создал произвольную таблицу с произвольным содержимым и останавливаем кластер postgres:**

```
postgres=# create table test(c1 text);
CREATE TABLE
postgres=# insert into test values('1');
INSERT 0 1
```

Останавливаем работу кластера:

```
student@studentPC:~$ sudo systemctl stop postgresql@16-main
student@studentPC:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 down   postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```

**Шаг 3. Создаю и добавляю дополнительный диск к виртуальной машине в размере 10ГБ, так же инициализирую его в системе.**


Средствами ПО VirtualBox создаю дис на 10ГБ и примонтирую его к созданной ВМ.

Монтируем средствами fdisk раздел /dev/sdb:

```
sda      8:0    0    50G  0 disk 
├─sda1   8:1    0     1M  0 part 
└─sda2   8:2    0    50G  0 part /
sdb      8:16   0    10G  0 disk 
└─sdb1   8:17   0    10G  0 part
```

Создем файловую систему формата ext4 на диске, путем его форматирования:

```
root@studentPC:~# mkfs.ext4 /dev/sdb1
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 2621184 4k blocks and 655360 inodes
UUID файловой системы: 6e64c7b6-abcf-44b4-9bbe-1414a6e20520
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Распределение групповых таблиц: готово                            
Сохранение таблицы inod'ов: готово                            
Создание журнала (16384 блоков): готово
Writing superblocks and filesystem accounting information: готово
```

Проверяем смонтирован ли диск в папку /backup созданую ранее, командой ```mount -t ext4 /dev/sdb1 /backup```

```
root@studentPC:~# mount -t ext4 /dev/sdb1 /backup
mount: /backup: /dev/sdb1 уже смонтирован в /backup.
       dmesg(1) may have more information after failed mount system call.
```

После перезагрузки VM диск остался примонтированным (Нужно было добавить UUID диска в файл /etc/fstab)

**Шаг 4. Перенес каталог postgres в примонтированный раздел на отдельном диске командой ```mv /var/lib/postgresql/16 /backup```:**


Попытался запустить кластер, вышла ошибка:

```
root@studentPC:/backup# sudo systemctl start postgresql@16-main
Job for postgresql@16-main.service failed because the service did not take the steps required by its unit configuration.
See "systemctl status postgresql@16-main.service" and "journalctl -xeu postgresql@16-main.service" for details.
```
**В конфигурационном файле postgresql.conf нужно изменить параметр ```data_directory='Путь к новой директории монитрования, в нашем случае: /backup/16/main'```**

Пробуем стартовать службу: ```sudo systemctl start postgresql@16-main```

Ошибки нет, смотрим статус службы Postgres: Успешно запущена

```
student@studentPC:~$ sudo systemctl status postgresql@16-main
● postgresql@16-main.service - PostgreSQL Cluster 16-main
     Loaded: loaded (/usr/lib/systemd/system/postgresql@.service; enabled-runtime; preset: enabled)
     Active: active (running) since Sat 2025-04-19 13:50:57 MSK; 29min ago
    Process: 4402 ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 16-main start (code=exited, status=0/SUCCESS)
   Main PID: 4407 (postgres)
      Tasks: 6 (limit: 9434)
     Memory: 19.4M (peak: 27.6M)
        CPU: 1.347s
     CGroup: /system.slice/system-postgresql.slice/postgresql@16-main.service
             ├─4407 /usr/lib/postgresql/16/bin/postgres -D /backup/16/main -c config_file=/etc/postgresql/16/main/postgresql.conf
             ├─4408 "postgres: 16/main: checkpointer "
             ├─4409 "postgres: 16/main: background writer "
             ├─4411 "postgres: 16/main: walwriter "
             ├─4412 "postgres: 16/main: autovacuum launcher "
             └─4413 "postgres: 16/main: logical replication launcher "

```

**Шаг 5. Смотрим через консоль psql содержимое ранее созданной таблицы test:**

Делаем запрос: 

```
postgres=# select * from test;
 c1 
----
 1
(1 row)
```

Видим данные остались, аномалий нет.