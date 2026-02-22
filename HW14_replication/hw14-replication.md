0. Создаем три ВМ и ставим на них Postgres с помощью apt

```shell
# создаем ВМ через cli Я.Облако
yc compute instance create \
  --name otus-postgres-hw14-1 \
  --hostname otus-postgres-hw14-1 \
  --cores 2 \
  --core-fraction 50 \
  --memory 2 \
  --create-boot-disk size=10G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2404-lts \
  --network-interface subnet-name=default-ru-central1-b,nat-ip-version=ipv4 \
  --ssh-key ~/.ssh/ssh-key-postgres.pub
```

```shell
# Ставим Postgres, пользуясь apt
yc-user@otus-postgres-hw14-1:~$ sudo apt update && sudo apt upgrade -y -q && \
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && \
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && \
sudo apt-get update && \
sudo apt -y install postgresql

# Проверяем что кластер запущен
yc-user@otus-postgres-hw14-1:~$ sudo pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 online postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log

# Настраиваем подключение извне
yc-user@otus-postgres-hw14-2:~$ sudo cp /etc/postgresql/18/main/postgresql.conf /etc/postgresql/18/main/postgresql.conf.bak
yc-user@otus-postgres-hw14-2:~$ sudo sed -i "s/#listen_addresses = 'localhost'/listen_addresses = '*'/g" /etc/postgresql/18/main/postgresql.conf
yc-user@otus-postgres-hw14-2:~$ sudo cp /etc/postgresql/18/main/pg_hba.conf /etc/postgresql/18/main/pg_hba.conf.bak
yc-user@otus-postgres-hw14-2:~$ sudo sed -i -e $'$a\\\nhost  all  all  0.0.0.0/0  scram-sha-256' /etc/postgresql/18/main/pg_hba.conf

# Устанавливаем пароль на пользователя postgres 
yc-user@otus-postgres-hw14-1:~$ sudo -u postgres psql -c "alter user postgres with password 'postgres';"
ALTER ROLE    

# Перезапускаем кластер, чтобы применились изменения
yc-user@otus-postgres-hw14-1:~$ sudo pg_ctlcluster 18 main restart
```
> Для ВМ2 и ВМ3 - аналогично

1. Настройте ВМ1:
> Создайте таблицу test, которая будет для операций записи
```shell
yc-user@otus-postgres-hw14-1:~$ sudo -u postgres psql -c "create table test (name character(10));"
CREATE TABLE
```
> Создайте таблицу test2, которая будет для чтения
```shell
yc-user@otus-postgres-hw14-1:~$ sudo -u postgres psql -c "create table test2 (name character(10));"
CREATE TABLE
```
> Настройте публикацию таблицы test
```shell
# Включаем возможность логической репликации
yc-user@otus-postgres-hw14-1:~$ sudo -u postgres psql -c "alter system set wal_level = logical;"
ALTER SYSTEM

# Перезапускаем кластер, чтобы применились изменения
yc-user@otus-postgres-hw14-1:~$ sudo pg_ctlcluster 18 main restart

# Создаем публикацию таблицы test
yc-user@otus-postgres-hw14-1:~$ sudo -u postgres psql -c "create publication test_pub for table test;"
CREATE PUBLICATION

# Проверяем что публикация создалась
yc-user@otus-postgres-hw14-1:~$ sudo -u postgres psql -c "\dRp+"
                                      Publication test_pub
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Generated columns | Via root
----------+------------+---------+---------+---------+-----------+-------------------+----------
 postgres | f          | t       | t       | t       | t         | none              | f
Tables:
    "public.test"
```

2. Настройте ВМ2:
> Создайте таблицу test2, которая будет для операций записи
```shell
yc-user@otus-postgres-hw14-2:~$ sudo -u postgres psql -c "create table test2 (name character(10));"
CREATE TABLE
```

> Создайте таблицу test, которая будет для чтения
```shell
yc-user@otus-postgres-hw14-2:~$ sudo -u postgres psql -c "create table test (name character(10));"
CREATE TABLE
```

> Настройте публикацию таблицы test2
```shell
# Включаем возможность логической репликации
yc-user@otus-postgres-hw14-2:~$ sudo -u postgres psql -c "alter system set wal_level = logical;"
ALTER SYSTEM

# Перезапускаем кластер, чтобы применились изменения
yc-user@otus-postgres-hw14-2:~$ sudo pg_ctlcluster 18 main restart

# Создаем публикацию таблицы test2
yc-user@otus-postgres-hw14-2:~$ sudo -u postgres psql -c "create publication test2_pub for table test2;"
CREATE PUBLICATION

# Проверяем что публикация создалась
yc-user@otus-postgres-hw14-2:~$ sudo -u postgres psql -c "\dRp+"
                                     Publication test2_pub
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Generated columns | Via root
----------+------------+---------+---------+---------+-----------+-------------------+----------
 postgres | f          | t       | t       | t       | t         | none              | f
Tables:
    "public.test2"
    
# 
yc-user@otus-postgres-hw14-2:~$ sudo -u postgres psql -c "alter user postgres with password 'postgres';"
ALTER ROLE 
```

> Сделайте подписку таблицы test на публикацию таблицы test с ВМ1
```shell
yc-user@otus-postgres-hw14-2:~$ sudo -u postgres psql -c "create subscription test_sub connection 'host=otus-postgres-hw14-1 port=5432 user=postgres password=postgres' publication test_pub with (copy_data = false);"
NOTICE:  created replication slot "test_sub" on publisher
CREATE SUBSCRIPTION
yc-user@otus-postgres-hw14-2:~$ sudo -u postgres psql -c "\dRs+"
                                                                                                                        List of subscriptions
   Name   |  Owner   | Enabled | Publication | Binary | Streaming | Two-phase commit | Disable on error | Origin | Password required | Run as owner? | Failover | Synchronous commit |                              Conninfo                               | Skip LSN
----------+----------+---------+-------------+--------+-----------+------------------+------------------+--------+-------------------+---------------+----------+--------------------+---------------------------------------------------------------------+----------
 test_sub | postgres | t       | {test_pub}  | f      | parallel  | d                | f                | any    | t                 | f             | f        | off                | host=otus-postgres-hw14-1 port=5432 user=postgres password=postgres | 0/0
(1 row)

```

