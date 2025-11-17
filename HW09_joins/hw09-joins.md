0. Создаем новую ВМ с параметрами как в предыдущем задании и заново ставим Postgres

```shell
# создаем ВМ через cli Я.Облако
yc compute instance create \
  --name otus-postgres-hw09-1 \
  --hostname otus-postgres-hw09-1 \
  --cores 2 \
  --core-fraction 50 \
  --memory 2 \
  --create-boot-disk size=10G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2404-lts \
  --network-interface subnet-name=default-ru-central1-b,nat-ip-version=ipv4 \
  --ssh-key ~/.ssh/ssh-key-postgres.pub
```

```shell
# Ставим Postgres, пользуясь apt
yc-user@otus-postgres-hw09-1:~$ sudo apt update && sudo apt upgrade -y -q && \
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && \
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && \
sudo apt-get update && \
sudo apt -y install postgresql

# Проверяем что кластер запущен
yc-user@otus-postgres-hw09-1:~$ sudo pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 online postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```

К работе приложить структуру таблиц, для которых выполнялись соединения
> Сделаем минимально необходимую схему для демонстрации разных типов соединений,
> состоящую из двух таблиц с одинаковой структурой (id integer, value_left(right) character (1)) и 
> заполненной данными, удобными для наглядной демонстрации результатов запросов

```
postgres=# create database joins_demo;
CREATE DATABASE
postgres=# \c joins_demo
You are now connected to database "joins_demo" as user "postgres".
joins_demo=# create table a (id integer PRIMARY KEY, value_left character (1));
CREATE TABLE
joins_demo=# create table b (id integer PRIMARY KEY, value_right character (1));
CREATE TABLE
joins_demo=# insert into a (id, value_left) values (1, 'A'), (2, 'B'), (3, 'C'), (4, 'D');
INSERT 0 4
joins_demo=# insert into b (id, value_right) values (2, 'E'), (3, 'F'), (4, 'G'), (5, 'H');
INSERT 0 4
```
 
Необходимо:
1. Реализовать прямое соединение двух или более таблиц

> В результат попали только записи, ключи  ```id``` которых есть и в таблице ```a``` и в таблице ```b```
```
joins_demo=# select 
    a.id as a_id, 
    b.id as b_id, 
    value_left, 
    value_right 
from 
    a inner join b using (id);
    
 a_id | b_id | value_left | value_right
------+------+------------+-------------
    2 |    2 | B          | E
    3 |    3 | C          | F
    4 |    4 | D          | G
(3 rows)
```

2. Реализовать левостороннее (или правостороннее) соединение двух или более таблиц

> Левостороннее соединение.\
> В результат попали записи, все записи в таблице ```a``` и соответствующие им по ключу ```id``` в таблице ```b```
```
joins_demo=# select 
    a.id as a_id, 
    b.id as b_id, 
    value_left, 
    value_right 
from 
    a left join b using (id);
    
 a_id | b_id | value_left | value_right
------+------+------------+-------------
    1 |      | A          |
    2 |    2 | B          | E
    3 |    3 | C          | F
    4 |    4 | D          | G
(4 rows)
```

> Правостороннее соединение.\
> В результат попали записи, все записи в таблице ```b``` и соответствующие им по ключу ```id``` в таблице ```a```
```
joins_demo=# select 
    a.id as a_id, 
    b.id as b_id, 
    value_left, 
    value_right 
from 
    a right join b using (id);
    
 a_id | b_id | value_left | value_right
------+------+------------+-------------
    2 |    2 | B          | E
    3 |    3 | C          | F
    4 |    4 | D          | G
      |    5 |            | H
(4 rows)
```

3. Реализовать кросс соединение двух или более таблиц

> В результат попало декартово произведение множеств записей из таблиц ```a``` и ```b```
```
joins_demo=# select 
    a.id as a_id, 
    b.id as b_id, 
    value_left, 
    value_right 
from 
    a cross join b;
    
 a_id | b_id | value_left | value_right
------+------+------------+-------------
    1 |    2 | A          | E
    1 |    3 | A          | F
    1 |    4 | A          | G
    1 |    5 | A          | H
    2 |    2 | B          | E
    2 |    3 | B          | F
    2 |    4 | B          | G
    2 |    5 | B          | H
    3 |    2 | C          | E
    3 |    3 | C          | F
    3 |    4 | C          | G
    3 |    5 | C          | H
    4 |    2 | D          | E
    4 |    3 | D          | F
    4 |    4 | D          | G
    4 |    5 | D          | H
(16 rows)
```

4. Реализовать полное соединение двух или более таблиц

> В результат попали все записи из таблиц ```a``` и ```b```\
> Записи, для которых нет соответствующих им по ключу ```id``` записей в другой таблице содержат только поля из таблицы, где данные для этого ключа есть

```
joins_demo=# select 
    a.id as a_id, 
    b.id as b_id, 
    value_left, 
    value_right 
from 
    a full join b using (id);
    
 a_id | b_id | value_left | value_right
------+------+------------+-------------
    1 |      | A          |
    2 |    2 | B          | E
    3 |    3 | C          | F
    4 |    4 | D          | G
      |    5 |            | H
(5 rows)
```

5. Реализовать запрос, в котором будут использованы разные типы соединений. 
> Добавим третью таблицу, чтобы было интереснее
```
joins_demo=# create table c (id integer PRIMARY KEY, value_extra character (1));
CREATE TABLE
joins_demo=# insert into c (id, value_extra) values (1, 'i'), (2, 'j'), (3, 'k'), (4, 'l'), (5, 'm');
INSERT 0 5
```

