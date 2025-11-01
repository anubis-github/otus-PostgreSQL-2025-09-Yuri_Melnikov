1. Развернуть виртуальную машину любым удобным способом
```shell
# создаем ВМ через cli Я.Облако
> yc compute instance create \
  --name postgres-4-4-20-hdd-1 \
  --hostname postgres-4-4-20-hdd-1 \
  --cores 4 \
  --core-fraction 100 \
  --memory 4 \
  --create-boot-disk size=20G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2404-lts \
  --network-interface subnet-name=default-ru-central1-b,nat-ip-version=ipv4 \
  --ssh-key ~/.ssh/ssh-key-postgres.pub
  
# Проверяем что создалась
> yc compute instance list
+----------------------+-----------------------+---------------+---------+----------------+-------------+
|          ID          |         NAME          |    ZONE ID    | STATUS  |  EXTERNAL IP   | INTERNAL IP |
+----------------------+-----------------------+---------------+---------+----------------+-------------+
| epdnjb25tl7ahgil1h0f | postgres-4-4-20-hdd-1 | ru-central1-b | RUNNING | 158.160.27.229 | 10.129.0.30 |
+----------------------+-----------------------+---------------+---------+----------------+-------------+  
```

2. Поставить на неё PostgreSQL 18 любым способом
```shell
# Ставим Postgres, пользуясь apt
yc-user@postgres-4-4-20-hdd-1:~$  sudo apt update && sudo apt upgrade -y -q && \
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && \
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && \
sudo apt-get update && \
sudo apt -y install postgresql

# Запускаем cli клиента Postgres
yc-user@postgres-4-4-20-hdd-1:~$ sudo -u postgres psql
psql (18.0 (Ubuntu 18.0-1.pgdg24.04+3))
Type "help" for help.

postgres=#
```

```
# Меняем пароль у пользователя postgres
postgres=# \password
Enter new password for user "postgres":
Enter it again:
postgres=#

# Меняем конфигурацию так, чтобы Postgres слушал входящие соединения не только на 
# loopback интерфейсе, но и на всех других IPv4
postgres=# alter system set listen_addresses = '*';
ALTER SYSTEM

# Проверяем настройку
postgres=# select name, setting from pg_settings where name = 'listen_addresses';
name       |  setting
------------------+-----------
listen_addresses | localhost
(1 row)
```

```shell
# Редактируем pg_hba.conf и прописываем c каких ip адресов к каким БД какие пользователи могут подключаться
# host    all             all             0.0.0.0/0               scram-sha-256 
yc-user@postgres-4-4-20-hdd-1:~$ sudo nano /etc/postgresql/18/main/pg_hba.conf

# Перезапускаем кластер, чтобы изменения в конфигурации применились
yc-user@postgres-4-4-20-hdd-1:~$ sudo pg_ctlcluster 18 main restart
```

```shell
# Узнаем внешний IP адрес нашей ВМ
❯ yc compute instances list
+----------------------+-----------------------+---------------+---------+----------------+-------------+
|          ID          |         NAME          |    ZONE ID    | STATUS  |  EXTERNAL IP   | INTERNAL IP |
+----------------------+-----------------------+---------------+---------+----------------+-------------+
| epdnjb25tl7ahgil1h0f | postgres-4-4-20-hdd-1 | ru-central1-b | RUNNING | 89.169.183.199 | 10.129.0.30 |
+----------------------+-----------------------+---------------+---------+----------------+-------------+

# Подключаемся клиентом psql с локальной машины к установленному на ВМ Postgres
❯ psql -p 5432 -U postgres -h 89.169.183.199
Пароль пользователя postgres:
psql (18.0 (Homebrew))
SSL-соединение (протокол: TLSv1.3, шифр: TLS_AES_256_GCM_SHA384, сжатие: выкл., ALPN: postgresql)
Введите "help", чтобы получить справку.

postgres=#
```

> Перед началом тестов выводим настройки, которые будем менять для увеличения производительности
```
postgres=# select name, setting, category, context, source from pg_settings where name = 'synchronous_commit' OR name = 'fsync' OR name = 'work_mem';
name        | setting |          category          | context |       source
--------------------+---------+----------------------------+---------+--------------------
fsync              | on      | Write-Ahead Log / Settings | sighup  | configuration file
synchronous_commit | on      | Write-Ahead Log / Settings | user    | configuration file
work_mem           | 4096    | Resource Usage / Memory    | user    | session
(3 строки)
```

```shell
# Инициализируем pgbench для БД postgres и пользователя postgres
yc-user@postgres-4-4-20-hdd-1:~$ pgbench -d postgres -h localhost -U postgres -i
Password:
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
vacuuming...
creating primary keys...
done in 0.29 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.17 s, vacuum 0.06 s, primary keys 0.05 s).
```

