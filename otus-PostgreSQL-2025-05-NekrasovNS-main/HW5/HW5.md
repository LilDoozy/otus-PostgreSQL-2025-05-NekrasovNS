# Домашняя работа № 5

**1. Установил и развернул виртуальную машину средствами ПО VirtualBox, накатил ОС Linux Ubuntu 24.04 с дефолтными настройками системы.**

Характеристики ВМ:

```
CPU: 1 core
RAM: 2 GB
HDD: 30 GB

```

**2. Пробую прогнать тест производительности стандартной БД по умолчанию -d postgres без измененеия конфига производительности, чтобы увидить разницу до и после правки конфига:**


```
pgbenсh -с 50 -j 2 -P60 -T600 -d postgres

transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 382084
number of failed transactions: 0 (0.000%)
latency average = 78.240 ms
latency stddev = 29.774 ms
initial connection time = 91.516 ms
tps = 636.723303 (without initial connection time)
```

Правим конфиги производительности с помощью калькуляторов производительности PGTUNE (Тип БД выбрал OLTP):

```
# DB Version: 16
# OS Type: linux
# DB Type: oltp
# Total Memory (RAM): 2 GB
# CPUs num: 1
# Data Storage: hdd

max_connections = 300
shared_buffers = 512MB
effective_cache_size = 1536MB
maintenance_work_mem = 128MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 873kB
huge_pages = off
min_wal_size = 2GB
max_wal_size = 8GB
```
Запускаем тест производительности повторно, после правки кофига с данными с PGTUNE:

Удалось снизить показатель tps с 636 до 616, но показатели latency немного увеличились.

```
pgbenсh -с 50 -j 2 -P60 -T600 -d postgres

transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 369752
number of failed transactions: 0 (0.000%)
latency average = 80.943 ms
latency stddev = 31.679 ms
initial connection time = 85.341 ms
tps = 616.130081 (without initial connection time)
```

