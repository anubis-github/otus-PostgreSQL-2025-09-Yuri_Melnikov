0. Создаем новую ВМ с параметрами как в предыдущем задании и заново ставим Postgres

```shell
# создаем ВМ через cli Я.Облако
yc compute instance create \
  --name otus-postgres-hw08-1 \
  --hostname otus-postgres-hw08-1 \
  --cores 2 \
  --core-fraction 100 \
  --memory 4 \
  --create-boot-disk size=10G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2404-lts \
  --network-interface subnet-name=default-ru-central1-b,nat-ip-version=ipv4 \
  --ssh-key ~/.ssh/ssh-key-postgres.pub
```

```shell
# Ставим Postgres, пользуясь apt
yc-user@otus-postgres-hw08-1:~$ sudo apt update && sudo apt upgrade -y -q && \
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && \
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && \
sudo apt-get update && \
sudo apt -y install postgresql

# Проверяем что кластер запущен
yc-user@otus-postgres-hw08-1:~$ sudo pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 online postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```

1. Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд.

> Подключаемся к кластеру
```shell
yc-user@otus-postgres-hw08-1:~$ sudo -u postgres psql
psql (18.0 (Ubuntu 18.0-1.pgdg24.04+3))
Type "help" for help.

postgres=#
```

> Ставим параметры ```log_lock_waits``` в ```on``` и ```deadlock_timeout``` равным ```200мс```
```
postgres=# SHOW log_lock_waits;
 log_lock_waits
----------------
 off
(1 row)

postgres=# alter system set log_lock_waits to on;
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# SHOW log_lock_waits;
 log_lock_waits
----------------
 on
(1 row)

postgres=# show deadlock_timeout;
 deadlock_timeout
------------------
 1s
(1 row)

postgres=# alter system set deadlock_timeout to 200;
ALTER SYSTEM

postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# show deadlock_timeout;
 deadlock_timeout
------------------
 200ms
(1 row)
```

2. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения

