1. Создать инстанс ВМ с 2 ядрами и 4 Гб ОЗУ и SSD 10GB
```shell
# создаем ВМ через cli Я.Облако
> yc compute instance create \
  --name otus-postgres-1 \
  --hostname otus-postgres-1 \
  --cores 2 \
  --core-fraction 100 \
  --memory 4 \
  --create-boot-disk size=10G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2404-lts \
  --network-interface subnet-name=default-ru-central1-b,nat-ip-version=ipv4 \
  --ssh-key ~/.ssh/ssh-key-postgres.pub
  
# Проверяем что ВМ создана и запущена
❯ yc compute instance list
+----------------------+-----------------+---------------+---------+----------------+-------------+
|          ID          |      NAME       |    ZONE ID    | STATUS  |  EXTERNAL IP   | INTERNAL IP |
+----------------------+-----------------+---------------+---------+----------------+-------------+
| epdofdp5hmvi6142elq4 | otus-postgres-1 | ru-central1-b | RUNNING | 89.169.189.226 | 10.129.0.30 |
+----------------------+-----------------+---------------+---------+----------------+-------------+  
```
2. Установить на него PostgreSQL 18 с дефолтными настройками
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

3. Создать БД для тестов: выполнить ```pgbench -i postgres```

```shell
yc-user@otus-postgres-1:~$ sudo -u postgres pgbench -i postgres
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
vacuuming...
creating primary keys...
done in 1.10 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.94 s, vacuum 0.05 s, primary keys 0.10 s).
```

4. Запустить ```pgbench -c 8 -P 6 -T 60 -U postgres postgres```

```shell
yc-user@otus-postgres-1:~$ sudo -u postgres pgbench -c 8 -P 6 -T 60 -U postgres postgres
pgbench (18.0 (Ubuntu 18.0-1.pgdg24.04+3))
starting vacuum...end.
progress: 6.0 s, 500.8 tps, lat 15.868 ms stddev 16.734, 0 failed
progress: 12.0 s, 494.0 tps, lat 16.176 ms stddev 17.926, 0 failed
progress: 18.0 s, 490.8 tps, lat 16.250 ms stddev 18.368, 0 failed
progress: 24.0 s, 495.2 tps, lat 16.215 ms stddev 17.506, 0 failed
progress: 30.0 s, 496.3 tps, lat 15.988 ms stddev 17.347, 0 failed
progress: 36.0 s, 499.5 tps, lat 16.136 ms stddev 18.238, 0 failed
progress: 42.0 s, 597.7 tps, lat 13.376 ms stddev 14.604, 0 failed
progress: 48.0 s, 589.8 tps, lat 13.559 ms stddev 15.409, 0 failed
progress: 54.0 s, 618.7 tps, lat 12.915 ms stddev 13.443, 0 failed
progress: 60.0 s, 465.3 tps, lat 17.128 ms stddev 18.881, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 31497
number of failed transactions: 0 (0.000%)
latency average = 15.232 ms
latency stddev = 16.857 ms
initial connection time = 28.426 ms
tps = 524.975304 (without initial connection time)
```

5. Применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла
> Видимо, имеются в виду настройки Amazon RDS для Postgres:
> - log_autovacuum_min_duration = 0
> - autovacuum_max_workers = 10
> - autovacuum_naptime = 15s
> - autovacuum_vacuum_threshold = 25
> - autovacuum_vacuum_scale_factor = 0.05
> - autovacuum_vacuum_cost_delay = 10
> - autovacuum_vacuum_cost_limit = 1000

 ```shell
 yc-user@otus-postgres-1:~$ sudo -u postgres psql
 ```

> Посмотрим значения по умолчанию для настроек, которые собираемся менять
```
postgres=# select
    name, setting, category, context, source 
from 
    pg_settings
where
    name in (
        'log_autovacuum_min_duration',
        'autovacuum_max_workers',
        'autovacuum_naptime',
        'autovacuum_vacuum_threshold',
        'autovacuum_vacuum_scale_factor',
        'autovacuum_vacuum_cost_delay',
        'autovacuum_vacuum_cost_limit'
    );
              name              | setting |              category               | context | source
--------------------------------+---------+-------------------------------------+---------+---------
 autovacuum_max_workers         | 3       | Vacuuming / Automatic Vacuuming     | sighup  | default
 autovacuum_naptime             | 60      | Vacuuming / Automatic Vacuuming     | sighup  | default
 autovacuum_vacuum_cost_delay   | 2       | Vacuuming / Automatic Vacuuming     | sighup  | default
 autovacuum_vacuum_cost_limit   | -1      | Vacuuming / Automatic Vacuuming     | sighup  | default
 autovacuum_vacuum_scale_factor | 0.2     | Vacuuming / Automatic Vacuuming     | sighup  | default
 autovacuum_vacuum_threshold    | 50      | Vacuuming / Automatic Vacuuming     | sighup  | default
 log_autovacuum_min_duration    | 600000  | Reporting and Logging / What to Log | sighup  | default
(7 rows)
```