> 5.1 Соединение вида ```a left b inner c``` с иcпользованием ```using```


> В результат попали:
> - все записи в таблице ```a```
> - записи в таблице ```b``` соответствующие по ключу ```id``` из таблицы ```a```
> - записи в таблице ```c``` соответствующие по ключу ```id``` из таблицы ```a```\
> Важный момент. При этом запись с ключом 1 из таблицы ```c``` тоже попала в результат

```
joins_demo=# select 
    a.id as a_id, 
    b.id as b_id, 
    c.id as c_id,  
    value_left, 
    value_right, 
    value_extra 
from 
    a left join b using (id) 
    inner join c using (id);

a_id | b_id | c_id | value_left | value_right | value_extra
------+------+------+------------+-------------+-------------
1 |      |    1 | A          |             | i
2 |    2 |    2 | B          | E           | j
3 |    3 |    3 | C          | F           | k
4 |    4 |    4 | D          | G           | l
(4 rows)
```

> 5.2 Соединение вида ```a left b inner c``` с иcпользованием ```on```  

> В результат попали только записи, ключи которых есть в каждой из таблиц ```a```, ```b``` и ```c```\
> Важный момент. В отличие от предыдущего варианта запроса с ```using``` теперь более жесткое ограничение внутреннего соединения ограничило результат только записями, ключи которых есть во всех трех таблицах 
```
joins_demo=# select 
    a.id as a_id, 
    b.id as b_id, 
    c.id as c_id, 
    value_left, 
    value_right, 
    value_extra 
from 
    a left join b on a.id = b.id 
    inner join c on b.id = c.id;
    
a_id | b_id | c_id | value_left | value_right | value_extra
------+------+------+------------+-------------+-------------
2 |    2 |    2 | B          | E           | j
3 |    3 |    3 | C          | F           | k
4 |    4 |    4 | D          | G           | l
(3 rows)
```

> 5.3 Соединение вида ```a right b inner c``` с иcпользованием ```using```

> В результат попали:
> - все записи в таблице ```b```
> - записи в таблице ```a``` соответствующие по ключу ```id``` из таблицы ```b```
> - записи в таблице ```c``` соответствующие по ключу ```id``` из таблицы ```b```

```
joins_demo=# select 
    a.id as a_id, 
    b.id as b_id, 
    c.id as c_id,  
    value_left, 
    value_right, 
    value_extra 
from 
    a right join b using (id) 
    inner join c using (id);
    
 a_id | b_id | c_id | value_left | value_right | value_extra
------+------+------+------------+-------------+-------------
    2 |    2 |    2 | B          | E           | j
    3 |    3 |    3 | C          | F           | k
    4 |    4 |    4 | D          | G           | l
      |    5 |    5 |            | H           | m
(4 rows)
```

> 5.4 Соединение вида ```a right b inner c``` с иcпользованием ```on```

> В результат попали:
> - все записи в таблице ```b```
> - записи в таблице ```a``` соответствующие по ключу ```id``` из таблицы ```b```  
> - записи в таблице ```c``` соответствующие по ключу ```id``` из таблицы ```b```\
> Важный момент. В случае правого соединения между таблицами ```a``` и ```b``` результат оказался ожидаемо идентичным для связки через ```on``` и через ```using``` 
```
joins_demo=# select 
    a.id as a_id, 
    b.id as b_id, 
    c.id as c_id,  
    value_left, 
    value_right, 
    value_extra 
from 
    a right join b using (id) 
    inner join c using (id);
    
 a_id | b_id | c_id | value_left | value_right | value_extra
------+------+------+------------+-------------+-------------
    2 |    2 |    2 | B          | E           | j
    3 |    3 |    3 | C          | F           | k
    4 |    4 |    4 | D          | G           | l
      |    5 |    5 |            | H           | m
(4 rows)
```

> 5.5 Соединение вида ```a cross b inner c``` с иcпользованием ```using``` и ```on```

> В результат попали:
> - декартово произведение множеств записей из таблиц ```a``` и ```b```
> - записи из таблицы ```c``` соответвствующие по ключу ```id``` в таблице ```b``` 
```
joins_demo=# select 
    a.id as a_id, 
    b.id as b_id, 
    c.id as c_id, 
    value_left, 
    value_right, 
    value_extra 
from 
    a cross join b 
    inner join c on b.id = c.id;
    
 a_id | b_id | c_id | value_left | value_right | value_extra
------+------+------+------------+-------------+-------------
    1 |    2 |    2 | A          | E           | j
    2 |    2 |    2 | B          | E           | j
    3 |    2 |    2 | C          | E           | j
    4 |    2 |    2 | D          | E           | j
    1 |    3 |    3 | A          | F           | k
    2 |    3 |    3 | B          | F           | k
    3 |    3 |    3 | C          | F           | k
    4 |    3 |    3 | D          | F           | k
    1 |    4 |    4 | A          | G           | l
    2 |    4 |    4 | B          | G           | l
    3 |    4 |    4 | C          | G           | l
    4 |    4 |    4 | D          | G           | l
    1 |    5 |    5 | A          | H           | m
    2 |    5 |    5 | B          | H           | m
    3 |    5 |    5 | C          | H           | m
    4 |    5 |    5 | D          | H           | m
(16 rows)
```