> Сессия 1
```
# Создаем базу данных locks и переключаемся на нее
postgres=# create database locks;
CREATE DATABASE
postgres=# \c locks
You are now connected to database "locks" as user "postgres".

# Создаем таблицу accounts
locks=# CREATE TABLE accounts(
  acc_no integer PRIMARY KEY,
  amount numeric
);
CREATE TABLE

# Вставляем 3 записи
locks=# INSERT INTO accounts (acc_no, amount) VALUES (1,1000.00), (2,2000.00), (3,3000.00);
INSERT 0 3

# Запоминаем id backend-процесса
locks=# SELECT pg_backend_pid();
 pg_backend_pid
----------------
           1512
(1 row)

# Начинаем транзакцию, чтобы UPDATE не отпустил блокировку сразу
locks=# BEGIN;
BEGIN
locks=*# UPDATE accounts SET amount = amount - 100.00 WHERE acc_no = 1;
UPDATE 1
```
> Сессия 2
```
# Переключаемся на базу данных locks
postgres=# \c locks
You are now connected to database "locks" as user "postgres".

# Запоминаем id backend-процесса
locks=#  SELECT pg_backend_pid();
 pg_backend_pid
----------------
           1619
(1 row)

# Пытаемся обновить ту же самую запись с acc_no = 1
locks=# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
```
> Смотрим лог сервера и видим что процесс 1619 (сессия 2)\
> ожидает разделяемую блокировку, которую удерживает процесс 1512 (сессия 1)\
> по сообщениям\
> ```process 1619 still waiting for ShareLock on transaction 768 after 200.122 ms```\
> и\
> ```Process holding the lock: 1512. Wait queue: 1619```
```shell
yc-user@otus-postgres-hw08-1:~$ sudo -su postgres;

postgres@otus-postgres-hw08-1:/home/yc-user$ tail -n 20 /var/log/postgresql/postgresql-18-main.log
...
2025-11-14 19:33:39.313 UTC [961] LOG:  checkpoint starting: time
2025-11-14 19:33:40.630 UTC [961] LOG:  checkpoint complete: wrote 13 buffers (0.1%), wrote 1 SLRU buffers; 0 WAL file(s) added, 0 removed, 0 recycled; write=1.307 s, sync=0.003 s, total=1.318 s; sync files=12, longest=0.003 s, average=0.001 s; distance=155 kB, estimate=3934 kB; lsn=0/1C14150, redo lsn=0/1C140F8
2025-11-14 19:34:59.432 UTC [1619] postgres@locks LOG:  process 1619 still waiting for ShareLock on transaction 768 after 200.122 ms
2025-11-14 19:34:59.432 UTC [1619] postgres@locks DETAIL:  Process holding the lock: 1512. Wait queue: 1619.
2025-11-14 19:34:59.432 UTC [1619] postgres@locks CONTEXT:  while updating tuple (0,1) in relation "accounts"
2025-11-14 19:34:59.432 UTC [1619] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
```
> Делаем COMMIT в сессии 1
```
locks=*# commit;
COMMIT
```
> Смотрив в сессии 2, что UPDATE завершился и делаем коммит
```
locks=# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
UPDATE 1
locks=#
```
> Смотрим логи сервера и видим, что процесс 1619 (сессия 2) смог захватить разделяемую блокировку после ожидания 228 секунд\
> по сообщениям\
> ```process 1619 acquired ShareLock on transaction 768 after 228385.492 ms```
```shell
postgres@otus-postgres-hw08-1:/home/yc-user$ tail -n 20 /var/log/postgresql/postgresql-18-main.log
2025-11-14 19:33:39.313 UTC [961] LOG:  checkpoint starting: time
2025-11-14 19:33:40.630 UTC [961] LOG:  checkpoint complete: wrote 13 buffers (0.1%), wrote 1 SLRU buffers; 0 WAL file(s) added, 0 removed, 0 recycled; write=1.307 s, sync=0.003 s, total=1.318 s; sync files=12, longest=0.003 s, average=0.001 s; distance=155 kB, estimate=3934 kB; lsn=0/1C14150, redo lsn=0/1C140F8
2025-11-14 19:34:59.432 UTC [1619] postgres@locks LOG:  process 1619 still waiting for ShareLock on transaction 768 after 200.122 ms
2025-11-14 19:34:59.432 UTC [1619] postgres@locks DETAIL:  Process holding the lock: 1512. Wait queue: 1619.
2025-11-14 19:34:59.432 UTC [1619] postgres@locks CONTEXT:  while updating tuple (0,1) in relation "accounts"
2025-11-14 19:34:59.432 UTC [1619] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
2025-11-14 19:38:47.618 UTC [1619] postgres@locks LOG:  process 1619 acquired ShareLock on transaction 768 after 228385.492 ms
2025-11-14 19:38:47.618 UTC [1619] postgres@locks CONTEXT:  while updating tuple (0,1) in relation "accounts"
2025-11-14 19:38:47.618 UTC [1619] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
```

3. Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.

> Сессия 1
```
# Переключаемся на базу данных locks
postgres=# \c locks
You are now connected to database "locks" as user "postgres".

# Запоминаем id backend-процесса
locks=# SELECT pg_backend_pid();
 pg_backend_pid
----------------
           1512
(1 row)

locks=# begin;
BEGIN
locks=*# UPDATE accounts SET amount = amount - 100.00 WHERE acc_no = 1;
UPDATE 1
```
> Сессия 2
```
# Переключаемся на базу данных locks
postgres=# \c locks
You are now connected to database "locks" as user "postgres".

# Запоминаем id backend-процесса
locks=#  SELECT pg_backend_pid();
 pg_backend_pid
----------------
           1619
(1 row)

locks=# BEGIN;
BEGIN
locks=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
```

> Сессия 3 
```
# Переключаемся на базу данных locks
postgres=# \c locks
You are now connected to database "locks" as user "postgres".

# Запоминаем id backend-процесса
locks=#  SELECT pg_backend_pid();
 pg_backend_pid
----------------
           2313
(1 row)

locks=# BEGIN;
BEGIN
locks=*# UPDATE accounts SET amount = amount + 200.00 WHERE acc_no = 1;
```

