0. Создаем новую ВМ с параметрами как в предыдущем задании и заново ставим Postgres

```shell
# создаем ВМ через cli Я.Облако
yc compute instance create \
  --name otus-postgres-hw07-1 \
  --hostname otus-postgres-hw07-1 \
  --cores 2 \
  --core-fraction 100 \
  --memory 4 \
  --create-boot-disk size=10G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2404-lts \
  --network-interface subnet-name=default-ru-central1-b,nat-ip-version=ipv4 \
  --ssh-key ~/.ssh/ssh-key-postgres.pub
```

```shell
# Ставим Postgres, пользуясь apt
yc-user@otus-postgres-1:~$ sudo apt update && sudo apt upgrade -y -q && \
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && \
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && \
sudo apt-get update && \
sudo apt -y install postgresql

# Проверяем что кластер запущен
yc-user@otus-postgres-1:~$ sudo pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 online postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```
1. Настройте выполнение контрольной точки раз в 30 секунд.

```
postgres=# select name, setting, context, unit, source from pg_settings where name = 'checkpoint_timeout';
name        | setting | context | unit | source
--------------------+---------+---------+------+---------
checkpoint_timeout | 300     | sighup  | s    | default
(1 row)
```
```
postgres=# alter system set checkpoint_timeout=30;
ALTER SYSTEM
postgres=# select pg_reload_conf();
pg_reload_conf
----------------
t
(1 row)
```
```
postgres=# select name, setting, context, unit, source from pg_settings where name = 'checkpoint_timeout';
name        | setting | context | unit |       source
--------------------+---------+---------+------+--------------------
checkpoint_timeout | 30      | sighup  | s    | configuration file
(1 row)
```

2. 10 минут c помощью утилиты pgbench подавайте нагрузку.

```
yc-user@otus-postgres-hw07-1:~$ sudo -u postgres pgbench -i postgres
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
vacuuming...
creating primary keys...
done in 1.17 s (drop tables 0.00 s, create tables 0.10 s, client-side generate 0.90 s, vacuum 0.06 s, primary keys 0.11 s).
```

```
yc-user@otus-postgres-hw07-1:~$ sudo -u postgres pgbench -c 8 -P 30 -T 600 -U postgres postgres
pgbench (18.0 (Ubuntu 18.0-1.pgdg24.04+3))
starting vacuum...end.
progress: 30.0 s, 449.1 tps, lat 17.780 ms stddev 19.850, 0 failed
progress: 60.0 s, 459.7 tps, lat 17.405 ms stddev 19.922, 0 failed
progress: 90.0 s, 471.5 tps, lat 16.959 ms stddev 18.695, 0 failed
progress: 120.0 s, 425.3 tps, lat 18.809 ms stddev 21.306, 0 failed
progress: 150.0 s, 472.8 tps, lat 16.915 ms stddev 17.905, 0 failed
progress: 180.0 s, 416.2 tps, lat 19.220 ms stddev 22.594, 0 failed
progress: 210.0 s, 481.1 tps, lat 16.623 ms stddev 18.481, 0 failed
progress: 240.0 s, 479.6 tps, lat 16.678 ms stddev 18.712, 0 failed
progress: 270.0 s, 488.9 tps, lat 16.362 ms stddev 18.105, 0 failed
progress: 300.0 s, 443.4 tps, lat 18.038 ms stddev 19.252, 0 failed
progress: 330.0 s, 464.3 tps, lat 17.223 ms stddev 19.262, 0 failed
progress: 360.0 s, 446.9 tps, lat 17.894 ms stddev 19.584, 0 failed
progress: 390.0 s, 442.8 tps, lat 18.064 ms stddev 19.232, 0 failed
progress: 420.0 s, 465.3 tps, lat 17.191 ms stddev 18.890, 0 failed
progress: 450.0 s, 452.7 tps, lat 17.666 ms stddev 19.506, 0 failed
progress: 480.0 s, 421.8 tps, lat 18.957 ms stddev 21.168, 0 failed
progress: 510.0 s, 485.7 tps, lat 16.474 ms stddev 18.293, 0 failed
progress: 540.0 s, 487.1 tps, lat 16.418 ms stddev 17.984, 0 failed
progress: 570.0 s, 467.2 tps, lat 17.116 ms stddev 18.635, 0 failed
progress: 600.0 s, 452.2 tps, lat 17.687 ms stddev 20.368, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 275219
number of failed transactions: 0 (0.000%)
latency average = 17.437 ms
latency stddev = 19.390 ms
initial connection time = 26.301 ms
tps = 458.672502 (without initial connection time)
```

