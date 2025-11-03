# Домашняя работа № 4

**Шаг 1. Создал новый кластер PostgresSQL 16 на виртаульной машины под ОС Linux Ubuntu 24.04, открыл терминал, зашел под пользователем postgres в консльну утилиту PSQL:**

```
postgres@studentPC:~$ psql 
psql (16.8 (Ubuntu 16.8-0ubuntu0.24.04.1))
Type "help" for help.

postgres=#
```

Создал тестовую БД ```testdb```:

```
postgres=# create database testdb;
CREATE DATABASE
```
Подключился к созданной БД:

```
postgres=# \c testdb;
You are now connected to database "testdb" as user "postgres".
```

**Шаг 2. Создаю новую схему ```testnm``` и новую таблицу, приступаю к наполнению их значениями и создание новой роли:**

Создаю новую схему ```testnm```:

```
testdb=# create schema testnm;
CREATE SCHEMA
```

Cоздаю новую таблицу ```t1``` с одной колонкой ```c1``` типа ```integer```:

```
testdb=# create table t1 (c1 integer);
CREATE TABLE
```

Подставляю строку со значением ```c1=1```:

```
testdb=# insert INTO t1 values (1);
INSERT 0 1
```

Создаю новую роль ```readonly```:

```
testdb=# create role readonly;
CREATE ROLE
```

Предоставляю новой роли право на подключение к БД, на использование схемы и ```SELECT```-a для всех таблиц:

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

**Шаг 3. Работа над созданием нового пользователя и добавление его в созданную роль, а также выполнения ряда запросов от нового пользователя:**

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

Произошла ошибка, отказ в доступе, так как таблица t1 была создана в схеме public, у роли readonly нет прав на схему public

```
estdb=> select * from t1;
ERROR:  permission denied for table t1
testdb=> \dt 
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)
```

Подключился обратно к БД под пользователем postgres, удаляю таблицу t1 и создаю ее заново но уже с явным указанием схемы :

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
При попытке сделать селект к заново созданнйо таблице произошла ошибка:

так получилось потому что, когда мы выдавали grant на селект к таблице он выдался к той таблице которая существовала на тот момент времени, к свеже созданной таблице нужно снова применить grant на селект, либо же применить ALTER deafult (Это уже к всем новым таблицам):

```
testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
```

```
testdb=# alter default privileges in schema testnm grant select on tables to readonly;
ALTER DEFAULT PRIVILEGES
```
Делаем тот же запрос:

```
testdb=# select * from testnm.t1;
 c1 
----
  1
(1 row)
```

**Шаг 4. Создание таблиц и попытка отобрать права у пользователя на создания таблиц:**

Создаем таблицу и наполняем ее данными:

```
testdb=# create table t2(c1 integer); insert into t2 values (2);
CREATE TABLE
INSERT 0 1
```
Роль public добавлется всем новым пользователям по дефолту, то есть каждый новый пользователь имеет право по умолчанию создавать объекты в схеме public и в любой БД, если есть право на коннект к конкретной БД.

Пробуем отобрать у роли public право на создание объектов:

```
testdb=# revoke create on schema public from public;
REVOKE
testdb=# revoke all on database testdb from public;
REVOKE
```
Пробуем из под созданного пользователя затестить создание таблицы и наполнить ее данными:

```
testdb=> create table t3(c1 integer); insert into t2 values (2);
ERROR:  permission denied for schema public
LINE 1: create table t3(c1 integer);
                     ^
ERROR:  permission denied for table t2
```

Как видим пермишины действуют, удалось выполнить этот пункт.