> Сессия 4. Содержимое представления pg_locks (c комментарием к каждой строке)
```
postgres=# \c locks
You are now connected to database "locks" as user "postgres".

locks=# SELECT pid, pg_blocking_pids(pid) AS wait_for, locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted FROM pg_locks WHERE pid IN (1512, 1619, 2313);
 pid  | wait_for |   locktype    |   relation    | virtxid | xid |       mode       | granted
------+----------+---------------+---------------+---------+-----+------------------+---------
-- сессия 1 (pid=1512), тип - блокировка отношения, отношение - таблица accounts, режим - RowExclusiveLock (исключительныый на уровне строк, при операции UPDATE или DELETE), успешно получена
 1512 | {}       | relation      | accounts      |         |     | RowExclusiveLock | t
          
-- сессия 1 (pid=1512), тип - блокировка отношения, отношение - индекс (primary key) accounts_pkey, режим - RowExclusiveLock (исключительныый на уровне строк, при операции UPDATE или DELETE), успешно получена
 1512 | {}       | relation      | accounts_pkey |         |     | RowExclusiveLock | t
          
-- сессия 1 (pid=1512), тип - блокировка виртуальной транзакции, режим - ExclusiveLock (исключительный), успешно получена 
 1512 | {}       | virtualxid    |               | 7/11    |     | ExclusiveLock    | t
          
-- сессия 2 (pid=1619), тип - блокировка отношения, отношение - индекс (primary key) accounts_pkey, режим - RowExclusiveLock (исключительныый на уровне строк, при операции UPDATE или DELETE), успешно получена
 1619 | {1512}   | relation      | accounts      |         |     | RowExclusiveLock | t
          
-- сессия 2 (pid=1619), тип - блокировка отношения, отношение - индекс (primary key) accounts_pkey, режим - RowExclusiveLock (исключительныый на уровне строк, при операции UPDATE или DELETE), успешно получена
 1619 | {1512}   | relation      | accounts_pkey |         |     | RowExclusiveLock | t
          
-- сессия 2 (pid=1619), тип - блокировка виртуальной транзакции, режим - ExclusiveLock (исключительный), успешно получена
 1619 | {1512}   | virtualxid    |               | 9/5     |     | ExclusiveLock    | t
          
-- сессия 3 (pid=2313), тип - блокировка отношения, отношение - индекс (primary key) accounts_pkey, режим - RowExclusiveLock (исключительныый на уровне строк, при операции UPDATE или DELETE), успешно получена
 2313 | {1619}   | relation      | accounts      |         |     | RowExclusiveLock | t
          
-- сессия 3 (pid=2313), тип - блокировка отношения, отношение - индекс (primary key) accounts_pkey, режим - RowExclusiveLock (исключительныый на уровне строк, при операции UPDATE или DELETE), успешно получена
 2313 | {1619}   | relation      | accounts_pkey |         |     | RowExclusiveLock | t
          
-- сессия 3 (pid=2313), тип - блокировка виртуальной транзакции, режим - ExclusiveLock (исключительный), успешно получена
 2313 | {1619}   | virtualxid    |               | 11/6    |     | ExclusiveLock    | t
          
-- сессия 2 (pid=1619), тип - блокировка версии строки, отношение - таблица accounts, режим - ExclusiveLock, успешно получена
 1619 | {1512}   | tuple         | accounts      |         |     | ExclusiveLock    | t
          
-- сессия 3 (pid=2313), тип - блокировка транзакции в сессии 3 на транзакцию в сессии 2, режим - ExclusiveLock (исключительный), успешно получена
 2313 | {1619}   | transactionid |               |         | 775 | ExclusiveLock    | t
          
-- сессия 2 (pid=1619), тип - блокировка транзакции, режим - ShareLock (разделяемый), в ожидании
 1619 | {1512}   | transactionid |               |         | 773 | ShareLock        | f
          
-- сессия 2 (pid=1619), тип - блокировка транзакции в сессии 2 на транзакцию в сессии 1, режим - ExclusiveLock (исключительный), успешно получена
 1619 | {1512}   | transactionid |               |         | 774 | ExclusiveLock    | t
          
-- сессия 1 (pid=1512), тип - блокировка транзакции в сессии 1, режим - ExclusiveLock (исключительный), успешно получена
 1512 | {}       | transactionid |               |         | 773 | ExclusiveLock    | t
          
-- сессия 3 (pid=2313), тип - блокировка версии строки, отношение - таблица accounts, режим - ExclusiveLock (исключительныый, при операции UPDATE или DELETE), в ожидании
 2313 | {1619}   | tuple         | accounts      |         |     | ExclusiveLock    | f         
 ```

