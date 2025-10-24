# Логический уровень PostgreSQL

1. Создайте новый кластер PostgresSQL 18
> Будем пользоваться Postgres 18 на виртуальной машине в ЯО, \
> установленный в ДЗ3 и созданным кластером по умолчанию\

3. Зайдите в созданный кластер под пользователем ```postgres```
```shell
postgres@postgres-vm-2-2-20-hdd:~$ psql
psql (18.0 (Ubuntu 18.0-1.pgdg24.04+3))
Type "help" for help.

postgres=#
```
3. Создайте новую базу данных ```testdb```
> postgres=# create database testdb;\
> CREATE DATABASE

4. Зайдите в созданную базу данных под пользователем ```postgres```

```shell
postgres@postgres-vm-2-2-20-hdd:~$ psql -dtestdb
psql (18.0 (Ubuntu 18.0-1.pgdg24.04+3))
Type "help" for help.

testdb=#
```

5. Создайте новую схему ```testnm```
> testdb=# create schema testnm;\
> CREATE SCHEMA

6. Cоздайте новую таблицу ```t1``` с одной колонкой ```c1``` типа ```integer```
> testdb=# create table t1 (c1 integer);\
> CREATE TABLE

7. Вставьте строку со значением ```c1=1```
> testdb=# insert into t1 values (1);\
> INSERT 0 1

8. Создайте новую роль ```readonly```
> testdb=# create role readonly;\
> CREATE ROLE

9. Дайте новой роли право на подключение к базе данных ```testdb```
> Право на подключение к базе данных есть атрибут роли.\
> У роли по умолчанию атрибут login установлен в false.\
> Чтобы его изменить надо использовать не GRANT, а ALTER на самой роли
>                    
> testdb=# alter role readonly login;\
> ALTER ROLE

10. Дайте новой роли право на использование схемы ```testnm```
> testdb=# grant usage on schema testnm to readonly;\
> GRANT

11. Задайте новой роли право на ```select``` для всех таблиц схемы ```testnm```
> testdb=# grant select on all tables in schema testnm to readonly;\
> GRANT

12. Cоздайте пользователя ```testread``` с паролем ```test123```
> testdb=# create user testread with password 'test123';\
> CREATE ROLE

13. Дайте роль ```readonly``` пользователю ```testread```
> testdb=# grant readonly to testread;\
> GRANT ROLE

14. Зайдите под пользователем ```testread``` в базу данных ```testdb```
```shell
postgres@postgres-vm-2-2-20-hdd:~$ psql -hlocalhost -p5432  -dtestdb -Utestread
Password for user testread:
psql (18.0 (Ubuntu 18.0-1.pgdg24.04+3))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off, ALPN: postgresql)
Type "help" for help.

testdb=>
```
15. Сделайте ```select * from t1;```
> testdb=> select * from t1;\
> ERROR:  permission denied for table t1
>
16. Получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)
> Не получилось. Хотя делал сам, не по шпаргалке. Мне нужно совершать ошибки, чтобы лучше запоминать материал

17. Напишите что именно произошло в тексте домашнего задания
> Почему-то у пользователя testread нет доступа к таблице t1
>
18. У вас есть идеи почему? ведь права то дали?
> Нет, идей нет
>
19. Посмотрите на список таблиц
>    testdb-> \d\
>    List of relations\
>    Schema | Name | Type  |  Owner\
>    --------+------+-------+----------\
>    public | t1   | table | postgres\

> А вот теперь есть идея, я создал таблицу ``t1`` в схеме по умолчанию ``public``,\ 
> а доступ пользователю ``testread`` дал к пространству ``testnm``   

20. Подсказка в шпаргалке под пунктом 20
> Подсказка уже не нужна

21.А почему так получилось с таблицей (если делали сами и без шпаргалки то может у вас все нормально)
> При создании таблицы я не указал явно схему и она создалась в схеме по умолчанию ``public``

22. Вернитесь в базу данных ```testdb``` под пользователем ```postgres```
```shell
postgres@postgres-vm-2-2-20-hdd:~$ psql -dtestdb
psql (18.0 (Ubuntu 18.0-1.pgdg24.04+3))
Type "help" for help.

testdb=#
```

23. Удалите таблицу ```t1```
> testdb=# drop table t1;\
> DROP TABLE\
>
> testdb=# \d\
> Did not find any relations.

