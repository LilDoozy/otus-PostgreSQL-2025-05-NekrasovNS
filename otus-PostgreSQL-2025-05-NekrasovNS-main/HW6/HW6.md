# Домашняя работа № 6

**Шаг 1. Установил и настроил ВМ на платформе VirtualBox, в качестве дистрибутива установлена OS Linux Ubunru 24.04 с деффолтными настройками, так же установлен PostgresSQL, приступаю к созданию таблицы проведения нагрузочного тестирования и изменения параметров конфига postgres:**

Была настроена ВМ с следующими хар-ками:

```
CPU: 6 core
RAM: 8GB
HDD: 50GB
```

Установлен PostgresSQL 16.8, создана БД для тестов, так же проинициализирован pgbench:

```
postgres@studentPC:~$ pgbench -i postgres
dropping old tables...
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.04 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.82 s (drop tables 0.25 s, create tables 0.09 s, client-side generate 0.15 s, vacuum 0.08 s, primary keys 0.25 s).
```

Запускаю нагрузочное тестирование БД:

```
pgbench -c8 -P 6 -T 60 -U postgres postgres

postgres@studentPC:~$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (16.8 (Ubuntu 16.8-0ubuntu0.24.04.1))
starting vacuum...end.
progress: 6.0 s, 316.7 tps, lat 24.991 ms stddev 18.183, 0 failed
progress: 12.0 s, 316.7 tps, lat 25.185 ms stddev 21.347, 0 failed
progress: 18.0 s, 331.0 tps, lat 24.090 ms stddev 18.926, 0 failed
progress: 24.0 s, 330.4 tps, lat 24.121 ms stddev 17.886, 0 failed
progress: 30.0 s, 333.2 tps, lat 23.943 ms stddev 17.299, 0 failed
progress: 36.0 s, 323.1 tps, lat 24.667 ms stddev 19.079, 0 failed
progress: 42.0 s, 330.0 tps, lat 24.134 ms stddev 18.230, 0 failed
progress: 48.0 s, 332.1 tps, lat 24.051 ms stddev 17.754, 0 failed
progress: 54.0 s, 330.5 tps, lat 24.054 ms stddev 18.215, 0 failed
progress: 60.0 s, 330.0 tps, lat 24.205 ms stddev 18.044, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 19649
number of failed transactions: 0 (0.000%)
latency average = 24.340 ms
latency stddev = 18.519 ms
initial connection time = 25.473 ms
tps = 327.494501 (without initial connection time)
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

Видим что немного ухудшилось показание **tps**, но колчиство успешно обработанных транзакций увеличилось, показателя **latency** относительно не изменились:

```
pgbench -c8 -P 6 -T 60 -U postgres postgres

postgres@studentPC:~$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (16.8 (Ubuntu 16.8-0ubuntu0.24.04.1))
starting vacuum...end.
progress: 6.0 s, 330.3 tps, lat 23.982 ms stddev 16.667, 0 failed
progress: 12.0 s, 336.7 tps, lat 23.639 ms stddev 16.958, 0 failed
progress: 18.0 s, 325.9 tps, lat 24.472 ms stddev 18.792, 0 failed
progress: 24.0 s, 338.9 tps, lat 23.555 ms stddev 17.801, 0 failed
progress: 30.0 s, 339.3 tps, lat 23.502 ms stddev 18.213, 0 failed
progress: 36.0 s, 335.5 tps, lat 23.741 ms stddev 16.777, 0 failed
progress: 42.0 s, 323.8 tps, lat 24.640 ms stddev 21.001, 0 failed
progress: 48.0 s, 335.3 tps, lat 23.772 ms stddev 17.305, 0 failed
progress: 54.0 s, 337.5 tps, lat 23.620 ms stddev 17.147, 0 failed
progress: 60.0 s, 339.5 tps, lat 23.506 ms stddev 18.220, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 20065
number of failed transactions: 0 (0.000%)
latency average = 23.840 ms
latency stddev = 17.922 ms
initial connection time = 22.371 ms
tps = 334.373131 (without initial connection time)
```

**Шаг 2. Работаем с таблицей а так же с функцией Авто ваккум:**

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
 test    |    1000000 |          0 |      0 | 2025-04-28 12:48:02.901639+03
(1 row)
```

Обновил данные 5 раз, смотрю как изменился размер таблицы:

```
postgres=# SELECT pg_size_pretty(pg_total_relation_size('test'));
 pg_size_pretty 
----------------
 808 MB
(1 row)
```

Отключаю авто вакум таблицы и обновляю данные 10 раз, вывожу размер таблицы:

Размер таблицы возврос из-за обновлнеие данных

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
 test    |    1666551 |    9998248 |    599 | 2025-04-28 12:51:52.713327+03
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

ostgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum 
FROM pg_stat_user_TABLEs WHERE relname = 'test';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 test    |    1834412 |          0 |      0 | 2025-04-28 13:28:48.522367+03
(1 row)
```