3. Измерьте, какой объем журнальных файлов был сгенерирован за это время.

```shell
yc-user@otus-postgres-hw07-1:~$ sudo ls -lh /var/lib/postgresql/18/main/pg_wal
total 65M
-rw------- 1 postgres postgres  16M Nov  3 17:19 00000001000000000000001D
-rw------- 1 postgres postgres  16M Nov  3 17:16 00000001000000000000001E
-rw------- 1 postgres postgres  16M Nov  3 17:16 00000001000000000000001F
-rw------- 1 postgres postgres  16M Nov  3 17:17 000000010000000000000020
drwx------ 2 postgres postgres 4.0K Nov  3 14:29 archive_status
drwx------ 2 postgres postgres 4.0K Nov  3 14:29 summaries
```

4. Оцените, какой объем приходится в среднем на одну контрольную точку.
> За 10 минут должно было произойти 20 контрольных точек, то есть 65М/20 примерно равно 3,25M

5. Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?

> Почему-то выполнилось всего 3 контрольных точки вместо 20. Возможно из-за большой нагрузки, но это странно.\
> Еще странно, что в конце вывода статистики идет запись pg_waldump: error: error in WAL record at 0/1D90C430: invalid record length at 0/1D90C468: expected at least 24, got 0 

> Пересмотрел лекцию, перечитал документацию про настройку checkpoint_timeout, но ответа не нашел

```shell
yc-user@otus-postgres-hw07-1:~$ sudo /usr/lib/postgresql/18/bin/pg_waldump -p /var/lib/postgresql/18/main/pg_wal 00000001000000000000001D 000000010000000000000020 | grep CHECK
rmgr: XLOG        len (rec/tot):    114/   114, tx:          0, lsn: 0/1D8FED00, prev 0/1D8FECC8, desc: CHECKPOINT_ONLINE redo 0/1CF3E1F8; tli 1; prev tli 1; fpw true; wal_level replica; xid 0:274452; oid 24580; multi 1; offset 0; oldest xid 744 in DB 1; oldest multi 1 in DB 1; oldest/newest commit timestamp xid: 0/0; oldest running xid 274422; online
rmgr: XLOG        len (rec/tot):     30/    30, tx:          0, lsn: 0/1D90C328, prev 0/1D90C2F0, desc: CHECKPOINT_REDO wal_level replica
rmgr: XLOG        len (rec/tot):    114/   114, tx:          0, lsn: 0/1D90C3B8, prev 0/1D90C380, desc: CHECKPOINT_ONLINE redo 0/1D90C328; tli 1; prev tli 1; fpw true; wal_level replica; xid 0:276046; oid 24580; multi 1; offset 0; oldest xid 744 in DB 1; oldest multi 1 in DB 1; oldest/newest commit timestamp xid: 0/0; oldest running xid 276046; online
pg_waldump: error: error in WAL record at 0/1D90C430: invalid record length at 0/1D90C468: expected at least 24, got 0
```

6. Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.

> Выключаем synchronous_commit
```
postgres=# show synchronous_commit
postgres-# ;
synchronous_commit
--------------------
on
(1 row)

postgres=# alter system set synchronous_commit = off;
ALTER SYSTEM
postgres=# show synchronous_commit;
synchronous_commit
--------------------
on
(1 row)

postgres=# select pg_reload_conf();
pg_reload_conf
----------------
t
(1 row)

postgres=# show synchronous_commit;
synchronous_commit
--------------------
off
(1 row)
```

> Инициализируем и запускаем pgbench
```shell
yc-user@otus-postgres-hw07-1:~$ sudo -u postgres pgbench -i postgres
dropping old tables...
creating tables...
generating data (client-side)...
vacuuming...
creating primary keys...
done in 0.22 s (drop tables 0.02 s, create tables 0.00 s, client-side generate 0.12 s, vacuum 0.04 s, primary keys 0.04 s).
```