> Изменяем параметры
```
postgres=# alter system set log_autovacuum_min_duration = 0;
ALTER SYSTEM
postgres=# alter system set autovacuum_max_workers = 10;
ALTER SYSTEM
postgres=# alter system set autovacuum_naptime = 15;
ALTER SYSTEM
postgres=# alter system set autovacuum_vacuum_threshold = 25;
ALTER SYSTEM
postgres=# alter system set autovacuum_vacuum_scale_factor = 0.05;
ALTER SYSTEM
postgres=# alter system set autovacuum_vacuum_cost_delay = 10;
ALTER SYSTEM
postgres=# alter system set autovacuum_vacuum_cost_limit = 1000;
ALTER SYSTEM
```

> Применяем изменения в конфигурации, заставляя postman перечитать настройки
```
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)
```

> Проверяем что новые параметры применились
```
postgres=# select
    name, setting, category, context, source from pg_settings
where
    name in (
        'log_autovacuum_min_duration',
        'autovacuum_max_workers',
        'autovacuum_naptime',
        'autovacuum_vacuum_threshold',
        'autovacuum_vacuum_scale_factor',
        'autovacuum_vacuum_cost_delay',
        'autovacuum_vacuum_cost_limit'
    );
              name              | setting |              category               | context |       source
--------------------------------+---------+-------------------------------------+---------+--------------------
 autovacuum_max_workers         | 10      | Vacuuming / Automatic Vacuuming     | sighup  | configuration file
 autovacuum_naptime             | 15      | Vacuuming / Automatic Vacuuming     | sighup  | configuration file
 autovacuum_vacuum_cost_delay   | 10      | Vacuuming / Automatic Vacuuming     | sighup  | configuration file
 autovacuum_vacuum_cost_limit   | 1000    | Vacuuming / Automatic Vacuuming     | sighup  | configuration file
 autovacuum_vacuum_scale_factor | 0.05    | Vacuuming / Automatic Vacuuming     | sighup  | configuration file
 autovacuum_vacuum_threshold    | 25      | Vacuuming / Automatic Vacuuming     | sighup  | configuration file
 log_autovacuum_min_duration    | 0       | Reporting and Logging / What to Log | sighup  | configuration file
(7 rows)
```

6. Протестировать заново

```shell
yc-user@otus-postgres-1:~$ sudo -u postgres pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (18.0 (Ubuntu 18.0-1.pgdg24.04+3))
starting vacuum...end.
progress: 6.0 s, 521.5 tps, lat 15.207 ms stddev 14.502, 0 failed
progress: 12.0 s, 380.8 tps, lat 21.071 ms stddev 21.155, 0 failed
progress: 18.0 s, 304.0 tps, lat 26.265 ms stddev 26.032, 0 failed
progress: 24.0 s, 198.8 tps, lat 40.247 ms stddev 32.319, 0 failed
progress: 30.0 s, 277.3 tps, lat 28.880 ms stddev 26.531, 0 failed
progress: 36.0 s, 605.0 tps, lat 13.060 ms stddev 13.134, 0 failed
progress: 42.0 s, 312.5 tps, lat 25.766 ms stddev 24.626, 0 failed
progress: 48.0 s, 575.3 tps, lat 13.976 ms stddev 15.367, 0 failed
progress: 54.0 s, 538.2 tps, lat 14.860 ms stddev 14.475, 0 failed
progress: 60.0 s, 559.7 tps, lat 14.287 ms stddev 14.270, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 25647
number of failed transactions: 0 (0.000%)
latency average = 18.707 ms
latency stddev = 20.286 ms
initial connection time = 25.541 ms
tps = 427.487633 (without initial connection time)
```

7. Что изменилось и почему?
> Количество tps изменилось и стало более неравномерным от теста к тесту. \
> Если в первом тесте tps колебались в диапазоне от 500 до 600, \
> то после применния настроек появились просадки до 200-300 tps
> 
> Предполагаю, что более агрессивные настройки автовакуума спровоцировали \
> деградацию производительность системы

8. Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк

```shell
yc-user@otus-postgres-1:~$ sudo -u postgres psql
psql (18.0 (Ubuntu 18.0-1.pgdg24.04+3))
Type "help" for help.
```
```
postgres=# CREATE TABLE book( id serial, name char(100), author char (100) );
CREATE TABLE
postgres=# INSERT INTO book(name) SELECT 'noname' FROM generate_series(1,1000000);
INSERT 0 1000000
```

9. Посмотреть размер файла с таблицей
```
postgres=# SELECT pg_size_pretty(pg_total_relation_size('book'));
 pg_size_pretty
----------------
 135 MB
(1 row)

```

10. 5 раз обновить все строчки и добавить к каждой строчке любой символ

