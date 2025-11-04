# Домашняя работа № 4

**1. Создал новый кластер PostgresSQL 16 на VM под ОС Linux Ubuntu 24.04, открыл терминал, зашел под пользователем postgres в PSQL:**

```
postgres@pgdbvip:~$ psql
psql (16.10 (Ubuntu 16.10-1.pgdg22.04+1))
Введите "help", чтобы получить справку.

postgres=#

```

Создаем тестовую БД ```testdb```:

```
postgres=# create database testdb;
CREATE DATABASE
```
Подключаемся к созданной БД:

```
postgres=# \c testdb;
Вы подключены к базе данных "testdb" как пользователь "postgres".
```

**2. Создаем новую схему ```testnm``` и новую таблицу, заполним таблицу значениями и создадим новую роль:**

Создаю новую схему ```testnm```:

```
testdb=# create schema testnm;
CREATE SCHEMA
```
Поскольку по умолчанию search_path будет $user и public, сделаем:
```
postgres=# ALTER DATABASE "testdb" set search_path to "$user", testnm;
ALTER DATABASE
```

Cоздаю новую таблицу ```t1``` с одной колонкой ```c1``` типа ```integer```:

```
testdb=# create table t1 (c1 integer);
CREATE TABLE
```

Вставляем строку со значением ```c1=1```:

```
testdb=# insert INTO t1 values (1);
INSERT 0 1

```

Создаем новую роль ```readonly```:

```
testdb=# create role readonly;
CREATE ROLE
```

Предоставляем новой роли право на подключение к БД, на использование схемы и ```SELECT```-a для всех таблиц:

Доступ к подключению к БД ```testdb```:

```
testdb=# grant connect on database testdb to readonly;
GRANT
```

Доступ на использование схемы ```testnm```:

```
testdb=# grant usage on schema testnm to readonly;
GRANT
```

SELECT для все таблиц:

```
testdb=# grant select on all tables in schema testnm to readonly;
GRANT
```

**3. Работа над созданием нового пользователя и добавление его в созданную роль, а также выполнения ряда запросов от нового пользователя:**

Создание пользователя:

```
testdb=# create user testread with password 'test123';
CREATE ROLE
```

Добавление созданому пользователю роли readonly:

```
testdb=# grant readonly to testread;
GRANT ROLE
```
Попытка сделать запрос к таблице t1:
 
```
psql -U testread -d testdb
```

Запрос выплнен успешно, т.к. я заранее задал search_path 
```
testdb=> select * from t1;
 c1
----
  1
(1 строка)

testdb=> \dt
         Список отношений
 Схема  | Имя |   Тип   | Владелец
--------+-----+---------+----------
 testnm | t1  | таблица | postgres
(1 строка)

```

Удаляю таблицу t1 и создаю ее заново но уже с явным указанием схемы :

```
testdb=# drop table t1;
DROP TABLE
```

Создаем заново с указанием названия схемы и наполняем данными:

```
testdb=# create table testnm.t1(c1 integer);
CREATE TABLE
```
```
testdb=# insert into testnm.t1 values (1);
INSERT 0 1
```
Выполняем select:

```
testdb=# testdb=> select * from testnm.t1;
ОШИБКА:  нет доступа к таблице t1

```
Это происходит из-за того, что гранты я выдавал на ранее созданную схему. Из-за того, что я ее пересоздал, необходимо заново выдать гранты на селект для данной таблицы.
Чтобы не столкнуться с этим в дальнейшем, можно сделать alter default privileges.
```
testdb=# alter default privileges in schema testnm grant select on tables to readonly;
ALTER DEFAULT PRIVILEGES

```
И проверяем:
```
testdb=# select * from testnm.t1;
 c1
----
  1
(1 строка)
```

**4. Создание таблиц и попытка отобрать права у пользователя на создания таблиц:**

```
Потому что я подключен к БД под пользователем postgres, который имеет атрибуты:
```
testdb=# \du postgres
                                    Список ролей
 Имя роли |                                Атрибуты
----------+-------------------------------------------------------------------------
 postgres | Суперпользователь, Создаёт роли, Создаёт БД, Репликация, Пропускать RLS
```
Пробуем из под пользователя testread проверить создание таблицы и наполнить ее данными:

```
testdb=> create table t3(c1 integer); insert into t2 values (2);
ERROR:  permission denied for schema public
LINE 1: create table t3(c1 integer);
                     ^
ERROR:  permission denied for table t2
```

Прав пользователю testread на CREATE и INSERT в схеме testnm я не выдавал.

