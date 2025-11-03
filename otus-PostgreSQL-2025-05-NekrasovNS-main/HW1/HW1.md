# Домашняя работа №1

**Шаг 1. Развренул БД PostgresSQL на виртуальной машине на платформе LINUX OS UBUNTU (VirtualBox)**

**Шаг 2. Создал таблицу, добавил данные:**

```
create table persons(id serial, first_name text, second_name text);

insert into persons(first_name, second_name) values('ivan', 'ivanov');

insert into persons(first_name, second_name) values('petr', 'petrov');

commit;
```

**Шаг 3. посмотреть текущий уровень изоляции:**
```
postgres=# show transaction isolation level;
 transaction_isolation
 read committed
(1 row)
```
READ COMMITTED
На этом уровне транзакция может читать только те изменения в других параллельных транзакциях, которые уже были закоммичены. Это нас спасает от грязного чтения, но не спасает от неповторяющегося чтения и от фантомного чтения.

**Шаг 4. Добавил в первой сессии запись:**

```
insert into persons(first_name, second_name) values('sergey', 'sergeev');
```
Сделал запрос в второй сессии:

```
select from persons;
```
Результат: в второй сессии запись не появилась, так как транзакция не подтверждена (Не сделан commit)

```
postgres=*#  select from persons;
--
(2 rows)
```
**Шаг 5. Подтверждаю изменения в первой сесии транзакции командой Commit:**

```
postgres=*# comm
comment  commit   
postgres=*# commit;
COMMIT
```
Делаю Select в второй сессии:

```
postgres=*# select from persons;
--
(3 rows)
```
Вижу что значение добавилось, стало больше строк, так как в первой сессии мы подтвердили изменения командой commit.

**Шаг 6. Начинаю новые транзакции в сессиях, но уже в режиме repeatable read:**

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

Добавляю новую запись в первой сесии:

```
postgres=*# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
```

Выполнил select в второй сессии:

```
postgres=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

Вижу что новая запись не добавилась, пока в первой и в второй сессиях мы не выполним команду commit, изменения не будут зафиксированы.

После выполнения команды commit в первой и второй сессиях мы получим результат добавления записи:

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

