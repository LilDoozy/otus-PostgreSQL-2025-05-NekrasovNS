# Домашняя работа № 12 | Хранимые функции и процедуры

## Шаг 1. Разворачиваем БД и создаем нужные нам таблицы:

Создаем БД и подключаемся к ней:


```
postgres=# create database trigger_db;
CREATE DATABASE
postgres=# \c trigger_db 
You are now connected to database "trigger_db" as user "postgres".
```

Создаем схему и настраиваем путь:

```
trigger_db=# DROP SCHEMA IF EXISTS pract_functions CASCADE;
NOTICE:  schema "pract_functions" does not exist, skipping
DROP SCHEMA
trigger_db=# CREATE SCHEMA pract_functions;
CREATE SCHEMA

trigger_db=# SET search_path = pract_functions, public;
SET
```

Создаем таблицы для товаров (goods) и для продаж (sales):

```
trigger_db=# CREATE TABLE goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);
INSERT INTO goods (goods_id, good_name, good_price)
VALUES  (1, 'Спички хозайственные', .50),
                (2, 'Автомобиль Ferrari FXX K', 185000000.01);
CREATE TABLE
INSERT 0 2

trigger_db=# select * from goods;
 goods_id |        good_name         |  good_price  
----------+--------------------------+--------------
        1 | Спички хозайственные     |         0.50
        2 | Автомобиль Ferrari FXX K | 185000000.01
(2 rows)
```

```
trigger_db=# CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);

INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);
CREATE TABLE
INSERT 0 4

trigger_db=# select * from sales;
 sales_id | good_id |          sales_time           | sales_qty 
----------+---------+-------------------------------+-----------
        1 |       1 | 2025-06-10 13:24:49.659403+03 |        10
        2 |       1 | 2025-06-10 13:24:49.659403+03 |         1
        3 |       1 | 2025-06-10 13:24:49.659403+03 |       120
        4 |       2 | 2025-06-10 13:24:49.659403+03 |         1
(4 rows)
```

В задании приложен запрос для генерации отчета – сумма продаж по каждому товару:

```
trigger_db=# SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;
        good_name         |     sum      
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)
```

C увеличением объёма данных отчет стал создаваться медленно

Принято решение денормализовать БД, создать таблицу (Ветрина) структура которой повторяет структуру отчета:


```
trigger_db=# CREATE TABLE good_sum_mart
(
        good_name   varchar(63) NOT NULL,
        sum_sale        numeric(16, 2)NOT NULL
);
CREATE TABLE
trigger_db=# \dt
                 List of relations
     Schema      |     Name      | Type  |  Owner   
-----------------+---------------+-------+----------
 pract_functions | good_sum_mart | table | postgres
 pract_functions | goods         | table | postgres
 pract_functions | sales         | table | postgres
(3 rows)
```

## Шаг 2. Задача установить триггер на таблице продаж для поддержания данных в витрине в актуальном состоянии:


Вычисляет сумму продажи и записывает ее в витрину:

```
trigger_db=# delete from good_sum_mart *;
DELETE 0
trigger_db=# insert into good_sum_mart (good_name, sum_sale)
trigger_db-# select G.good_name, sum(G.good_price * S.sales_qty)
trigger_db-# from goods G
trigger_db-# inner join sales S ON S.good_id = G.goods_id
trigger_db-# group by G.good_name;
INSERT 0 2
trigger_db=# select * from good_sum_mart;
        good_name         |   sum_sale   
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)
```

Создаем функцию:

```
trigger_db=# CREATE OR REPLACE FUNCTION trg_db() 
RETURNS trigger
AS
$TRIG_FUNC$ 
BEGIN
DELETE FROM good_sum_mart * ;
INSERT INTO good_sum_mart (good_name, sum_sale) 
SELECT G.good_name, sum(G.good_price * S.sales_qty) 
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id 
GROUP BY G.good_name; 
RETURN NULL;
END;
$TRIG_FUNC$
LANGUAGE plpgsql
;
CREATE FUNCTION
trigger_db=# 
trigger_db=# \df
                            List of functions
     Schema      |  Name  | Result data type | Argument data types | Type 
-----------------+--------+------------------+---------------------+------
 pract_functions | trg_db | trigger          |                     | func
(1 row)
```

И сам триггер:

```
trigger_db=# CREATE TRIGGER trgr_db 
AFTER INSERT OR UPDATE OR DELETE 
ON sales
FOR EACH STATEMENT
EXECUTE FUNCTION trg_db(); 
CREATE TRIGGER
```

## Шаг 3. Смторим результаты выполнения:

Пробуем добавлять строчки, обновлять и удалять, посмотрим как изменится результат в нашей таблице:

Добавляем строчки:

```
trigger_db=# select * from good_sum_mart;
        good_name         |   sum_sale   
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)

trigger_db=# insert into sales (good_id, sales_qty) VALUES (1, 20);
INSERT 0 1
trigger_db=# select * from good_sum_mart;
        good_name         |   sum_sale   
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        75.50
(2 rows)

trigger_db=# insert into sales (good_id, sales_qty) VALUES (1, 40);
INSERT 0 1
trigger_db=# select * from good_sum_mart;
        good_name         |   sum_sale   
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        95.50
(2 rows)
```

Делаю *Update*:

```
trigger_db=# update sales set sales_qty = 100 where sales_id=5;
UPDATE 1
trigger_db=# select * from sales;
 sales_id | good_id |          sales_time           | sales_qty 
----------+---------+-------------------------------+-----------
        1 |       1 | 2025-06-10 13:24:49.659403+03 |        10
        2 |       1 | 2025-06-10 13:24:49.659403+03 |         1
        3 |       1 | 2025-06-10 13:24:49.659403+03 |       120
        4 |       2 | 2025-06-10 13:24:49.659403+03 |         1
        6 |       1 | 2025-06-10 13:56:42.685811+03 |        40
        5 |       1 | 2025-06-10 13:56:08.017479+03 |       100
(6 rows)
```

Пробую *Delete*:

```
trigger_db=# delete from sales where sales_id = 6;
DELETE 1
trigger_db=# select * from good_sum_mart;
        good_name         |   sum_sale   
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |       115.50
(2 rows)

trigger_db=# select * from sales;
 sales_id | good_id |          sales_time           | sales_qty 
----------+---------+-------------------------------+-----------
        1 |       1 | 2025-06-10 13:24:49.659403+03 |        10
        2 |       1 | 2025-06-10 13:24:49.659403+03 |         1
        3 |       1 | 2025-06-10 13:24:49.659403+03 |       120
        4 |       2 | 2025-06-10 13:24:49.659403+03 |         1
        5 |       1 | 2025-06-10 13:56:08.017479+03 |       100
(5 rows)
```

**Триггер по всем командам отработал корректно и штатно**