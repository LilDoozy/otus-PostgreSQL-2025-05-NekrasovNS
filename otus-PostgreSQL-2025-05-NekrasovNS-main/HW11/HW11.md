# Домашняя работа № 11 | Секционирование таблицы

**Шаг 1: Скачиваю и разварачиваю тестовую БД demo:**

```
wget -P /tmp/ https://edu.postgrespro.ru/demo-big.zip

psql -f /tmp/demo-big-20170815.sql -U postgres
```


```
Name    |  Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | ICU Locale | ICU Rules |   Access privileges   |  Size   | Tablespace |                Description                 
-----------+----------+----------+-----------------+-------------+-------------+------------+-----------+-----------------------+---------+------------+--------------------------------------------
 demo      | postgres | UTF8     | libc            | ru_RU.UTF-8 | ru_RU.UTF-8 |            |           |                       | 2666 MB | pg_default | 
 dvdrental | postgres | UTF8     | libc            | ru_RU.UTF-8 | ru_RU.UTF-8 |            |           |                       | 15 MB   | pg_default | 
 joins     | postgres | UTF8     | libc            | ru_RU.UTF-8 | ru_RU.UTF-8 |            |           |                       | 7628 kB | pg_default | 
 postgres  | postgres | UTF8     | libc            | ru_RU.UTF-8 | ru_RU.UTF-8 |            |           |                       | 29 MB   | pg_default | default administrative connection database
 template0 | postgres | UTF8     | libc            | ru_RU.UTF-8 | ru_RU.UTF-8 |            |           | =c/postgres          +| 7361 kB | pg_default | unmodifiable empty database
           |          |          |                 |             |             |            |           | postgres=CTc/postgres |         |            | 
 template1 | postgres | UTF8     | libc            | ru_RU.UTF-8 | ru_RU.UTF-8 |            |           | =c/postgres          +| 7580 kB | pg_default | default template for new databases
           |          |          |                 |             |             |            |           | postgres=CTc/postgres |         |            | 

```

Проверяем БД, все ли ок, выведем все ьаблички:

```
postgres=# \c demo 
You are now connected to database "demo" as user "postgres".
demo=# \dt
               List of relations
  Schema  |      Name       | Type  |  Owner   
----------+-----------------+-------+----------
 bookings | aircrafts_data  | table | postgres
 bookings | airports_data   | table | postgres
 bookings | boarding_passes | table | postgres
 bookings | bookings        | table | postgres
 bookings | flights         | table | postgres
 bookings | seats           | table | postgres
 bookings | ticket_flights  | table | postgres
 bookings | tickets         | table | postgres
(8 rows)
```

Узнаем размер самой большой таблички и возмем ее за основу работы:

Исходя из запроса выберем табличку *ticket_flights* ее размер 547МБ

```
demo=# select pg_size_pretty(pg_table_size('bookings.ticket_flights'));
select pg_size_pretty(pg_table_size('bookings.flights'));
select pg_size_pretty(pg_table_size('bookings.aircrafts')); 
select pg_size_pretty(pg_table_size('bookings.airports'));
select pg_size_pretty(pg_table_size('bookings.boarding_passes')); 
select pg_size_pretty(pg_table_size('bookings.bookings')); 
select pg_size_pretty (pg_table_size('bookings.seats'));

 pg_size_pretty 
----------------
 547 MB
(1 row)

 pg_size_pretty 
----------------
 21 MB
(1 row)

 pg_size_pretty 
----------------
 0 bytes
(1 row)

 pg_size_pretty 
----------------
 0 bytes
(1 row)

 pg_size_pretty 
----------------
 456 MB
(1 row)

 pg_size_pretty 
----------------
 105 MB
(1 row)

 pg_size_pretty 
----------------
 96 kB
(1 row)
```

**Шаг 2: Создаем таблицу для секционирования по списку:**

Приступаем к созданию таблицы:

```
demo=# 
create table bookings.ticket_flights_partitioning (like bookings.ticket_flights) partition by list (fare_conditions);
CREATE TABLE
```

В новосозданной таблице создаем партиции:

```
demo=# create table bookings.ticket_flights_economy partition of bookings.ticket_flights_partitioning for values in ('Economy');
CREATE TABLE
create table bookings.ticket_flights_comfort partition of bookings.ticket_flights_partitioning for values in ('Comfort');
CREATE TABLE
create table bookings.ticket_flights_business partition of bookings.ticket_flights_partitioning for values in ('Business');
CREATE TABLE
```

**Шаг 3: Подставляем данные и смотрим как изменяется производительность, так же произведем сбор данных:**

Подставляем данные:

```
demo=# insert into bookings.ticket_flights_partitioning(ticket_no, flight_id, fare_conditions, amount) 
select ticket_no, 
       flight_id,
       fare_conditions,
       amount
from bookings.ticket_flights;
INSERT 0 8391852
```
Пробуем сделать запрос к несекционированной и секционированной таблице, делаем анализ производительности запросов:

```
Несекционированный запрос:

demo=# explain analyze 
select fp.ticket_no,
       sum(fp.amount) as total_amount
from bookings.ticket_flights as fp 
where 1=1
and fp.fare_conditions = 'Economy' 
group by fp.ticket_no;
                                                                              QUERY PLAN                                                                              
----------------------------------------------------------------------------------------------------------------------------------------------------------------------
 GroupAggregate  (cost=0.56..660471.82 rows=2522765 width=46) (actual time=743.888..338845.019 rows=2893799 loops=1)
   Group Key: ticket_no
   ->  Index Scan using ticket_flights_pkey on ticket_flights fp  (cost=0.56..591970.03 rows=7393446 width=20) (actual time=743.463..148804.085 rows=7392231 loops=1)
         Filter: ((fare_conditions)::text = 'Economy'::text)
         Rows Removed by Filter: 999621
 Planning Time: 1.492 ms
 JIT:
   Functions: 7
   Options: Inlining true, Optimization true, Expressions true, Deforming true
   Timing: Generation 18.972 ms, Inlining 139.830 ms, Optimization 244.363 ms, Emission 327.223 ms, Total 730.387 ms
 Execution Time: 390883.594 ms
(11 rows)

```