```shell
yc-user@otus-postgres-hw07-1:~$ sudo -u postgres pgbench -c 8 -P 30 -T 600 -U postgres postgres
pgbench (18.0 (Ubuntu 18.0-1.pgdg24.04+3))
starting vacuum...end.
progress: 30.0 s, 2017.4 tps, lat 3.958 ms stddev 2.032, 0 failed
progress: 60.0 s, 1988.5 tps, lat 4.019 ms stddev 2.015, 0 failed
progress: 90.0 s, 1962.1 tps, lat 4.074 ms stddev 2.156, 0 failed
progress: 120.0 s, 2018.5 tps, lat 3.960 ms stddev 2.084, 0 failed
progress: 150.0 s, 2036.0 tps, lat 3.925 ms stddev 2.033, 0 failed
progress: 180.0 s, 2060.3 tps, lat 3.879 ms stddev 1.991, 0 failed
progress: 210.0 s, 2040.4 tps, lat 3.917 ms stddev 2.009, 0 failed
progress: 240.0 s, 2036.4 tps, lat 3.925 ms stddev 1.979, 0 failed
progress: 270.0 s, 2032.9 tps, lat 3.931 ms stddev 1.991, 0 failed
progress: 300.0 s, 2029.5 tps, lat 3.938 ms stddev 1.993, 0 failed
progress: 330.0 s, 2055.8 tps, lat 3.888 ms stddev 2.016, 0 failed
progress: 360.0 s, 2083.1 tps, lat 3.836 ms stddev 1.952, 0 failed
progress: 390.0 s, 2027.3 tps, lat 3.942 ms stddev 2.019, 0 failed
progress: 420.0 s, 2012.6 tps, lat 3.971 ms stddev 2.004, 0 failed
progress: 450.0 s, 1995.0 tps, lat 4.006 ms stddev 2.054, 0 failed
progress: 480.0 s, 2080.1 tps, lat 3.842 ms stddev 1.995, 0 failed
progress: 510.0 s, 2046.6 tps, lat 3.905 ms stddev 2.008, 0 failed
progress: 540.0 s, 2050.2 tps, lat 3.898 ms stddev 1.994, 0 failed
progress: 570.0 s, 2032.6 tps, lat 3.932 ms stddev 2.047, 0 failed
progress: 600.0 s, 2072.1 tps, lat 3.857 ms stddev 1.962, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 1220332
number of failed transactions: 0 (0.000%)
latency average = 3.929 ms
latency stddev = 2.018 ms
initial connection time = 26.453 ms
tps = 2033.926959 (without initial connection time)
```

> Показатель TPS возрос с 458 до 2033 так как при завершении транзакции Postgres\
> больше не ждет физической записи транзакции в WAL файл перед тем как отправить клиенту сообщение о завершении. 
> Это поднимает производительность, но увеличивает риск потери данных при сбое 

7. Создайте новый кластер с включенной контрольной суммой страниц.

> Чтобы создать новый кластер сделаем еще одну ВМ c именем ```otus-postgres-hw07-2``` и развернем на ней Postgres\
> как это сделано в п.0\
> После этого пересоздадим кластер main с параметром ```--data-checksums```, который включит контрольные суммы

```shell
# Останавливаем кластер
yc-user@otus-postgres-hw07-2:~$ sudo -u postgres pg_ctlcluster 18 main stop
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@18-main

# Проверяем что кластер остановлен
yc-user@otus-postgres-hw07-2:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 down   postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log

# Пересоздаем каталог /var/lib/postgresql/18/main так как иначе initdb не будет пересоздавать кластер
yc-user@otus-postgres-hw07-2:~$ sudo -u postgres rm -rf /var/lib/postgresql/18/main
yc-user@otus-postgres-hw07-2:~$ sudo -u postgres mkdir /var/lib/postgresql/18/main
yc-user@otus-postgres-hw07-2:~$ sudo -u postgres chown -R postgres /var/lib/postgresql/18/main

# Пересоздаем кластер с параметром --data-checksums
yc-user@otus-postgres-hw07-2:~$ sudo -u postgres /usr/lib/postgresql/18/bin/initdb -D /var/lib/postgresql/18/main --data-checksums
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "C.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are enabled.

fixing permissions on existing directory /var/lib/postgresql/18/main ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default "max_connections" ... 100
selecting default "shared_buffers" ... 128MB
selecting default time zone ... Etc/UTC
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok

initdb: warning: enabling "trust" authentication for local connections
initdb: hint: You can change this by editing pg_hba.conf or using the option -A, or --auth-local and --auth-host, the next time you run initdb.

Success. You can now start the database server using:

    /usr/lib/postgresql/18/bin/pg_ctl -D /var/lib/postgresql/18/main -l logfile start

# Запускаем кластер
yc-user@otus-postgres-hw07-2:~$ sudo -u postgres pg_ctlcluster 18 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@18-main
  
# Убеждаемся что он запустился  
yc-user@otus-postgres-hw07-2:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 online postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```

