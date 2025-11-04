# Домашняя работа № 6

**1. Установил и настроил ВМ на платформе VirtualBox, в качестве дистрибутива установлена OS Linux Ubunru 24.04 с деффолтными настройками, так же установлен PostgresSQL, приступаю к созданию таблицы проведения нагрузочного тестирования и изменения параметров конфига postgres:**

Была настроена ВМ с следующими хар-ками:

```
CPU: 1 core
RAM: 2GB
HDD: 30GB
```

Установлен PostgresSQL 16.10, создана БД для тестов, так же проинициализирован pgbench:

```
postgres@pgdbvip:~$ pgbench -i  postgres
dropping old tables...
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.06 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.21 s (drop tables 0.01 s, create tables 0.00 s, client-side generate 0.08 s, vacuum 0.05 s, primary keys 0.06 s).
```

Запускаю нагрузочное тестирование БД:

```
pgbench -c8 -P 6 -T 60 -U postgres postgres

postgres@pgdbvip:~$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (16.10 (Ubuntu 16.10-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 2000.5 tps, lat 3.987 ms stddev 1.189, 0 failed
progress: 12.0 s, 2011.0 tps, lat 3.978 ms stddev 1.158, 0 failed
progress: 18.0 s, 2023.1 tps, lat 3.953 ms stddev 1.165, 0 failed
progress: 24.0 s, 1991.0 tps, lat 4.017 ms stddev 1.169, 0 failed
progress: 30.0 s, 1944.4 tps, lat 4.114 ms stddev 1.588, 0 failed
progress: 36.0 s, 1958.6 tps, lat 4.084 ms stddev 1.426, 0 failed
progress: 42.0 s, 2034.0 tps, lat 3.933 ms stddev 1.138, 0 failed
progress: 48.0 s, 2010.3 tps, lat 3.979 ms stddev 1.154, 0 failed
progress: 54.0 s, 2038.7 tps, lat 3.924 ms stddev 1.130, 0 failed
progress: 60.0 s, 1987.5 tps, lat 4.025 ms stddev 1.169, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 120003
number of failed transactions: 0 (0.000%)
latency average = 3.999 ms
latency stddev = 1.242 ms
initial connection time = 15.380 ms
tps = 1999.690689 (without initial connection time)
```

Применяю параметры для postgres конфига приложенный к занятию:

Конфиг из занятия:

```
max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB
```
После применения конфига и рестарта службы postgres, запускаем нагрузочное тестирование снова с теми же параметрами:

```
pgbench -c8 -P 6 -T 60 -U postgres postgres

postgres@pgdbvip:~$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (16.10 (Ubuntu 16.10-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 1962.0 tps, lat 4.065 ms stddev 1.265, 0 failed
progress: 12.0 s, 2033.0 tps, lat 3.935 ms stddev 1.215, 0 failed
progress: 18.0 s, 2018.5 tps, lat 3.963 ms stddev 1.187, 0 failed
progress: 24.0 s, 1939.8 tps, lat 4.123 ms stddev 1.604, 0 failed
progress: 30.0 s, 2020.3 tps, lat 3.959 ms stddev 1.134, 0 failed
progress: 36.0 s, 2030.0 tps, lat 3.940 ms stddev 1.170, 0 failed
progress: 42.0 s, 2031.2 tps, lat 3.939 ms stddev 1.140, 0 failed
progress: 48.0 s, 2034.8 tps, lat 3.931 ms stddev 1.100, 0 failed
progress: 54.0 s, 1995.2 tps, lat 4.008 ms stddev 1.912, 0 failed
progress: 60.0 s, 2013.2 tps, lat 3.974 ms stddev 1.169, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 120476
number of failed transactions: 0 (0.000%)
latency average = 3.984 ms
latency stddev = 1.319 ms
initial connection time = 15.011 ms
tps = 2007.410269 (without initial connection time)

```
Видим что немного ухудшилось показание tps, но колчиство успешно обработанных транзакций увеличилось, показателя latency относительно не изменились

**2. Работаем с таблицей, а так же с функцией autovacuum:**

Создаем таблицу и отключаем autovacuum, генерируем в нее данные:

```
postgres=# CREATE TABLE test(
  id serial,
  fio char(100)
) WITH (autovacuum_enabled = off);
CREATE TABLE
postgres=# INSERT INTO test(fio) SELECT 'noname' FROM generate_series(1,1000000);
INSERT 0 1000000
```

Смотрим размер таблицы:

```
postgres=# SELECT pg_size_pretty(pg_total_relation_size('test'));
 pg_size_pretty 
----------------
 135 MB
(1 row)
```
Обновил строчки 5 раз, смотрим мертвые строчки и когда был последний Авто вакум:

```
postgres=# update test set fio = 'name';
UPDATE 1000000
```

```
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum 
FROM pg_stat_user_TABLEs WHERE relname = 'test';
 relname | n_live_tup | n_dead_tup | ratio% | last_autovacuum 
---------+------------+------------+--------+-----------------
 test    |    1000000 |    4999944 |    499 | 
(1 row)
```

Авто вакум был отключен, включил его принудительно, снова делаю запрос на мертвые строчки и запрос последнего авто вакума:

```
postgres=# ALTER TABLE test SET (autovacuum_enabled = on);
ALTER TABLE

postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum
FROM pg_stat_user_TABLEs WHERE relname = 'test';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 test    |    1000000 |          0 |      0 | 2025-11-04 18:08:08.543033+00
(1 row)
```

Обновил данные 5 раз, смотрю как изменился размер таблицы:

```
postgres=# SELECT pg_size_pretty(pg_total_relation_size('test'));
 pg_size_pretty
----------------
 943 MB
(1 row)
```

Отключаю autovacuum таблицы и обновляю данные 10 раз, вывожу размер таблицы:

Размер таблицы возрос из-за обновления данных

```
postgres=# ALTER TABLE test SET (autovacuum_enabled = off);
ALTER TABLE

postgres=# SELECT pg_size_pretty(pg_total_relation_size('test'));
 pg_size_pretty 
----------------
 1482 MB
(1 row)
``` 

Дополнительно смотрю на состояние строк в таблице без авто вакума:

```
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum
FROM pg_stat_user_TABLEs WHERE relname = 'test';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 test    |    1000000 |    9997373 |    999 | 2025-11-04 18:11:09.090497+00
(1 row)
```

Включаю авто вакуум в таблице:

Автовакуум в PostgreSQL это процесс, который автоматически очищает базу от «мертвых» строк. PostgreSQL использует модель MVCC, при которой каждая операция изменения данных не удаляет или не обновляет строку, а создаёт её новую версию, оставляя старую на месте. Эти старые версии строк остаются в таблице до тех пор, пока не будет выполнено вакуумирование. Они занимают место и замедляют работу базы, увеличивая объём, который приходится сканировать при выполнении запросов.

Автовакуум нужен для:

Удаления неактуальных строк, освобождая место для новых данных.

Обновления статистики для оптимизатора запросов.

Предотвращения переполнения транзакций (так называемый wraparound).

```
postgres=# ALTER TABLE test SET (autovacuum_enabled = on);
ALTER TABLE

postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum
FROM pg_stat_user_TABLEs WHERE relname = 'test';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 test    |     998008 |          0 |      0 | 2025-11-04 18:15:41.037615+00

```

