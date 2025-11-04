# Домашняя работа № 9

**1. Подготовка к работе. Создание структуры таблиц, для которых
выполнялись соединения.**

Создаем таблицу для работ:


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
(2 строки)
```
Заполняем таблицы:

```
joins=# insert into suppliers (supplier_id, supplier_name) values (1000, 'Wildberries');
INSERT 0 1
joins=# insert into suppliers (supplier_id, supplier_name) values (1001, 'OZON');
INSERT 0 1
joins=# insert into suppliers (supplier_id, supplier_name) values (1002, 'YandexMarket');
INSERT 0 1
joins=# insert into suppliers (supplier_id, supplier_name) values (1003, 'MegaMarket');
INSERT 0 1

joins=# select * from suppliers;
 supplier_id | supplier_name 
-------------+---------------
       1000  | Wildberries
       1001  | OZON
       1002  | YandexMarket
       1003  | MegaMarket
(4 строки)

joins=# insert into orders (order_id, supplier_id, order_date) values (50125, 1000, '2025/11/01');
INSERT 0 1
joins=# insert into orders (order_id, supplier_id, order_date) values (50126, 1001, '2025/11/02');
INSERT 0 1
joins=# insert into orders (order_id, supplier_id, order_date) values (50127, 1004, '2025/11/03');
INSERT 0 1

joins=# select * from orders;
 order_id | supplier_id | order_date
----------+-------------+------------
    50125 |        1000 | 2025-11-01
    50126 |        1001 | 2025-11-02
    50127 |        1004 | 2025-11-03
(3 строки)
```


**2. Прямое соединение двух или более таблиц.**

Соединяем 2 таблицы командой *INNER JOIN*:

Запрос соединяет таблицу *suppliers* и *orders* и выдает совпадение по *supplier_id*

```
joins=#  SELECT suppliers.supplier_id, suppliers.supplier_name, orders.order_date
FROM suppliers
INNER JOIN orders
ON suppliers.supplier_id = orders.supplier_id;
 supplier_id | supplier_name | order_date
-------------+---------------+------------
        1000 | Wildberries   | 2025-11-01
        1001 | OZON          | 2025-11-02
(2 строки)
```

**3. Левостороннее соединение двух или более таблиц:**

Выполняем левостороннее соединение 2-ух таблиц командой *LEFT OUTER JOIN*: 

Этот пример с левосторонним соединением вернёт все строки из таблицы *suppliers* и только те строки из таблицы *orders*, в которых совпадают объединённые поля.

Если значение *supplier_id* в таблице поставщиков отсутствует в таблице *orders*, все поля в таблице заказов будут отображаться как пустые *null* в выборке результата.

```
joins=# SELECT suppliers.supplier_id, suppliers.supplier_name, orders.order_date
FROM suppliers
LEFT OUTER JOIN orders
ON suppliers.supplier_id = orders.supplier_id;
 supplier_id | supplier_name | order_date
-------------+---------------+------------
        1000 | Wildberries   | 2025-11-01
        1001 | OZON          | 2025-11-02
        1003 | MegaMarket    |
        1002 | YandexMarket  |
(4 строки)
```

**4. Кросс-соединение 2-ух таблиц.**

Выполняем кросс-соединение 2-ух таблиц командой *CROSS JOIN*:

Cоединим таблицу заказов *suppliers* и таблицу покупателей *orders*

Если в таблице *orders* 3 строки, а в таблице *suppliers* 4 строки, то в результате перекрестного соединения создается 3 * 4 = 12 строк вне зависимости, связаны ли данные строки или нет.

Некоторые из имен столбцов могут даже задублироваться, содержа при этом разные значения в одной и той же записи - поэтому SELECT * не стоит использовать при таком соединении (Использовал для простоты наглядности).

```
joins=# SELECT * FROM suppliers CROSS JOIN orders;
 supplier_id | supplier_name | order_id | supplier_id | order_date
-------------+---------------+----------+-------------+------------
        1000 | Wildberries   |    50125 |        1000 | 2025-11-01
        1001 | OZON          |    50125 |        1000 | 2025-11-01
        1002 | YandexMarket  |    50125 |        1000 | 2025-11-01
        1003 | MegaMarket    |    50125 |        1000 | 2025-11-01
        1000 | Wildberries   |    50126 |        1001 | 2025-11-02
        1001 | OZON          |    50126 |        1001 | 2025-11-02
        1002 | YandexMarket  |    50126 |        1001 | 2025-11-02
        1003 | MegaMarket    |    50126 |        1001 | 2025-11-02
        1000 | Wildberries   |    50127 |        1004 | 2025-11-03
        1001 | OZON          |    50127 |        1004 | 2025-11-03
        1002 | YandexMarket  |    50127 |        1004 | 2025-11-03
        1003 | MegaMarket    |    50127 |        1004 | 2025-11-03
(12 строк)
```

**5. Полное соединение двух или более таблиц:**

Выполняем полное соединение 2-ух таблиц командой *FULL OUTER JOIN*:

Этот пример *FULL OUTER JOIN* вернет все строки из таблицы *suppliers* и все строки из таблицы *orders*, и всякий раз, когда условие соединения не выполняется, *nulls* - пустое значение, будет расширен до этих полей в результирующем наборе.

Если значение supplier_id в таблице *suppliers* отсутствует в таблице *orders*, все поля в таблице *orders* будут отображаться в наборе результатов как *null*. Если значение supplier_id в таблице *orders* отсутствует в таблице *suppliers*, все поля в таблице *suppliers* будут отображаться в наборе результатов как *null*.

```
joins=# SELECT suppliers.supplier_id, suppliers.supplier_name, orders.order_date
FROM suppliers
FULL OUTER JOIN orders
ON suppliers.supplier_id = orders.supplier_id;
 supplier_id | supplier_name | order_date
-------------+---------------+------------
        1000 | Wildberries   | 2025-11-01
        1001 | OZON          | 2025-11-02
             |               | 2025-11-03
        1003 | MegaMarket    |
        1002 | YandexMarket  |
(5 строк)
```

Строки для MegaMarket и YandexMarket будут включены в выборку, поскольку использовалось *FULL OUTER JOIN*. Заметим, что поле order_date для этих записей не содержит значений (либо будет *null*).
Строка для supplier_id 1004 тоже будет включена в выборку, поскольку использовалось *FULL OUTER JOIN*. Заметим, что поля supplier_id и supplier_name для этих записей не содержит значений (либо будет *null*).