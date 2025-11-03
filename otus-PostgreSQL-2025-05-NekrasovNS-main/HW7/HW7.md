# Домашняя работа № 7

**Шаг 1. Настрока контрольной точки + нагрузочное тестирование БД Postgres**

Настроил выполнение контрольной точки раз в 30 секунд - по умолчанию параметр раз в 5 минут - Перезагружаю службу postgres:

```
# - Checkpoints -

checkpoint_timeout = 30s                # range 30s-1d
checkpoint_completion_target = 0.9      # checkpoint target duration, 0.0 - 1.0
#checkpoint_flush_after = 256kB         # measured in pages, 0 disables
#checkpoint_warning = 30s               # 0 disables
max_wal_size = 2GB
min_wal_size = 1GB

postgres@studentPC:~$ systemctl restart postgresql
```

Выполняю нагрузочное тестирование в течении 10 минут: 

```
postgres@studentPC:~$ pgbench -i postgres
dropping old tables...
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.04 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.80 s (drop tables 0.20 s, create tables 0.07 s, client-side generate 0.14 s, vacuum 0.28 s, primary keys 0.10 s).
postgres@studentPC:~$ pgbench -c 8 -j 8 -P 30 -T 600 -U postgres postgres
pgbench (16.8 (Ubuntu 16.8-0ubuntu0.24.04.1))
starting vacuum...end.
progress: 30.0 s, 302.3 tps, lat 26.407 ms stddev 19.204, 0 failed
progress: 60.0 s, 335.9 tps, lat 23.780 ms stddev 17.057, 0 failed
progress: 90.0 s, 325.5 tps, lat 24.541 ms stddev 18.705, 0 failed
progress: 120.0 s, 335.8 tps, lat 23.790 ms stddev 17.701, 0 failed
progress: 150.0 s, 338.2 tps, lat 23.622 ms stddev 16.976, 0 failed
progress: 180.0 s, 334.1 tps, lat 23.907 ms stddev 17.604, 0 failed
progress: 210.0 s, 332.5 tps, lat 24.027 ms stddev 17.612, 0 failed
progress: 240.0 s, 335.9 tps, lat 23.786 ms stddev 16.852, 0 failed
progress: 270.0 s, 326.3 tps, lat 24.471 ms stddev 17.755, 0 failed
progress: 300.0 s, 335.4 tps, lat 23.818 ms stddev 17.345, 0 failed
progress: 330.0 s, 330.9 tps, lat 24.145 ms stddev 16.846, 0 failed
progress: 360.0 s, 334.5 tps, lat 23.882 ms stddev 17.686, 0 failed
progress: 390.0 s, 334.9 tps, lat 23.849 ms stddev 17.088, 0 failed
progress: 420.0 s, 333.2 tps, lat 23.979 ms stddev 17.867, 0 failed
progress: 450.0 s, 335.3 tps, lat 23.831 ms stddev 17.147, 0 failed
progress: 480.0 s, 333.0 tps, lat 23.984 ms stddev 17.696, 0 failed
progress: 510.0 s, 335.9 tps, lat 23.788 ms stddev 17.423, 0 failed
progress: 540.0 s, 335.0 tps, lat 23.837 ms stddev 17.301, 0 failed
progress: 570.0 s, 326.6 tps, lat 24.467 ms stddev 17.806, 0 failed
progress: 600.0 s, 326.0 tps, lat 24.499 ms stddev 18.453, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 8
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 198822
number of failed transactions: 0 (0.000%)
latency average = 24.107 ms
latency stddev = 17.614 ms
initial connection time = 13.200 ms
tps = 331.359840 (without initial connection time)
```

Был создан log файл на 18К (Килобайт) - По умолчанию каждый файл журнала предзаписи (WAL) в PostgreSQL занимает 16 МБ:

```
postgres@studentPC:/var/log/postgresql$ ls -lh
итого 48K
-rw-r----- 1 postgres adm  18K мая  3 20:56 postgresql-16-main.log
-rw-r----- 1 postgres adm  20K апр 28 11:21 postgresql-16-main.log.1
-rw-r----- 1 postgres adm 3,2K апр 20 11:50 postgresql-16-main.log.2.gz
```
Статистика выполнения контрольных точек:

```
postgres=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 155
checkpoints_req       | 11
checkpoint_write_time | 3742073
checkpoint_sync_time  | 2741
buffers_checkpoint    | 336499
buffers_clean         | 128529
maxwritten_clean      | 1284
buffers_backend       | 333447
buffers_backend_fsync | 0
buffers_alloc         | 237983
stats_reset           | 2025-04-19 14:44:04.731716+03
```

Выполняю нагрузочное тестирование в синхронном и ассинхроном режиме (Сравниваем tps):

Синхронный режим включен по умолчанию (***synchronous_commit = on, commit_delay = 0, commit_siblings = 5***)

```
postgres@studentPC:~$ pgbench -c 8 -j 6 -P 30 -T 120 -U postgres postgres
pgbench (16.8 (Ubuntu 16.8-0ubuntu0.24.04.1))
starting vacuum...end.
progress: 30.0 s, 328.6 tps, lat 24.286 ms stddev 19.106, 0 failed
progress: 60.0 s, 337.1 tps, lat 23.687 ms stddev 17.745, 0 failed
progress: 90.0 s, 338.8 tps, lat 23.582 ms stddev 17.274, 0 failed
progress: 120.0 s, 339.8 tps, lat 23.497 ms stddev 16.786, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 6
maximum number of tries: 1
duration: 120 s
number of transactions actually processed: 40338
number of failed transactions: 0 (0.000%)
latency average = 23.761 ms
latency stddev = 17.741 ms
initial connection time = 14.319 ms
tps = 336.107547 (without initial connection time)
```

Теперь пробуем в асинхронном режиме (***synchronous_commit = off, wal_writer_delay = 200ms***):

Видим большую разницу в параметре пропускной способности базы данных (tps) в пользу ассинхронного режима, почти в 2.5 раза:

```
postgres@studentPC:~$ pgbench -c 8 -j 6 -P 30 -T 120 -U postgres postgres
pgbench (16.8 (Ubuntu 16.8-0ubuntu0.24.04.1))
starting vacuum...end.
progress: 30.0 s, 903.0 tps, lat 8.809 ms stddev 5.268, 0 failed
progress: 60.0 s, 898.7 tps, lat 8.858 ms stddev 5.353, 0 failed
progress: 90.0 s, 909.1 tps, lat 8.757 ms stddev 5.295, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 6
maximum number of tries: 1
duration: 120 s
number of transactions actually processed: 108347
number of failed transactions: 0 (0.000%)
latency average = 8.816 ms
latency stddev = 5.315 ms
initial connection time = 15.051 ms
tps = 902.934074 (without initial connection time)
```
При синхронном режиме - При фиксации транзакции продолжение работы невозможно до тех пор, пока все журнальные записи об этой транзакции не окажутся на диске.

При асинхронном режиме- Транзакция завершается немедленно, журнал записывается в фоновом режиме, асинхронная запись эффективнее синхронной т.к. фиксация изменений не ждёт записи. Однако надёжность уменьшается, зафиксированные данные могут пропасть в случае сбоя. 

В реальности оба этих режима работают совместно. Даже при синхронной фиксации журнальные записи долгой транзакции будут записываться асинхронно, чтобы освободить буферы WAL.



**Создал новый кластер с включенной контрольной суммой страниц, cоздал таблицу, вставил несколько значений, включил кластер, изменил пару байт в таблице, включил кластер и сделал выборку из таблицы:**

Во время проделанных действий получил ошибку по несходству контрольной суммы страниц:

```
page verification failed, calculated checksum 21135 but expected 3252
```

Чтобы проигнорировать проблему, можно включить параметр ignore_checksum_failure.