3. на ВМ1:
> Сделайте подписку таблицы test2 на публикацию таблицы test2 с ВМ2
```shell
yc-user@otus-postgres-hw14-1:~$ sudo -u postgres psql -c "create subscription test2_sub connection 'host=otus-postgres-hw14-2 port=5432 user=postgres password=postgres' publication test2_pub with (copy_data = false);"
NOTICE:  created replication slot "test2_sub" on publisher
CREATE SUBSCRIPTION
```

4. Настройте ВМ3:

> Создайте таблицы: test и test2
```shell
yc-user@otus-postgres-hw14-3:~$ sudo -u postgres psql -c "create table test (name character(10));"
CREATE TABLE
yc-user@otus-postgres-hw14-3:~$ sudo -u postgres psql -c "create table test2 (name character(10));"
CREATE TABLE
```
> Подпишите test на публикацию таблицы test с ВМ1
```shell
# Создаем подписку
yc-user@otus-postgres-hw14-3:~$ sudo -u postgres psql -c "create subscription test31_sub connection 'host=otus-postgres-hw14-1 port=5432 user=postgres password=postgres' publication test_pub with (copy_data = false);"
NOTICE:  created replication slot "test31_sub" on publisher
CREATE SUBSCRIPTION

# Проверяем что подписка создалась
yc-user@otus-postgres-hw14-3:~$ sudo -u postgres psql -c "\dRs+"
                                            List of subscriptions
    Name    |  Owner   | Enabled | Publication | Binary | Streaming | Two-phase commit | Disable on error | Origin | Password required | Run as owner? | Failover | Synchronous commit |                              Conninfo                               | Skip LSN
------------+----------+---------+-------------+--------+-----------+------------------+------------------+--------+-------------------+---------------+----------+--------------------+---------------------------------------------------------------------+----------
 test31_sub | postgres | t       | {test_pub}  | f      | parallel  | d                | f                | any    | t                 | f             | f        | off                | host=otus-postgres-hw14-1 port=5432 user=postgres password=postgres | 0/0
(1 row)
```

> Подпишите test2 на публикацию таблицы test2 с ВМ2
```shell
# Создаем подписку
yc-user@otus-postgres-hw14-3:~$ sudo -u postgres psql -c "create subscription test32_sub connection 'host=otus-postgres-hw14-2 port=5432 user=postgres password=postgres' publication test2_pub with (copy_data = false);"
NOTICE:  created replication slot "test32_sub" on publisher
CREATE SUBSCRIPTION

# Проверяем что подписка создалась
yc-user@otus-postgres-hw14-3:~$ sudo -u postgres psql -c "\dRs+"
                                            List of subscriptions
    Name    |  Owner   | Enabled | Publication | Binary | Streaming | Two-phase commit | Disable on error | Origin | Password required | Run as owner? | Failover | Synchronous commit |                              Conninfo                               | Skip LSN
------------+----------+---------+-------------+--------+-----------+------------------+------------------+--------+-------------------+---------------+----------+--------------------+---------------------------------------------------------------------+----------
 test31_sub | postgres | t       | {test_pub}  | f      | parallel  | d                | f                | any    | t                 | f             | f        | off                | host=otus-postgres-hw14-1 port=5432 user=postgres password=postgres | 0/0
 test32_sub | postgres | t       | {test2_pub} | f      | parallel  | d                | f                | any    | t                 | f             | f        | off                | host=otus-postgres-hw14-2 port=5432 user=postgres password=postgres | 0/0
(2 rows)
```

> Используйте этот узел для чтения объединённых данных и резервного копирования
> Проверьте работу системы:
> Выполните вставку в test на ВМ1 — убедитесь, что данные появились в test на ВМ2 и ВМ3
```shell
# Вставка данных на ВМ1
yc-user@otus-postgres-hw14-1:~$ sudo -u postgres psql -c "insert into test(name) values ('Name_1_VM1');"
INSERT 0 1

# Проверка данных на ВМ2
yc-user@otus-postgres-hw14-2:~$ sudo -u postgres psql -c "select name from test;"
    name
------------
 Name_1_VM1
(1 row)

# Проверка данных на ВМ3
yc-user@otus-postgres-hw14-3:~$ sudo -u postgres psql -c "select name from test;"
    name
------------
 Name_1_VM1
(1 row)
``` 

> Выполните вставку в test2 на ВМ2 — убедитесь, что данные появились в test2 на ВМ1 и ВМ3
```shell
# Вставка данных на ВМ2
yc-user@otus-postgres-hw14-2:~$ sudo -u postgres psql -c "insert into test2(name) values ('Name_2_VM2');"
INSERT 0 1

# Проверка данных на ВМ1
yc-user@otus-postgres-hw14-1:~$ sudo -u postgres psql -c "select name from test2;"
    name
------------
 Name_2_VM2
(1 row)

# Проверка данных на ВМ3
yc-user@otus-postgres-hw14-3:~$ sudo -u postgres psql -c "select name from test2;"
    name
------------
 Name_2_VM2
(1 row)
```

5. Задание повышенной сложности(*):
> Настройте физическую репликацию с ВМ4, используя ВМ3 в качестве источника.