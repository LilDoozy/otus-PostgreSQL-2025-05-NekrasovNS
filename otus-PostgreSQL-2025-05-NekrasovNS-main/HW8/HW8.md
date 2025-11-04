# Домашняя работа № 8


**1. Стоит задача настроить сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Так же нужно попытаться воспроизвести ситуацию, при которой в журнале будут появляться такие сообщения.**

Проверка стандартных параметров значений log_lock_waits и deadlock_timeout:

```
locks=# show log_lock_waits;
 log_lock_waits
----------------
 off
(1 строка)

locks=# show deadlock_timeout;
 deadlock_timeout
------------------
 1s
(1 строка)
```

Изменение параметров и перезапуск службы:

```
locks=# alter system set deadlock_timeout = 200;
ALTER SYSTEM
locks=# alter system set log_lock_waits = on;
ALTER SYSTEM

Рестарт службы и проверка измененных параметров:

locks=# show log_lock_waits;
 log_lock_waits
----------------
 on
(1 строка)

locks=# show deadlock_timeout;
 deadlock_timeout
------------------
 200ms
(1 строка)
```

###2. pg_locks и блокировки.

Создаем базу и таблицу с произвольными значениями:

```
postgres=# create database locks;
CREATE DATABASE
CREATE DATABASE
postgres=# \c locks
Вы подключены к базе данных "locks" как пользователь "postgres".
locks=# create table accounts (acc_no integer PRIMARY KEY, amount numeric);
CREATE TABLE
locks=# insert into accounts values (1,1000.00),(2,2000.00),(3,3000.00);
INSERT 0 3
```

Проверка активных блокировок:

```
locks=*# select pg_backend_pid();
 pg_backend_pid 
----------------
           4546
(1 строка)

locks=*# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted FROM pg_locks WHERE pid = 4546;
  locktype  | relation | virtxid | xid |      mode       | granted 
------------+----------+---------+-----+-----------------+---------
 relation   | pg_locks |         |     | AccessShareLock | t
 virtualxid |          | 5/78    |     | ExclusiveLock   | t

```

Запустим транзакцию в параллельной сессии и обновим строку, проверим на наличие новых блокировок:

```
locks=# begin;
BEGIN

locks=*# UPDATE accounts SET amount = amount + 100 WHERE acc_no = 1;
UPDATE 1
locks=*# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted FROM pg_locks WHERE pid = 4546;
   locktype    |   relation    | virtxid |  xid   |       mode       | granted 
---------------+---------------+---------+--------+------------------+---------
 relation      | accounts_pkey |         |        | RowExclusiveLock | t
 relation      | accounts      |         |        | RowExclusiveLock | t
 relation      | pg_locks      |         |        | AccessShareLock  | t
 virtualxid    |               | 5/78    |        | ExclusiveLock    | t
 transactionid |               |         | 596835 | ExclusiveLock    | t
(5 строк)

```

Откроем еще одну сессию и сделаем индекс на поле **acc_no integer PRIMARY KEY:** 
```
locks=# begin;
BEGIN
locks=*# select pg_backend_pid();
 pg_backend_pid 
----------------
           5529
(1 строка)

locks=*# CREATE INDEX ON accounts(acc_no); (Операция подвисла, ждет высвобождение свободных ресурсов)
```
Операция зависла, т.к. ждет ресурсов для выполнения.
Проверяем, на какую блокировку ссылается, выполнив запрос в первой сессии и находим номер блокирующего процесса:

```
SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted FROM pg_locks WHERE pid = 5529;
  locktype  | relation | virtxid | xid |     mode      | granted 
------------+----------+---------+-----+---------------+---------
 virtualxid |          | 6/4     |     | ExclusiveLock | t
 relation   | accounts |         |     | ShareLock     | f (видим что вот блокировка, он ее не может взять, т.к. этот обьект заблокирован 2-ой нашей сессией)
(2 строки)


locks=# SELECT pg_blocking_pids(5529);
 pg_blocking_pids 
------------------
 {4546}

locks=# SELECT * FROM pg_stat_activity WHERE pid = ANY(pg_blocking_pids(5529)) \gx
-[ RECORD 1 ]----+--------------------------------------------------------------------------------------------------------------------------------
datid            | 16674
datname          | locks
pid              | 4546
leader_pid       | 
usesysid         | 10
usename          | postgres
application_name | psql
client_addr      | 
client_hostname  | 
client_port      | -1
backend_start    | 2025-05-13 10:02:03.672241+03
xact_start       | 2025-05-13 10:02:15.934172+03
query_start      | 2025-05-13 11:09:17.040618+03
state_change     | 2025-05-13 11:09:17.040863+03
wait_event_type  | Client
wait_event       | ClientRead
state            | idle in transaction
backend_xid      | 596835
backend_xmin     | 
query_id         | 
query            | SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted FROM pg_locks WHERE pid = 4546;
backend_type     | client backend
```

Как только завершится транзакция во 2-ой сесии, то в 3-ей сессии выполнится транзакция по создании индекса:

```
2-ая сессия:

locks=*# commit;
COMMIT

3-ая сессия:

locks=*# CREATE INDEX ON accounts(acc_no);
CREATE INDEX
locks=*# commit;
COMMIT
```

###3. Взаимоблокировка трех транзакций.

Создадим таблицу в БД locks, и добавим в нее данные:

```
locks=# create table public.zoo (zoo_id integer, zoo_name text, zoo_animal_population integer);

locks=# insert into public.zoo (zoo_id, zoo_name, zoo_animal_population) values (1, 'Moscow_zoo', 1000), (2, 'Germany_zoo', 1000), (3, 'Japanese_zoo', 1000);
INSERT 0 3
```

Запускаю 3 сессии транзакции:

| транзакция 1 | транзакция 2 | транзакция 3 |
|:-------------------|:-------------------|:-------------------|
| begin; | | |
| update public.zoo </br>set zoo_animal_population = zoo_animal_population + 1000 </br>where zoo_id = 1; | | |
| | begin; | |
| | update public.zoo </br>set zoo_animal_population = zoo_animal_population + 1000 </br>where zoo_id = 2; | |
| | | begin; |
| | | update public.zoo </br>set zoo_animal_population = zoo_animal_population + 1000 </br>where zoo_id = 3; |
| update public.zoo </br>set zoo_animal_population = zoo_animal_population - 100 </br>where zoo_id = 2; | | |
| | update public.zoo </br>set zoo_animal_population = zoo_animal_population - 200 </br>where zoo_id = 3; | |
| | | update public.zoo </br>set zoo_animal_population = zoo_animal_population - 300 </br>where zoo_id = 1; |
| | | commit; |
| | commit; | |
| commit; | | |

Получаем ошибку:

```
ERROR:  deadlock detected
DETAIL:  Process 22084 waits for ShareLock on transaction 596839; blocked by process 12357.
Process 12357 waits for ShareLock on transaction 596840; blocked by process 22083.
Process 22083 waits for ShareLock on transaction 596841; blocked by process 22084.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "zoo"
```

### 4. Взаимоблокировка 2-х транзакций, выполняющих UPDATE одной и той же таблицы (без where):

Две транзакции, одновременно выполняющие операцию UPDATE над одной и той же таблицей без использования предложения WHERE, могут столкнуться с взаимоблокировкой. Это происходит, когда одна транзакция обновляет строки в одном порядке, а другая - в противоположном. Такая ситуация, хоть и редкая, может возникнуть, если система управления базами данных (СУБД) выбирает разные планы выполнения запросов для каждой транзакции. Например, одна транзакция может читать таблицу последовательно, а другая - использовать индекс для доступа к строкам.