```shell
# Запускаем pgbench чтобы понять базовую производительность БД без дополнительных настроек
yc-user@postgres-4-4-20-hdd-1:~$ pgbench -i -d postgres -h localhost -U postgres -c 50 -j 2 -P 10 -T 60
Password:
pgbench (18.0 (Ubuntu 18.0-1.pgdg24.04+3))
starting vacuum...end.
progress: 10.0 s, 414.7 tps, lat 111.649 ms stddev 148.736, 0 failed
progress: 20.0 s, 480.8 tps, lat 102.670 ms stddev 130.225, 0 failed
progress: 30.0 s, 402.9 tps, lat 126.115 ms stddev 158.027, 0 failed
progress: 40.0 s, 499.8 tps, lat 99.724 ms stddev 123.369, 0 failed
progress: 50.0 s, 464.5 tps, lat 106.152 ms stddev 127.756, 0 failed
progress: 60.0 s, 395.8 tps, lat 127.897 ms stddev 178.984, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 26635
number of failed transactions: 0 (0.000%)
latency average = 111.701 ms
latency stddev = 144.817 ms
initial connection time = 601.150 ms
tps = 447.091193 (without initial connection time)
```

3. Настроить кластер PostgreSQL 18 на максимальную производительность не обращая внимание на возможные проблемы с надежностью в случае аварийной перезагрузки виртуальной машины

> Настраиваем кластер на максимальную производительность с помощью изменения следующих параметров:
>  - fsync - отвечает за принудительную синхронизацию данных с диском после каждой транзакции
>  - synchronous_commit - определяет, ожидает ли сервер физической записи транзакционных журналов (WAL) на диск перед тем, как сообщить клиенту об успехе.
> 
```
postgres=# alter system set fsync='0';
ALTER SYSTEM
postgres=# alter system set synchronous_commit='0';
ALTER SYSTEM
postgres=# select pg_reload_conf();
pg_reload_conf
----------------
t
(1 строка)
```
> Проверяем что параметры конфигурации fsync и synchronous_commit выключены
```
postgres=# select name, setting, category, context, source from pg_settings where name = 'synchronous_commit' OR name = 'fsync';\
name        | setting |          category          | context |       source\
--------------------+---------+----------------------------+---------+--------------------\
fsync              | off     | Write-Ahead Log / Settings | sighup  | configuration file\
synchronous_commit | off     | Write-Ahead Log / Settings | user    | configuration file\
(3 строки)
```

4. Нагрузить кластер через утилиту через утилиту pgbench (https://postgrespro.ru/docs/postgrespro/14/pgbench)

> Запускам pgbench, имитирующий нагрузку, создаваемую:
> - 50 клиентами (-с)
> - в 2 рабочих потока (-j)
> - с выведением отчета каждые 10 секунд (-P)
> - в течение 60 секунд (-T)
```shell
yc-user@postgres-4-4-20-hdd-1:~$ pgbench -d postgres -h localhost -U postgres -c 50 -j 2 -P 10 -T 60
Password:
pgbench (18.0 (Ubuntu 18.0-1.pgdg24.04+3))
starting vacuum...end.
progress: 10.0 s, 1702.7 tps, lat 27.587 ms stddev 37.898, 0 failed
progress: 20.0 s, 1793.5 tps, lat 27.885 ms stddev 36.036, 0 failed
progress: 30.0 s, 1734.4 tps, lat 28.838 ms stddev 37.789, 0 failed
progress: 40.0 s, 1691.8 tps, lat 29.460 ms stddev 38.075, 0 failed
progress: 50.0 s, 1680.8 tps, lat 29.811 ms stddev 39.135, 0 failed
progress: 60.0 s, 1660.6 tps, lat 30.116 ms stddev 40.263, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 102688
number of failed transactions: 0 (0.000%)
latency average = 28.951 ms
latency stddev = 38.217 ms
initial connection time = 558.538 ms
tps = 1725.633195 (without initial connection time)
```

5. Написать какого значения tps удалось достичь, показать какие параметры в какие значения устанавливали и почему

> Получилось достичь показателя 1725 tps вместо 447 tps до изменения настроек.\
> 
> Устанавливались в off параметры fsync и synchronous_commit. \
> 
> Так как при выключении этих параметров Postgres перестает \
> принудительно записывать на диск данные по завершению каждой транзакции, \
> и дожидаться записи данных в WAL файл, то ожидаемо что количество производимых \
> транзакций в секунду увеличилось.
> 
> За эту скорость мы заплатили риском потери наших данных в случае отказа. 
> Часть данных, которые были изменены в результате транзакции 
> могут быть не записаны на диск из оперативной памяти и, как следствие,
> утеряны при сбое.

6. Задание со *: аналогично протестировать через утилиту https://github.com/Percona-Lab/sysbench-tpcc (требует установки
https://github.com/akopytov/sysbench)