# Домашняя работа № 7

**1. Настрока контрольной точки и нагрузочное тестирование БД Postgres**

Настраиваем выполнение контрольной точки раз в 30 секунд. Перезагружаем службу postgres:

```

# - Checkpoints -

checkpoint_timeout = 30s                # range 30s-1d
checkpoint_completion_target = 0.9      # checkpoint target duration, 0.0 - 1.0
#checkpoint_flush_after = 256kB         # measured in pages, 0 disables
#checkpoint_warning = 30s               # 0 disables
max_wal_size = 16GB
min_wal_size = 4GB

```

НТ в течении 10 минут: 

```
postgres@pgdbvip:~$ pgbench -i postgres
dropping old tables...
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.07 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.22 s (drop tables 0.01 s, create tables 0.01 s, client-side generate 0.09 s, vacuum 0.05 s, primary keys 0.06 s).

postgres@pgdbvip:~$ pgbench -c 8 -j 8 -P 30 -T 600 -U postgres postgres
pgbench (16.10 (Ubuntu 16.10-1.pgdg22.04+1))
starting vacuum...end.
progress: 30.0 s, 2283.5 tps, lat 3.501 ms stddev 0.821, 0 failed
progress: 60.0 s, 2211.7 tps, lat 3.617 ms stddev 1.137, 0 failed
progress: 90.0 s, 2192.5 tps, lat 3.649 ms stddev 0.968, 0 failed
progress: 120.0 s, 2261.2 tps, lat 3.536 ms stddev 0.884, 0 failed
progress: 150.0 s, 2249.8 tps, lat 3.557 ms stddev 1.133, 0 failed
progress: 180.0 s, 2275.7 tps, lat 3.515 ms stddev 0.887, 0 failed
progress: 210.0 s, 2226.1 tps, lat 3.594 ms stddev 1.083, 0 failed
progress: 240.0 s, 2238.8 tps, lat 3.573 ms stddev 0.907, 0 failed
progress: 270.0 s, 2184.5 tps, lat 3.662 ms stddev 1.152, 0 failed
progress: 300.0 s, 2268.6 tps, lat 3.526 ms stddev 0.841, 0 failed
progress: 330.0 s, 2214.0 tps, lat 3.613 ms stddev 1.072, 0 failed
progress: 360.0 s, 2255.6 tps, lat 3.547 ms stddev 0.857, 0 failed
progress: 390.0 s, 2262.1 tps, lat 3.536 ms stddev 1.103, 0 failed
progress: 420.0 s, 2294.4 tps, lat 3.487 ms stddev 0.886, 0 failed
progress: 450.0 s, 2246.5 tps, lat 3.561 ms stddev 1.122, 0 failed
progress: 480.0 s, 2307.1 tps, lat 3.467 ms stddev 0.817, 0 failed
progress: 510.0 s, 2240.0 tps, lat 3.571 ms stddev 1.059, 0 failed
progress: 540.0 s, 2303.3 tps, lat 3.473 ms stddev 0.930, 0 failed
progress: 570.0 s, 2223.7 tps, lat 3.597 ms stddev 1.065, 0 failed
progress: 600.0 s, 2150.3 tps, lat 3.720 ms stddev 1.034, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 8
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 1346693
number of failed transactions: 0 (0.000%)
latency average = 3.564 ms
latency stddev = 0.996 ms
initial connection time = 17.301 ms
tps = 2244.446347 (without initial connection time)
```

Был создан log файл на 64К (Килобайт).
По умолчанию каждый файл журнала предзаписи (WAL) занимает 16 МБ:

```
postgres@pgdbvip:/var/log/postgresql$ ls -lh
total 68K
-rw-r----- 1 postgres adm 64K ноя  4 23:36 postgresql-16-main.log
```
Статистика контрольных точек:

```
postgres=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 101
checkpoints_req       | 7
checkpoint_write_time | 3232093
checkpoint_sync_time  | 189
buffers_checkpoint    | 271286
buffers_clean         | 92948
maxwritten_clean      | 929
buffers_backend       | 315916
buffers_backend_fsync | 0
buffers_alloc         | 230052
stats_reset           | 2025-11-04 14:46:59.458246+00
```

Выполняем НТ в синхронном и ассинхроном режиме:

Синхронный режим включен ***synchronous_commit = on, commit_delay = 0, commit_siblings = 5***:

```
postgres@pgdbvip:~$ pgbench -c 8 -j 6 -P 30 -T 120 -U postgres postgres
pgbench (16.10 (Ubuntu 16.10-1.pgdg22.04+1))
starting vacuum...end.
progress: 30.0 s, 2234.7 tps, lat 3.578 ms stddev 0.896, 0 failed
progress: 60.0 s, 2245.7 tps, lat 3.562 ms stddev 0.949, 0 failed
progress: 90.0 s, 2224.8 tps, lat 3.596 ms stddev 0.926, 0 failed
progress: 120.0 s, 2217.4 tps, lat 3.608 ms stddev 0.968, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 6
maximum number of tries: 1
duration: 120 s
number of transactions actually processed: 267685
number of failed transactions: 0 (0.000%)
latency average = 3.586 ms
latency stddev = 0.938 ms
initial connection time = 15.784 ms
tps = 2230.505023 (without initial connection time)
```

Теперь в асинхронном режиме ***synchronous_commit = off, wal_writer_delay = 200ms***:

Заметна разница в параметре пропускной способности базы данных (tps) в пользу ассинхронного режима, почти в 1.5 раза:

```
postgres@pgdbvip:~$ pgbench -c 8 -j 6 -P 30 -T 120 -U postgres postgres
pgbench (16.10 (Ubuntu 16.10-1.pgdg22.04+1))
starting vacuum...end.
progress: 30.0 s, 3044.6 tps, lat 2.626 ms stddev 0.598, 0 failed
progress: 60.0 s, 3080.3 tps, lat 2.597 ms stddev 0.554, 0 failed
progress: 90.0 s, 2974.2 tps, lat 2.690 ms stddev 0.807, 0 failed
progress: 120.0 s, 3110.8 tps, lat 2.571 ms stddev 0.398, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 6
maximum number of tries: 1
duration: 120 s
number of transactions actually processed: 366306
number of failed transactions: 0 (0.000%)
latency average = 2.620 ms
latency stddev = 0.609 ms
initial connection time = 16.744 ms
tps = 3052.410530 (without initial connection time)
```
При синхронном режиме: во время фиксации транзакции продолжение работы невозможно до тех пор, пока все журнальные записи об этой транзакции не окажутся на диске.

При асинхронном режиме: транзакция завершается немедленно, журнал записывается в фоновом режиме, асинхронная запись эффективнее синхронной, т.к. фиксация изменений не ждёт записи. Надёжность уменьшается, зафиксированные данные могут пропасть в случае сбоя. 

В реальности оба этих режима работают совместно. Даже при синхронной фиксации журнальные записи долгой транзакции будут записываться асинхронно, чтобы освободить буферы WAL.

**Создаем новый кластер с включенной контрольной суммой страниц, cоздаем таблицу, вставляем несколько значений, включаем кластер, меняем пару байтов в таблице и делаем выборку из таблицы:**

Во время проделанных действий получил ошибку по несходству контрольной суммы страниц:

```
page verification failed, calculated checksum 21135 but expected 3252
```

Чтобы проигнорировать проблему, можно включить параметр ignore_checksum_failure.
