1. Развернуть PostgreSQL (ВМ/Docker)

```shell
# создаем ВМ через cli Я.Облако
yc compute instance create \
  --name otus-postgres-hw13-1 \
  --hostname otus-postgres-hw13-1 \
  --cores 2 \
  --core-fraction 50 \
  --memory 2 \
  --create-boot-disk size=10G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2404-lts \
  --network-interface subnet-name=default-ru-central1-b,nat-ip-version=ipv4 \
  --ssh-key ~/.ssh/ssh-key-postgres.pub
```

```shell
# Ставим Postgres, пользуясь apt
yc-user@otus-postgres-hw13-1:~$ sudo apt update && sudo apt upgrade -y -q && \
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && \
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && \
sudo apt-get update && \
sudo apt -y install postgresql

# Проверяем что кластер запущен
yc-user@otus-postgres-hw13-1:~$ crt
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 online postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```

2. В БД test_db создать схему my_schema и две одинаковые таблицы (table1, table2).
```shell
yc-user@otus-postgres-hw13-1:~$ sudo -u postgres psql -c "create database test_db;"
CREATE DATABASE
yc-user@otus-postgres-hw13-1:~$ sudo -u postgres psql -d test_db -c "create schema my_schema;"
CREATE SCHEMA
yc-user@otus-postgres-hw13-1:~$ sudo -u postgres psql -d test_db -c "create table my_schema.table1 (name character(10));"
CREATE TABLE
yc-user@otus-postgres-hw13-1:~$ sudo -u postgres psql -d test_db -c "create table my_schema.table2 (name character(10));"
CREATE TABLE
```

3. Заполнить table1 100 строками с помощью generate_series.
```shell
# Генерируем данные
yc-user@otus-postgres-hw13-1:~$ sudo -u postgres psql -d test_db -c "insert into my_schema.table1 select generate_series(1,100)::char(10);"
INSERT 0 100

# Проверяем что данные сформированы
yc-user@otus-postgres-hw13-1:~$ sudo -u postgres psql -d test_db -c "select count(*) from my_schema.table1;"
 count
-------
   100
(1 row)
```

4. Создать каталог /var/lib/postgresql/backups/ под пользователем postgres
```shell
yc-user@otus-postgres-hw13-1:~$ sudo -u postgres mkdir /var/lib/postgresql/backups/
```

5. Бэкап через COPY: Выгрузить table1 в CSV командой \copy.
```shell
yc-user@otus-postgres-hw13-1:~$ sudo -u postgres psql -d test_db -c "\copy my_schema.table1 to /var/lib/postgresql/backups/table1.csv with (format csv, header)"
COPY 100
```

6. Восстановление из COPY: Загрузить данные из CSV в table2.
```shell
# Проверяем что до загрузки данных нет 
yc-user@otus-postgres-hw13-1:~$ sudo -u postgres psql -d test_db -c "select count(*) from my_schema.table2;"
 count
-------
     0
(1 row)
# Загружаем данные из файла
yc-user@otus-postgres-hw13-1:~$ sudo -u postgres psql -d test_db -c "\copy my_schema.table2 from /var/lib/postgresql/backups/table1.csv delimiter ',' csv header"
COPY 100
# Проверяем что данные появились 
yc-user@otus-postgres-hw13-1:~$ sudo -u postgres psql -d test_db -c "select count(*) from my_schema.table2;"
 count
-------
   100
(1 row)
```

7. Бэкап через pg_dump: Создать кастомный сжатый дамп (-Fc) только схемы my_schema:
```shell
yc-user@otus-postgres-hw13-1:~$ sudo -u postgres pg_dump -Fc --schema my_schema test_db > /tmp/my_schema.dump
```
8. Восстановление через pg_restore: В новую БД restored_db восстановить только table2 из дампа
Важно: Предварительно создать схему my_schema в restored_db.
```shell
# Создаем БД
yc-user@otus-postgres-hw13-1:~$ sudo -u postgres psql -c "create database restore_db;"
CREATE DATABASE
# Создаем схему
yc-user@otus-postgres-hw13-1:~$ sudo -u postgres psql -d restore_db -c "create schema my_schema;"
CREATE SCHEMA
# Восстанавливаем только таблицу table2
yc-user@otus-postgres-hw13-1:~$ sudo -u postgres pg_restore -d restore_db -t table2 /tmp/my_schema.dump
# Проверяем что данные восстановились
yc-user@otus-postgres-hw13-1:~$ sudo -u postgres psql -d restore_db -c "select count(*) from my_schema.table2;"
 count
-------
   100
(1 row)
```