24. Создайте ее заново но уже с явным указанием имени схемы ```testnm```
> testdb=# create table testnm.t1 (c1 integer);\
> CREATE TABLE

25. Вставьте строку со значением ```c1=1```
> testdb=# insert into testnm.t1 values (1);\
> INSERT 0 1

27. Зайдите под пользователем ```testread``` в базу данных ```testdb```
```shell
postgres@postgres-vm-2-2-20-hdd:~$ psql -hlocalhost -dtestdb -Utestread
Password for user testread:
psql (18.0 (Ubuntu 18.0-1.pgdg24.04+3))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off, ALPN: postgresql)
Type "help" for help.

testdb=>
```
27. Сделайте ```select * from testnm.t1;``` получилось?
> Не получилось, все равно нет привелегий, странно

>testdb=> select * from testnm.t1;\
>ERROR:  permission denied for table t1

28. Есть идеи почему? если нет - смотрите шпаргалку
> Есть идея что когда даем permission на select all tables, это распространяется\
> только на существовавшие на тот момент объект, а таблицу мы с тех пор пересоздали

29. Как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку
> Пробуем идею из п. 28 - пытаемся дать разрешение на таблицу опять

> testdb=> grant select on all tables in schema testnm to readonly;\
> ERROR:  permission denied for table t1

> Не получилось, видимо у этой роли нет прав давать привелегии (?)\
> Пробую подключиться к БД пользователем postgres и выдатать привелегии на select снова

```shell
postgres@postgres-vm-2-2-20-hdd:~$ psql -dtestdb
psql (18.0 (Ubuntu 18.0-1.pgdg24.04+3))
Type "help" for help.
```
> testdb=# grant select on all tables in schema testnm to readonly;\
> GRANT
> А вот теперь получилось выдать привилегии
> Подключаемся обратно пользователем testread и проверяем гипотезу

```shell
postgres@postgres-vm-2-2-20-hdd:~$ psql -hlocalhost -dtestdb -Utestread
Password for user testread:
psql (18.0 (Ubuntu 18.0-1.pgdg24.04+3))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off, ALPN: postgresql)
Type "help" for help.
```
30. Сделайте ```select * from testnm.t1;```

> testdb=> select * from testnm.t1;
> c1
> ----
> 1
> (1 row)

31. Получилось? есть идеи почему? если нет - смотрите шпаргалку
> Да, получилось, значит, гипотеза про выдачу привелегий только объекты, \
> которые существовали на момент выдачи привелегий верна? \
> Проверяю шпаргалку - так и есть, предположение было верным.\
> Тем не менее, так же ценно знать про команду 
> ``ALTER default privileges in SCHEMA testnm grant SELECT on TABLES to readonly;``,
> из шпаргалки, которая влючает выдачу привилегий по умолчанию для новых таблиц

32. Сделайте ```select * from testnm.t1;``` получилось? ура!
> Уже получилось :)

33. Теперь попробуйте выполнить команду ```create table t2(c1 integer); insert into t2 values (2);```
> testdb=> create table t2(c1 integer); insert into t2 values (2);\
> ERROR:  permission denied for schema public\
> LINE 1: create table t2(c1 integer);\
> ^\
> ERROR:  relation "t2" does not exist\
> LINE 1: insert into t2 values (2);

34. А как так? нам же никто прав на создание таблиц и ```insert``` в них под ролью ```readonly```?
есть идеи как убрать эти права? если нет - смотрите шпаргалку
> У меня все в порядке, создавать таблицу нет привилегий. Посмотрю шпаргалку в чем разница\
> между моими действиями и рекомендованными.
> 
> Почему-то у пользователя testread нет прав на создание таблицы в схеме public\
> В шпаргалке сказано про то, что в 15 PostgreSQL отозваны права на создание таблиц
> у схемы public по умолчанию. Видимо, дело в этом.

35. Если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды
> Видимо, подразумевалось, что можно явно отобрать права на создание таблиц у public, но у меня уже так

36. Теперь попробуйте выполнить команду ```create table t3(c1 integer); insert into t3 values (3);```
расскажите что получилось и почему
> Видимо, подразумевалось что теперь при попытке создания таблицы в схеме public будет ошибка,\
> которую я получил еще в пункте 33