8. Создайте таблицу.
```
postgres=# create table t1(name char(100));
CREATE TABLE
```

9. Вставьте несколько значений.

```
postgres=# insert into t1 values ('value1');
INSERT 0 1
postgres=# insert into t1 values ('value2');
INSERT 0 1
postgres=# insert into t1 values ('value3');
INSERT 0 1
postgres=# insert into t1 values ('value4');
INSERT 0 1
postgres=# insert into t1 values ('value5');
INSERT 0 1
```

> Смотрим в каком файле лежат данные таблицы
```
postgres=# SELECT pg_relation_filepath('t1');
 pg_relation_filepath
----------------------
 base/5/16388
(1 row)
```

10. Выключите кластер.

```shell
yc-user@otus-postgres-hw07-2:~$ sudo -u postgres pg_ctlcluster 18 main stop
yc-user@otus-postgres-hw07-2:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 down   postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```

11. Измените пару байт в таблице.

```shell
# Добавляем ХХХ в начало файла и сохраняем изменения
yc-user@otus-postgres-hw07-2:~$ sudo -u postgres nano /var/lib/postgresql/18/main/base/5/16388
  GNU nano 7.2                                                                           /var/lib/postgresql/18/main/base/5/16388                                                                               M
XXX^@^@^@^@�W|^AL:^@^@,^@�^]^@ ^D ^@^@^@^@���^@^@��^@���^@^@��^@���^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@^@>
^G Help          ^O Write Out     ^W Where Is      ^K Cut           ^T Execute       ^C Location      M-U Undo         M-A Set Mark     M-] To Bracket   M-Q Previous     ^B Back          ^◂ Prev Word
^X Exit          ^R Read File     ^\ Replace       ^U Paste         ^J Justify       ^/ Go To Line    M-E Redo         M-6 Copy         ^Q Where Was     M-W Next         ^F Forward       ^▸ Next Word
```

12. Включите кластер и сделайте выборку из таблицы.
```shell
yc-user@otus-postgres-hw07-2:~$ sudo -u postgres pg_ctlcluster 18 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@18-main

yc-user@otus-postgres-hw07-2:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 online postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log

yc-user@otus-postgres-hw07-2:~$ sudo -u postgres psql
psql (18.0 (Ubuntu 18.0-1.pgdg24.04+3))
Type "help" for help.

postgres=#
```
```
postgres=# select name from t1;
ERROR:  invalid page in block 0 of relation "base/5/16388"
```

13. Что и почему произошло?
> Произошла ошибка проверки контрольной суммы при доступе к таблице 

14. Как проигнорировать ошибку и продолжить работу?
> Поиском ищутся три рекомендации:
> - включить опцию zero_damaged_pages
> - сделать VACUUM FULL
> - reindex table
> 
> Попробуем первые 2 так как третья нужна при поврежденном файле индекса

> Устанавливаем ```zero_damaged_pages=on``` и запускаем запрос
```
postgres=# alter system set zero_damaged_pages=on;
ALTER SYSTEM
postgres=# select name, setting, context from pg_settings where name = 'zero_damaged_pages';
        name        | setting |  context
--------------------+---------+-----------
 zero_damaged_pages | off     | superuser
 
postgres=# alter system set zero_damaged_pages=on;
ALTER SYSTEM

postgres=# select name, setting, context from pg_settings where name = 'zero_damaged_pages';
        name        | setting |  context
--------------------+---------+-----------
 zero_damaged_pages | on      | superuser
(1 row)

postgres=# select name from t1;
WARNING:  invalid page in block 0 of relation "base/5/16388"; zeroing out page
 name
------
(0 rows)
```

> Не помогло, что не очень удивительно. Пробуем VACUUM FULL

```
postgres=# VACUUM FULL;
VACUUM
postgres=# select name from t1;
 name
------
(0 rows)
```

> Теперь помогло, ошибка ушла, но и данные потерялись