Другой запрос:

```
Секционированный запрос:

demo=# explain analyze                                                                                 
select fp.ticket_no,
       sum(fp.amount) as total_amount
from bookings.ticket_flights_partitioning as fp 
where 1=1
and fp.fare_conditions = 'Economy' 
group by fp.ticket_no;
                                                                               QUERY PLAN                                                                                
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Finalize GroupAggregate  (cost=104342.52..104420.52 rows=200 width=46) (actual time=165044.013..510193.332 rows=2893799 loops=1)
   Group Key: fp.ticket_no
   ->  Gather Merge  (cost=104342.52..104413.52 rows=600 width=46) (actual time=165043.452..336045.211 rows=5728477 loops=1)
         Workers Planned: 3
         Workers Launched: 3
         ->  Sort  (cost=103342.48..103342.98 rows=200 width=46) (actual time=164500.683..195446.293 rows=1432119 loops=4)
               Sort Key: fp.ticket_no
               Sort Method: external merge  Disk: 123304kB
               Worker 0:  Sort Method: external merge  Disk: 123328kB
               Worker 1:  Sort Method: external merge  Disk: 124160kB
               Worker 2:  Sort Method: external merge  Disk: 125312kB
               ->  Partial HashAggregate  (cost=103332.34..103334.84 rows=200 width=46) (actual time=91739.315..126167.749 rows=1432119 loops=4)
                     Group Key: fp.ticket_no
                     Batches: 85  Memory Usage: 13377kB  Disk Usage: 83728kB
                     Worker 0:  Batches: 101  Memory Usage: 13377kB  Disk Usage: 84392kB
                     Worker 1:  Batches: 101  Memory Usage: 13377kB  Disk Usage: 84928kB
                     Worker 2:  Batches: 85  Memory Usage: 13377kB  Disk Usage: 85728kB
                     ->  Parallel Seq Scan on ticket_flights_economy fp  (cost=0.00..91409.38 rows=2384591 width=20) (actual time=8.932..44994.428 rows=1848058 loops=4)
                           Filter: ((fare_conditions)::text = 'Economy'::text)
 Planning Time: 0.505 ms
 JIT:
   Functions: 47
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 4.158 ms, Inlining 0.000 ms, Optimization 2.213 ms, Emission 45.587 ms, Total 51.958 ms
 Execution Time: 564723.399 ms
(25 rows)

```
Как мы видим несекционный запрос выполнился сильно быстрее, но время генерации у секционного запроса лучше.

Дополнительно соберем статистику по таблице:

```
demo=# analyze verbose bookings.ticket_flights_partitioning;
INFO:  analyzing "bookings.ticket_flights_partitioning" inheritance tree
INFO:  "ticket_flights_comfort": scanned 1167 of 1167 pages, containing 139965 live rows and 0 dead rows; 2503 rows in sample, 139965 estimated total rows
INFO:  "ticket_flights_economy": scanned 61602 of 61602 pages, containing 7392231 live rows and 0 dead rows; 132131 rows in sample, 7392231 estimated total rows
INFO:  "ticket_flights_business": scanned 7164 of 7164 pages, containing 859656 live rows and 0 dead rows; 15366 rows in sample, 859656 estimated total rows
INFO:  analyzing "bookings.ticket_flights_comfort"
INFO:  "ticket_flights_comfort": scanned 1167 of 1167 pages, containing 139965 live rows and 0 dead rows; 139965 rows in sample, 139965 estimated total rows
INFO:  analyzing "bookings.ticket_flights_economy"
INFO:  "ticket_flights_economy": scanned 61602 of 61602 pages, containing 7392231 live rows and 0 dead rows; 150000 rows in sample, 7392231 estimated total rows
INFO:  analyzing "bookings.ticket_flights_business"
INFO:  "ticket_flights_business": scanned 7164 of 7164 pages, containing 859656 live rows and 0 dead rows; 150000 rows in sample, 859656 estimated total rows
ANALYZE
```

Повторим запрос с секционным запросом:

Запрос собрался же намного лучше, прирост в производительности почти в 2 раза (Время генерации и время исполнения):

```
demo=# explain analyze                                      
select fp.ticket_no,
       sum(fp.amount) as total_amount
from bookings.ticket_flights_partitioning as fp 
where 1=1
and fp.fare_conditions = 'Economy' 
group by fp.ticket_no;
                                                                   QUERY PLAN                                                                    
-------------------------------------------------------------------------------------------------------------------------------------------------
 HashAggregate  (cost=685321.49..799549.95 rows=2208060 width=46) (actual time=269609.476..323941.317 rows=2893799 loops=1)
   Group Key: fp.ticket_no
   Planned Partitions: 64  Batches: 321  Memory Usage: 13369kB  Disk Usage: 322968kB
   ->  Seq Scan on ticket_flights_economy fp  (cost=0.00..154004.89 rows=7392231 width=20) (actual time=37.251..132723.178 rows=7392231 loops=1)
         Filter: ((fare_conditions)::text = 'Economy'::text)
 Planning Time: 0.361 ms
 JIT:
   Functions: 11
   Options: Inlining true, Optimization true, Expressions true, Deforming true
   Timing: Generation 0.605 ms, Inlining 8.818 ms, Optimization 28.866 ms, Emission 18.932 ms, Total 57.222 ms
 Execution Time: 374096.447 ms
(11 rows)
```