> Мы видим следующие типы блокировок (есть вопросы, выделены **жирным шрифтом**)
> - блокировки уровня отношения таблицы accounts во всех трех сессиях с режимом RowExclusiveLock потому что мы пытаемся модифицировать одну и ту же строку из трех транзакций и все они блокируют ее на модификацию
> - блокировки уровня отношения индекса accounts_pkey во всех трех сессиях с режимом RowExclusiveLock потому что мы пытаемся модифицировать строку c одним и тем же первичным ключом из трех транзакций и все они блокируют ее и ключ на модификацию
> - блокировки уровня версии строки (tuple) в таблице accounts из сессии 2 (pid=1619) и 3 (pid=2313) c режимом ExclusiveLock. **Вопрос - почему для процесса (pid=1619) блокировка получена, а для процесса  (pid=2313) нет? Из-за того, что транзакции выстроились в очередь на модификацию этой записи?** 
> - блокировки уровня виртуальной транзакции во всех трех сессиях  c режимом ExclusiveLock. **Вопрос - что это и что значит xid в этом случае?**
> - блокировки уровня транзакции во всех трех сессиях c режимом ExclusiveLock
> - четвертую блокировку уровня транзакции в сессии 2 (pid=1619) с режимом ShareLock с xid равным xid транзакции из сессии 1 (773). **Вопрос - что это за блокировка?**

```
locks=# CREATE EXTENSION pgrowlocks;
CREATE EXTENSION
locks=# SELECT * FROM pgrowlocks('accounts') \gx
-[ RECORD 1 ]-----------------
locked_row | (0,8)
locker     | 773
multi      | f
xids       | {773}
modes      | {"No Key Update"}
pids       | {1512}
``` 

> Сессия 1
```
locks=# begin;
BEGIN
locks=*# UPDATE accounts SET amount = amount - 100.00 WHERE acc_no = 1;
UPDATE 1
locks=*# commit;
COMMIT
```
> Сессия 2
```
locks=# BEGIN;
BEGIN
locks=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
UPDATE 1
locks=*# commit;
COMMIT
```

> Сессия 3
```
locks=# BEGIN;
BEGIN
locks=*# UPDATE accounts SET amount = amount + 200.00 WHERE acc_no = 1;
UPDATE 1
```

> Сессия 4. Содержимое представления pg_locks
```
locks=# SELECT pid, pg_blocking_pids(pid) AS wait_for, locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted FROM pg_locks WHERE pid IN (1512, 1619, 2313);
 pid | wait_for | locktype | relation | virtxid | xid | mode | granted
-----+----------+----------+----------+---------+-----+------+---------
(0 rows)
```


4. Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?

