# Домашняя работа №1

**1. Развренул БД PostgresSQL на виртуальной машине на платформе ОС Linux Ubuntu 24.04 (VirtualBox)**

**2. Создал таблицу, добавил данные:**

```
create table persons(id serial, first_name text, second_name text);

insert into persons(first_name, second_name) values('ivan', 'ivanov');

insert into persons(first_name, second_name) values('petr', 'petrov');

commit;
```

**3. Текущий уровень изоляции:**
```
postgres=# show transaction isolation level;
 transaction_isolation
 read committed
(1 row)
```
READ COMMITTED
На этом уровне транзакция может читать только те изменения в других параллельных транзакциях, которые уже были закоммичены. Это нас спасает от грязного чтения, но не спасает от неповторяющегося чтения и от фантомного чтения.

**4. Добавил в первой сессии запись:**

```
insert into persons(first_name, second_name) values('sergey', 'sergeev');
```
Запрос в параллельной сессии:

```
select from persons;
```
Результат: в параллельной сессии запись не появилась, не сделан commit .

```
postgres=*#  select from persons;
--
(2 rows)
```
**5. Commit в первой сессии:**

```
postgres=*# commit;
COMMIT
```
Выполняю select в параллельной сессии:

```
postgres=*# select from persons;
--
(3 rows)
```
Значение добавилось, стало больше строк, так как в первой сессии мы сделали commit.

**6. Генерирую новые транзакции в сессиях, но уже в режиме repeatable read:**

```
postgres=# set transaction isolation level repeatable read;
SET

postgres=*# show transaction isolation level;
 transaction_isolation 
-----------------------
 repeatable read
(1 row)
```
REPEATABLE READ
Этот уровень означает, что пока транзакция не завершится, никто параллельно не может изменять или удалять строки, которые транзакция уже прочитала.

Это нас спасает и от грязного чтения, и от неповторяющегося чтения, но всё ещё мы не решаем проблему фантомного чтения.

Insert в первой сессии:

```
postgres=*# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
```

Select в параллельной сессии:

```
postgres=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

Пока в первой и в параллельной сессиях не выполню команду commit, изменения не будут зафиксированы.

После выполнения команды commit в первой и параллельной сессиях получил результат добавления записи:

```
postgres=*# select* from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```

