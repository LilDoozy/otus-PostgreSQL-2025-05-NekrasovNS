# Домашняя работа № 2

**Шаг.1 Развернул виртуальную ОС на платформе виртуализации VirtualBox, установил ОС Linux Ubuntu 24.04 с дефолтными настройками машины.**

**Шаг 2. Установил и развернул Docker Engine на ОС:**

```
student@studentPC:~$ systemctl status docker
● docker.service - Docker Application Container Engine
     Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; preset: e>
     Active: active (running) since Sun 2025-04-13 20:13:32 MSK; 1min 47s ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 1339 (dockerd)
      Tasks: 12
     Memory: 99.9M (peak: 100.8M)
        CPU: 2.242s
     CGroup: /system.slice/docker.service
             └─1339 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/cont>

```

**Шаг 3. Создал директорию в ОС: /var/lib/postgres и развернул туда контейнер docker с образом postgresql 17.4 (Смонтировал):**
```

initdb: warning: enabling "trust" authentication for local connections

initdb: hint: You can change this by editing pg_hba.conf or using the option -A, or --auth-local and --auth-host, the next time you run initdb.

waiting for server to start....2025-04-13 17:23:52.804 UTC [48] LOG: starting PostgreSQL 17.4 (Debian 17.4-1.pgdg120+2) on x86_64-pc-linux-gnu, compiled by gcc (Debian 12.2.0-14) 12.2.0, 64-bit

2025-04-13 17:23:52.807 UTC [48] LOG: listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"

2025-04-13 17:23:52.815 UTC [51] LOG: database system was shut down at 2025-04-13 17:23:52 UTC

2025-04-13 17:23:52.820 UTC [48] LOG: database system is ready to accept connections

done

server started

/usr/local/bin/docker-entrypoint.sh: ignoring /docker-entrypoint-initdb.d/*

2025-04-13 17:23:52.935 UTC [48] LOG: received fast shutdown request

waiting for server to shut down....2025-04-13 17:23:52.938 UTC [48] LOG: aborting any active transactions

2025-04-13 17:23:52.939 UTC [48] LOG: background worker "logical replication launcher" (PID 54) exited with exit code 1

2025-04-13 17:23:52.939 UTC [49] LOG: shutting down

2025-04-13 17:23:52.941 UTC [49] LOG: checkpoint starting: shutdown immediate

2025-04-13 17:23:52.955 UTC [49] LOG: checkpoint complete: wrote 3 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.005 s, sync=0.002 s, total=0.017 s; sync files=2, longest=0.002 s, average=0.001 s; distance=0 kB, estimate=0 kB; lsn=0/14E4FA0, redo lsn=0/14E4FA0

2025-04-13 17:23:52.959 UTC [48] LOG: database system is shut down

done

server stopped

PostgreSQL init process complete; ready for start up.

2025-04-13 17:23:53.056 UTC [1] LOG: starting PostgreSQL 17.4 (Debian 17.4-1.pgdg120+2) on x86_64-pc-linux-gnu, compiled by gcc (Debian 12.2.0-14) 12.2.0, 64-bit

2025-04-13 17:23:53.056 UTC [1] LOG: listening on IPv4 address "0.0.0.0", port 5432

2025-04-13 17:23:53.056 UTC [1] LOG: listening on IPv6 address "::", port 5432

2025-04-13 17:23:53.061 UTC [1] LOG: listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"

2025-04-13 17:23:53.067 UTC [62] LOG: database system was shut down at 2025-04-13 17:23:52 UTC

2025-04-13 17:23:53.071 UTC [1] LOG: database system is ready to accept connections
```
**Шаг 4. Создаю простую таблицу через контейнер docker и проверяю ее в postgresql:**
```
CREATE TABLE web_origins (
    client_id character varying(36) NOT NULL,
    value character varying(255)
);
```
Делаю вывод команды таблиц /dt:

```
root:/# psql -h localhost -p 5432 -U docker -d docker
psql (17.4.)
Type "help" for help.
docker=# \c
You are now connected to database "docker" as user "docker".
docker=# \dt
           List of relations
 Schema |    Name     | Type  | Owner
--------+-------------+-------+--------
 public | web_origins | table | docker
(1 row)
```
**Шаг 5. Успешно подключился к контейнеру с сервером извне инстансов, удалил серверный контйенер, создал его снова, повторно подключился к контейнеру сервера, снова вывел список таблиц:**

```
root:/# psql -h localhost -p 5432 -U docker -d docker
psql (17.4.)
Type "help" for help.
docker=# \c
You are now connected to database "docker" as user "docker".
docker=# \dt
           List of relations
 Schema |    Name     | Type  | Owner
--------+-------------+-------+--------
 public | web_origins | table | docker
(1 row)
```

В основном сталкнулся с проблемой развертывания postgres образа, выходила ошибка при запуске контейнера:

* *Error: Database is uninitialized and superuser password is not specified.* 

Было решено с помощью команды:

```
docker run -e POSTGRES_PASSWORD=<my_password> postgres:17.4
```

