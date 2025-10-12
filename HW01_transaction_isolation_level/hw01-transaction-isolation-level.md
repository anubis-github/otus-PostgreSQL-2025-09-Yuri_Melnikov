# Работа с уровнями изоляции транзакции в PostgreSQL

1. Cоздать новый проект в Яндекс облако или на любых ВМ, докере

> Создан проект otus-postgre-2025-09 в UI консоли Я.Облако

2. Создать ВМ с дефолтными параметрами

> ВМ cоздана в UI консоли Я.Облако с параметрами 2CPU, 2Gb RAM, 20Gb HDD

3. Добавить свой ssh ключ в metadata ВМ

> Специально сгенерированный для Я.Облако ключ добавлен в metadata ВМ в консоли Я.Облако 
   
4. Зайти удаленным ssh (первая сессия), не забывайте про ssh-add

> ❯ ❯ ssh-add ~/.ssh/ssh-key-postgres\
> Identity added: /Users/vinnipooh/.ssh/ssh-key-postgres (/Users/vinnipooh/.ssh/ssh-key-postgres)

> ❯ ssh -i ~/.ssh/ssh-key-postgres postgres@84.201.168.156

5. Поставить PostgreSQL

> ``#`` Буду использовать Postgres в Docker, настроенный в рамках HM02\
> postgres@postgres-vm-2-2-20-hdd:~$ sudo docker start pg-docker\
> pg-docker\
> 
> postgres@postgres-vm-2-2-20-hdd:~$ sudo docker ps\
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS         PORTS                                         NAMES
7a91fe960829   postgres:17   "docker-entrypoint.s…"   58 minutes ago   Up 2 minutes   0.0.0.0:5432->5432/tcp, [::]:5432->5432/tcp   pg-docker

6. Зайти вторым ssh (вторая сессия).

> ssh -i ~/.ssh/ssh-key-postgres postgres@84.201.155.126\
>   Welcome to Ubuntu 24.04.3 LTS (GNU/Linux 6.8.0-85-generic x86_64)\
>   ...\
> Last login: Sun Oct 12 19:10:02 2025 from 93.100.77.210\

7. Запустить везде psql из под пользователя postgres. Буду использовать для этих целей еще 2 контейнера с именами pg-client-1 и pg-client-2 в новых сессиях

> ``#`` Запуск контейнера pg-client-1\
> postgres@postgres-vm-2-2-20-hdd:~$ sudo docker run -it --rm --network pg-net --name pg-client-1 postgres:17 psql -h pg-docker -U postgres\
> Password for user postgres:\
> psql (17.6 (Debian 17.6-2.pgdg13+1))\
>Type "help" for help.\
>
>postgres=#

> ``#`` Запуск контейнера pg-client-2\
> postgres@postgres-vm-2-2-20-hdd:~$ sudo docker run -it --rm --network pg-net --name pg-client-2 postgres:17 psql -h pg-docker -U postgres\
> Password for user postgres:\
> psql (17.6 (Debian 17.6-2.pgdg13+1))\
>Type "help" for help.\
>
>postgres=#

8. Выключить auto commit

> Autocommit опция клиента psql. Можно его не выключать, если явно стартовать и заканчивать транзакции, как в этом задании

9. В первой сессии создать новую таблицу и наполнить ее данными

> postgres1=# create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov');
CREATE TABLE
INSERT 0 1
INSERT 0 1

10. Посмотреть текущий уровень изоляции: `show transaction isolation level`

> postgres1=# show transaction isolation level;\
> transaction_isolation\
> -----------------------\
> read committed\
> (1 row)

11. Начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции

> postgres1=# begin;\
> BEGIN

> postgres2=# begin;\
> BEGIN

12. В первой сессии добавить новую запись `insert into persons(first_name, second_name) values('sergey', 'sergeev');`

> postgres1=*# insert into persons(first_name, second_name) values('sergey', 'sergeev');\
> INSERT 0 1

14. Сделать `select * from persons` во второй сессии. Видите ли вы новую запись и если да то почему?

> ``#`` Новая запись не видна во второй сессии так как в первой мы не завершили транзакцию, 
> а при уровне изоляции read committed в текущей транзакции будут видны записи только успешно завершенных других транзакций\

> postgres2=# select * from persons;\
> id | first_name | second_name\
> ----+------------+-------------\
> 1 | ivan       | ivanov\
> 2 | petr       | petrov\
> (2 rows)

15. Завершить первую транзакцию: `commit;`

> postgres1=*# commit;\
> COMMIT\

16. Cделать `select * from persons;` во второй сессии. Видите ли вы новую запись и если да то почему?

> ``#`` Новая запись видна во второй сессии так как в первой мы завершили транзакцию.
> И, хотя во второй сессии транзакция не завершена, при уровне изоляции read committed 
> в текущей транзакции будут видны записи других успешно завершенных других транзакций\

> postgres2=# select * from persons;\
> id | first_name | second_name\
> ----+------------+-------------\
> 1 | ivan       | ivanov\
> 2 | petr       | petrov\
> 3 | sergey     | sergeev\
> (3 rows)

17. Завершите транзакцию во второй сессии 

> postgres2=*# commit;\
> COMMIT

18. Hачать новые но уже repeatable read транзации: `set transaction isolation level repeatable read;`

> postgres1=# begin;\
> BEGIN\
> postgres1=*# set transaction isolation level repeatable read;\
> SET

> postgres2=# begin;\
> BEGIN\
> postgres2=*# set transaction isolation level repeatable read;\
> SET

18. В первой сессии добавить новую запись `insert into persons(first_name, second_name) values('sveta', 'svetova');`

> postgres1=*# insert into persons(first_name, second_name) values('sveta', 'svetova');\
> INSERT 0 1

20. Cделать `select * from persons;` во второй сессии. Видите ли вы новую запись и если да то почему?

> ``#`` Новая запись не видна во второй сессии так как у нас стоит уровень изоляции repeatable read, 
> при котором данные других транзакций в текущей не видны независимо от того закоммичены они или нет

> postgres2=*# select * from persons;\
> id | first_name | second_name\
> ----+------------+-------------\
> 1 | ivan       | ivanov\
> 2 | petr       | petrov\
> 3 | sergey     | sergeev\
> (3 rows)

20. Завершить первую транзакцию: `commit;`

> postgres1=*# commit;\
> COMMIT

23. Cделать `select * from persons;` во второй сессии. Видите ли вы новую запись и если да то почему?

> ``#`` Новая запись не видна во второй сессии так как у нас стоит уровень изоляции repeatable read,
> при котором данные других транзакций в текущей не видны независимо от того закоммичены они или нет

> postgres2=*# select * from persons;\
> id | first_name | second_name\
> ----+------------+-------------\
> 1 | ivan       | ivanov\
> 2 | petr       | petrov\
> 3 | sergey     | sergeev\
> (3 rows)

22. Завершить вторую транзакцию

> postgres2=*# commit;\
> COMMIT

24. Сделать `select * from persons;` во второй сессии. Видите ли вы новую запись и если да то почему?

> ``#`` Теперь новая запись видна и во второй сессии так как эта транзакция тоже завершилась успешно и
> теперь видны данные, созданные и в других транзакциях

> postgres2=# select * from persons;\
> id | first_name | second_name\
> ----+------------+-------------\
> 1 | ivan       | ivanov\
> 2 | petr       | petrov\
> 3 | sergey     | sergeev\
> 4 | sveta      | svetova\
> (4 rows)