```
postgres=# UPDATE book SET name = name || '_1';
UPDATE 1000000

postgres=# UPDATE book SET name = name || '_2';
UPDATE 1000000

postgres=# UPDATE book SET name = name || '_3';
UPDATE 1000000

postgres=# UPDATE book SET name = name || '_4';
UPDATE 1000000

postgres=# UPDATE book SET name = name || '_5';
UPDATE 1000000
```

```
postgres=# select name from book limit 1;
                                                 name
------------------------------------------------------------------------------------------------------
 noname_1_2_3_4_5
(1 row)
```
11. Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум

```
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float AS "ratio%", last_autovacuum FROM pg_stat_user_tables WHERE relname = 'book';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 book    |    1008287 |          0 |      0 | 2025-11-03 13:25:39.489439+00
(1 row)
```

12. Подождать некоторое время, проверяя, пришел ли автовакуум
> Видимо, автовакуум уже прошел, поэтому n_dead_tup = 0

13. 5 раз обновить все строчки и добавить к каждой строчке любой символ

```
postgres=# UPDATE book SET name = name || '_1';
UPDATE 1000000
postgres=# UPDATE book SET name = name || '_2';
UPDATE 1000000
postgres=# UPDATE book SET name = name || '_3';
UPDATE 1000000
postgres=# UPDATE book SET name = name || '_4';
UPDATE 1000000
postgres=# UPDATE book SET name = name || '_5';
UPDATE 1000000
```

14. Посмотреть размер файла с таблицей

> Смотрим размер файла с таблицей
```postgres=# SELECT pg_size_pretty(pg_total_relation_size('book'));
pg_size_pretty
----------------
522 MB
(1 row)
```

> Ради интереса проверям работу автовакуума и на этот раз n_dead_tup не равно 0\
> Автовакуум еще не успел отработать с предыдущего раза, что видно\
> по не изменившемуся времени последнего срабатывания 13:35:50.090296+00
```
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float AS "ratio%", last_autovacuum FROM pg_stat_user_tables WHERE relname = 'book';
relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
book    |    1901953 |      56008 |      2 | 2025-11-03 13:35:50.090296+00
(1 row)
```

> Через некоторое время проверяем срабатывание автовакуума еще раз и в этот раз видно что он сработал\
> количество n_dead_tup = 0 и изменилось времени последнего срабатывания 13:36:43.667627+00
```
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float AS "ratio%", last_autovacuum FROM pg_stat_user_tables WHERE relname = 'book';
relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
book    |    1225375 |          0 |      0 | 2025-11-03 13:36:43.667627+00
(1 row)
```

15. Отключить автовакуум на конкретной таблице

```
postgres=# ALTER TABLE book SET (autovacuum_enabled = off);
ALTER TABLE
```

16. 10 раз обновить все строчки и добавить к каждой строчке любой символ

> Для последующего сразвнения запоминаем размер таблицы
```
postgres=# SELECT pg_size_pretty(pg_total_relation_size('book'));
 pg_size_pretty
----------------
 522 MB
(1 row)
```

> Обновляем строки
```
postgres=# UPDATE book SET name = name || '_1';
UPDATE 1000000
postgres=# UPDATE book SET name = name || '_2';
UPDATE 1000000
postgres=# UPDATE book SET name = name || '_3';
UPDATE 1000000
postgres=# UPDATE book SET name = name || '_4';
UPDATE 1000000
postgres=# UPDATE book SET name = name || '_5';
UPDATE 1000000
postgres=# UPDATE book SET name = name || '_6';
UPDATE 1000000
postgres=# UPDATE book SET name = name || '_7';
UPDATE 1000000
postgres=# UPDATE book SET name = name || '_8';
UPDATE 1000000
postgres=# UPDATE book SET name = name || '_9';
UPDATE 1000000
postgres=# UPDATE book SET name = name || '_10';
UPDATE 1000000
```

17. Посмотреть размер файла с таблицей

```
postgres=# SELECT pg_size_pretty(pg_total_relation_size('book'));
 pg_size_pretty
----------------
 1482 MB
(1 row)
```

18. Объясните полученный результат

> В модели работы параллельными изменениями MVCC каждое изменение есть создание еще одной версии данных
> что на физическом уровне реализуется через дублирование данных измененных кортежей

19. Не забудьте включить автовакуум)

> Включаем
```
postgres=# ALTER TABLE book SET (autovacuum_enabled = on);
ALTER TABLE
```

> Проверяем что сработал
```
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float AS "ratio%", last_autovacuum FROM pg_stat_user_tables WHERE relname = 'book';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 book    |    1225375 |    9998111 |    815 | 2025-11-03 13:36:43.667627+00
(1 row)
```

```
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float AS "ratio%", last_autovacuum FROM pg_stat_user_tables WHERE relname = 'book';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 book    |    1020599 |          0 |      0 | 2025-11-03 13:56:40.707065+00
(1 row)
```

20. Задание со *:
- Написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в искомой таблице.
- Не забыть вывести номер шага цикла.