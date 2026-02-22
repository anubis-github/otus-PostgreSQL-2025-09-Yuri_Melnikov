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
```
> Для ВМ2 и ВМ3 - аналогично

1. Настройте ВМ1:
> Создайте таблицу test, которая будет для операций записи
> Создайте таблицу test2, которая будет для чтения
> Настройте публикацию таблицы test

2. Настройте ВМ2:
> Создайте таблицу test2, которая будет для операций записи
> Создайте таблицу test, которая будет для чтения
> Настройте публикацию таблицы test2
> Сделайте подписку таблицы test на публикацию таблицы test с ВМ1

3. на ВМ1:
> Сделайте подписку таблицы test2 на публикацию таблицы test2 с ВМ2

4. Настройте ВМ3:

> Создайте таблицы: test и test2
> Подпишите test на публикацию таблицы test с ВМ1
> Подпишите test2 на публикацию таблицы test2 с ВМ2
> Используйте этот узел для чтения объединённых данных и резервного копирования
> Проверьте работу системы:

> Выполните вставку в test на ВМ1 — убедитесь, что данные появились в test на ВМ2 и ВМ3
> Выполните вставку в test2 на ВМ2 — убедитесь, что данные появились в test2 на ВМ1 и ВМ3

5. Задание повышенной сложности(*):
> Настройте физическую репликацию с ВМ4, используя ВМ3 в качестве источника.