# Домашняя работа № 9

**Шаг 1. Подготовка к работе и структура таблиц, для которых
выполнялись соединения:**

Для начала создал таблицу для работ:


```
postgres=# create database joins;
CREATE DATABASE
```

Создал 2 таблицы и наполнил их данными:

```
joins=# CREATE TABLE suppliers(                                    
  supplier_id INTEGER PRIMARY KEY,
  supplier_name TEXT NOT NULL
);
CREATE TABLE

joins=# create table orders (
  order_id INTEGER NOT NULL,
  supplier_id INTEGER NOT NULL,
  order_date DATE NOT NULL
);
CREATE TABLE

joins=# \dt;
           List of relations
 Schema |   Name    | Type  |  Owner   
--------+-----------+-------+----------
 public | orders    | table | postgres
 public | suppliers | table | postgres
(2 rows)
```
Наполняю данными:

```
joins=# insert into suppliers (supplier_id, supplier_name) values (10000, 'IBM');
INSERT 0 1
joins=# insert into suppliers (supplier_id, supplier_name) values (10001, 'HP');
INSERT 0 1
joins=# insert into suppliers (supplier_id, supplier_name) values (10002, 'Microsoft');
INSERT 0 1
joins=# insert into suppliers (supplier_id, supplier_name) values (10003, 'NVIDIA');
INSERT 0 1

joins=# select * from suppliers;
 supplier_id | supplier_name 
-------------+---------------
       10000 | IBM
       10001 | HP
       10002 | Microsoft
       10003 | NVIDIA
(4 rows)

joins=# insert into orders (order_id, supplier_id, order_date) values (500125, 10000, '2013/05/12');
INSERT 0 1
joins=# insert into orders (order_id, supplier_id, order_date) values (500126, 10001, '2013/05/13');
INSERT 0 1
joins=# insert into orders (order_id, supplier_id, order_date) values (500127, 10004, '2013/05/14');
INSERT 0 1

joins=# select * from orders;
 order_id | supplier_id | order_date 
----------+-------------+------------
   500125 |       10000 | 2013-05-12
   500126 |       10001 | 2013-05-13
   500127 |       10004 | 2013-05-14
(3 rows)
```


**Шаг 2. Прямое соединение двух или более таблиц:**

Выполняю соединение 2-ух таблиц командой *INNER JOIN*:

Запрос просто соединяет таблицу *suppliers* и *orders* и выдает совпадение по колонке *supplier_id*

```
joins=# SELECT suppliers.supplier_id, suppliers.supplier_name, orders.order_date
FROM suppliers
INNER JOIN orders
ON suppliers.supplier_id = orders.supplier_id;
 supplier_id | supplier_name | order_date 
-------------+---------------+------------
       10000 | IBM           | 2013-05-12
       10001 | HP            | 2013-05-13
(2 rows)
```

**Шаг 3. Левостороннее соединение двух или более таблиц:**

Выполняю левосторонее соединение 2-ух таблиц командой *LEFT OUTER JOIN*: 

Этот пример с левосторонним соединением вернёт все строки из таблицы *suppliers* и только те строки из таблицы *orders*, в которых совпадают объединённые поля.

Если значение *supplier_id* в таблице поставщиков отсутствует в таблице *orders*, все поля в таблице заказов будут отображаться как пустые *null* в выборке результата.

```
joins=# SELECT suppliers.supplier_id, suppliers.supplier_name, orders.order_date
FROM suppliers
LEFT OUTER JOIN orders
ON suppliers.supplier_id = orders.supplier_id;
 supplier_id | supplier_name | order_date 
-------------+---------------+------------
       10000 | IBM           | 2013-05-12
       10001 | HP            | 2013-05-13
       10003 | NVIDIA        | null
       10002 | Microsoft     | null
(4 rows)
```

**Шаг 4. Кросс соединение 2-ух таблиц:**

Выполняю кросс соединение 2-ух таблиц командой *CROSS JOIN*:

Cоединим таблицу заказов *suppliers* и таблицу покупателей *orders*

Если в таблице *orders* 3 строки, а в таблице *suppliers* 4 строки, то в результате перекрестного соединения создается 3 * 4 = 12 строк вне зависимости, связаны ли данные строки или нет.

Некоторые из имен столбцов могут даже задублироваться, содержа при этом разные значения в одной и той же записи - поэтому SELECT * не стоит использовать при таком соединении (Использовал для простоты наглядности).

```
joins=# SELECT * FROM suppliers 
joins-# CROSS JOIN
joins-# orders;
 supplier_id | supplier_name | order_id | supplier_id | order_date 
-------------+---------------+----------+-------------+------------
       10000 | IBM           |   500125 |       10000 | 2013-05-12
       10001 | HP            |   500125 |       10000 | 2013-05-12
       10002 | Microsoft     |   500125 |       10000 | 2013-05-12
       10003 | NVIDIA        |   500125 |       10000 | 2013-05-12
       10000 | IBM           |   500126 |       10001 | 2013-05-13
       10001 | HP            |   500126 |       10001 | 2013-05-13
       10002 | Microsoft     |   500126 |       10001 | 2013-05-13
       10003 | NVIDIA        |   500126 |       10001 | 2013-05-13
       10000 | IBM           |   500127 |       10004 | 2013-05-14
       10001 | HP            |   500127 |       10004 | 2013-05-14
       10002 | Microsoft     |   500127 |       10004 | 2013-05-14
       10003 | NVIDIA        |   500127 |       10004 | 2013-05-14
(12 rows)

```

**Шаг 5. Полное соединение двух или более таблиц:**

Выполняю полное соединение 2-ух таблиц командой *FULL OUTER JOIN*:

Этот пример *FULL OUTER JOIN* вернет все строки из таблицы *suppliers* и все строки из таблицы *orders*, и всякий раз, когда условие соединения не выполняется, *nulls* - пустое значение, будет расширен до этих полей в результирующем наборе.

Если значение supplier_id в таблице *suppliers* отсутствует в таблице *orders*, все поля в таблице *orders* будут отображаться в наборе результатов как *null*. Если значение supplier_id в таблице *orders* отсутствует в таблице *suppliers*, все поля в таблице *suppliers* будут отображаться в наборе результатов как *null*.

```
joins=# SELECT suppliers.supplier_id, suppliers.supplier_name, orders.order_date
FROM suppliers
FULL OUTER JOIN orders
ON suppliers.supplier_id = orders.supplier_id;
 supplier_id | supplier_name | order_date 
-------------+---------------+------------
       10000 | IBM           | 2013-05-12
       10001 | HP            | 2013-05-13
       10002 | Microsoft     | null
       10003 | NVIDIA        | null
       null  | null          | 2013-05-14
(5 rows)

```

Строки для Microsoft и NVIDIA будут включены в выборку, поскольку использовалось *FULL OUTER JOIN*. Однако можно заметить, что поле order_date для этих записей содержит значение *null*.

Строка для supplier_id 10004 тоже будет включена в выборку, поскольку использовалось *FULL OUTER JOIN*. Однако можно заметить, что поля supplier_id и supplier_name для этих записей содержат значение *null*.