> Так как запись блокировок в журнал сообщений длиннее 200мс было оставлено, то в логе можно видеть последовательность событий из предыдущего пункта
> Отвечая на вопрос - да, можно разобраться в ситуации постфактум, читая журнал сообщений
```shell
postgres@otus-postgres-hw08-1:/home/yc-user$ tail -n 100 /var/log/postgresql/postgresql-18-main.log
...
2025-11-14 21:04:11.274 UTC [1619] postgres@locks LOG:  process 1619 still waiting for ShareLock on transaction 773 after 200.125 ms
2025-11-14 21:04:11.274 UTC [1619] postgres@locks DETAIL:  Process holding the lock: 1512. Wait queue: 1619.
2025-11-14 21:04:11.274 UTC [1619] postgres@locks CONTEXT:  while updating tuple (0,8) in relation "accounts"
2025-11-14 21:04:11.274 UTC [1619] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
2025-11-14 21:04:24.577 UTC [2313] postgres@locks LOG:  process 2313 still waiting for ExclusiveLock on tuple (0,8) of relation 16389 of database 16388 after 200.210 ms
2025-11-14 21:04:24.577 UTC [2313] postgres@locks DETAIL:  Process holding the lock: 1619. Wait queue: 2313.
2025-11-14 21:04:24.577 UTC [2313] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 200.00 WHERE acc_no = 1;
2025-11-14 21:08:40.233 UTC [961] LOG:  checkpoint starting: time
2025-11-14 21:08:40.349 UTC [961] LOG:  checkpoint complete: wrote 1 buffers (0.0%), wrote 0 SLRU buffers; 0 WAL file(s) added, 0 removed, 0 recycled; write=0.101 s, sync=0.004 s, total=0.117 s; sync files=1, longest=0.004 s, average=0.004 s; distance=0 kB, estimate=2323 kB; lsn=0/1C15020, redo lsn=0/1C14FC0
2025-11-14 21:23:40.506 UTC [961] LOG:  checkpoint starting: time
2025-11-14 21:23:40.675 UTC [961] LOG:  checkpoint complete: wrote 1 buffers (0.0%), wrote 0 SLRU buffers; 0 WAL file(s) added, 0 removed, 0 recycled; write=0.101 s, sync=0.003 s, total=0.170 s; sync files=1, longest=0.003 s, average=0.003 s; distance=2 kB, estimate=2091 kB; lsn=0/1C15BB0, redo lsn=0/1C15B50
2025-11-14 22:08:41.493 UTC [961] LOG:  checkpoint starting: time
2025-11-14 22:08:43.212 UTC [961] LOG:  checkpoint complete: wrote 17 buffers (0.1%), wrote 1 SLRU buffers; 0 WAL file(s) added, 0 removed, 0 recycled; write=1.709 s, sync=0.003 s, total=1.720 s; sync files=17, longest=0.002 s, average=0.001 s; distance=55 kB, estimate=1887 kB; lsn=0/1C237C0, redo lsn=0/1C23760
2025-11-14 22:33:09.498 UTC [1619] postgres@locks LOG:  process 1619 acquired ShareLock on transaction 773 after 5338424.271 ms
2025-11-14 22:33:09.498 UTC [1619] postgres@locks CONTEXT:  while updating tuple (0,8) in relation "accounts"
2025-11-14 22:33:09.498 UTC [1619] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
2025-11-14 22:33:09.498 UTC [2313] postgres@locks LOG:  process 2313 acquired ExclusiveLock on tuple (0,8) of relation 16389 of database 16388 after 5325121.649 ms
2025-11-14 22:33:09.498 UTC [2313] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 200.00 WHERE acc_no = 1;
2025-11-14 22:33:09.698 UTC [2313] postgres@locks LOG:  process 2313 still waiting for ShareLock on transaction 774 after 200.173 ms
2025-11-14 22:33:09.698 UTC [2313] postgres@locks DETAIL:  Process holding the lock: 1619. Wait queue: 2313.
2025-11-14 22:33:09.698 UTC [2313] postgres@locks CONTEXT:  while rechecking updated tuple (0,9) in relation "accounts"
2025-11-14 22:33:09.698 UTC [2313] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 200.00 WHERE acc_no = 1;
2025-11-14 22:33:22.273 UTC [2313] postgres@locks LOG:  process 2313 acquired ShareLock on transaction 774 after 12774.234 ms
2025-11-14 22:33:22.273 UTC [2313] postgres@locks CONTEXT:  while rechecking updated tuple (0,9) in relation "accounts"
2025-11-14 22:33:22.273 UTC [2313] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 200.00 WHERE acc_no = 1;
2025-11-14 22:33:41.665 UTC [961] LOG:  checkpoint starting: time
2025-11-14 22:33:41.787 UTC [961] LOG:  checkpoint complete: wrote 1 buffers (0.0%), wrote 1 SLRU buffers; 0 WAL file(s) added, 0 removed, 0 recycled; write=0.103 s, sync=0.004 s, total=0.122 s; sync files=2, longest=0.003 s, average=0.002 s; distance=1 kB, estimate=1699 kB; lsn=0/1C23C98, redo lsn=0/1C23C40
2025-11-14 22:34:38.700 UTC [1619] postgres@locks LOG:  process 1619 still waiting for ShareLock on transaction 777 after 200.110 ms
2025-11-14 22:34:38.700 UTC [1619] postgres@locks DETAIL:  Process holding the lock: 1512. Wait queue: 1619.
2025-11-14 22:34:38.700 UTC [1619] postgres@locks CONTEXT:  while updating tuple (0,11) in relation "accounts"
2025-11-14 22:34:38.700 UTC [1619] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
2025-11-14 22:35:38.492 UTC [1619] postgres@locks LOG:  process 1619 acquired ShareLock on transaction 777 after 59991.915 ms
2025-11-14 22:35:38.492 UTC [1619] postgres@locks CONTEXT:  while updating tuple (0,11) in relation "accounts"
2025-11-14 22:35:38.492 UTC [1619] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
...
```

5. Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?

> Не смог придумать как может получиться такая ситуация, если использовать только одну команду ```UPDATE``` одной и той же таблицы без ```where```