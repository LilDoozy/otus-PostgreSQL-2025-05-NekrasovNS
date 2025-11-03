# Домашняя работа № 5

**Шаг 1. Установил и развернул виртуальную машину средствами ПО VirtualBox, накатил ОС Linux Ubuntu 24.04 с дефолтными настройками системы.**

Характеристики ВМ:

```
CPU: 6 core
RAM: 8 GB
HDD (not SSD): 50 GB
VRAM: 16MB
```

**Шаг 2. Пробую прогнать тест производительности стандартной БД по умолчанию -d postgres без измененеия конфига производительности, чтобы увидить разницу до и после правки конфига:**


```
pgbenсh -с 50 -j 2 -P60 -T600 -d postgres

transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 106436
number of failed transactions: 0 (0.000%)
latency average = 281.709 ms
latency stddev = 392.348 ms
initial connection time = 110.875 ms
tps = 177.376486 (without initial connection time)
```

Правим конфиги производительности с помощью калькуляторов производительности PGTUNE (Тип БД выбрал OLTP):

```
# DB Version: 16
# OS Type: linux
# DB Type: oltp
# Total Memory (RAM): 8 GB
# CPUs num: 6
# Connections num: 100
# Data Storage: hdd

max_connections = 100
shared_buffers = 2GB
effective_cache_size = 6GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 8MB
huge_pages = off
min_wal_size = 2GB
max_wal_size = 8GB
max_worker_processes = 6
max_parallel_workers_per_gather = 3
max_parallel_workers = 8
max_parallel_maintenance_workers = 3
```
Запускаем тест производительности повторно, после правки кофига с данными с PGTUNE:

Удалось снизить показатель tps с 177 до 170, но показатели latency немного увеличились.

```
pgbenсh -с 50 -j 2 -P60 -T600 -d postgres

transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 102104
number of failed transactions: 0 (0.000%)
latency average = 293.651 ms
latency stddev = 406.084 ms
initial connection time = 141.607 ms
tps = 170.158610 (without initial connection time)
```

