1. Установка PostgreSQL (на каждой ноде): 
```shell
sudo apt update && sudo apt upgrade -y -q && \
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && \
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && \
sudo apt-get update && \
sudo apt -y install postgresql
```

```shell
sudo pg_lsclusters
```

2. Создать пользователя replicator: 
```shell
sudo -u postgres psql
```

```postgresql
-- Пароль будет использоваться в конфиге Patroni
create user replicator replication login encrypted password 'replicator';

-- Пароль будет использоваться в конфиге Patroni
alter user postgres with password 'postgres';

-- Пароль будет использоваться в конфиге Patroni
-- ALTER USER rewind_user WITH PASSWORD 'rewind_user';
```

3. Отредактировать файл pg_hba.conf:
> Командой
```shell
sudo nano /etc/postgresql/18/main/pg_hba.conf
```
> добавить фрагмент
```shell
# HA cluster replication
host    all             all             0.0.0.0/0               scram-sha-256
host    replication     replicator      0.0.0.0/0               scram-sha-256
```

4. Отредактировать файл postgresql.conf:
> Командой
```shell
sudo nano /etc/postgresql/18/main/postgresql.conf
```
> добавить фрагмент
```shell
listen_addresses = '*'                  # what IP address(es) to listen on;
```
 
5. Перезапустить PostgreSql: 
```shell
sudo systemctl restart postgresql
```

6. На второй ноде остановить Postgres и удалить содержимое каталога /var/lib/postgresql/18/main/
```shell
sudo systemctl stop postgresql 
# почему-то эта команда не очищала каталог sudo rm -rf /var/lib/postgresql/18/main/*
sudo -su postgres
rm -rf /var/lib/postgresql/18/main/*
ls -l /var/lib/postgresql/18/main/
```

> Диагностика, которой пользовался для исследования проблем
```shell
sudo journalctl -xeu postgresql@18-main.service
sudo tail -n 20 /var/log/postgresql/postgresql-18-main.log
```