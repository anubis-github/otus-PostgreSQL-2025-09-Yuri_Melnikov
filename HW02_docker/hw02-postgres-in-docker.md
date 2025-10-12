# Работа с Postgres в Docker
1. Создать ВМ с Ubuntu 20.04/22.04

> VM cоздана в UI консоли Я.облако с параметрами 2CPU, 2Gb RAM, 20Gb HDD

2. Поставить на нем Docker Engine

> Использовались команды из официальной документации docker https://docs.docker.com/engine/install/ubuntu/

3. Сделать каталог /var/lib/postgres

> postgres@postgres-vm-2-2-20-hdd:~$ sudo mkdir /var/lib/postgres

4. Развернуть контейнер с PostgreSQL 15 смонтировав в него /var/lib/postgresql

> ``#`` Запускаем контейнер pg-docker\
> postgres@postgres-vm-2-2-20-hdd:~$ sudo docker run --name pg-docker --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:17

> Unable to find image 'postgres:17' locally\
> 17: Pulling from library/postgres\
> 8c7716127147: Pull complete\
> de7816569160: Pull complete\
> 1ed9605b075e: Pull complete\
> ba692dbf22b7: Pull complete\
> d5d1559014a5: Pull complete\
> f6a82cec6433: Pull complete\
> ce1f028def8f: Pull complete\
> be8c3c303f61: Pull complete\
> 2a9905500c81: Pull complete\
> 95861f01fbc3: Pull complete\
> 9138844e74a6: Pull complete\
> 29da297fd253: Pull complete\
> 4664e556e90a: Pull complete\
> 8d6e0d53c584: Pull complete\
> Digest: sha256:e6a4209d1a4893f2df3bdcde58f8926c3c929c4d51df90990ed1b36d83c1382a\
> Status: Downloaded newer image for postgres:17\
> c058396adc070c444604939e7d4e24b86ff7661116ce2c959575e70a3720cf97

> ``#`` Проверяем что контейнер поднялся\
> postgres@postgres-vm-2-2-20-hdd:~$ sudo docker ps

> CONTAINER ID   IMAGE         COMMAND                  CREATED         STATUS         PORTS                                         NAMES\
c058396adc07   postgres:17   "docker-entrypoint.s…"   3 minutes ago   Up 3 minutes   0.0.0.0:5432->5432/tcp, [::]:5432->5432/tcp   pg-docker

5. Развернуть контейнер с клиентом postgres и сделать таблицу с парой строк
> ``#`` Запускаем контейнер pg-client\
> postgres@postgres-vm-2-2-20-hdd:~$ sudo docker run -it --rm --network pg-net --name pg-client postgres:17 psql -h pg-docker -U postgres

> Password for user postgres:\
> psql (17.6 (Debian 17.6-2.pgdg13+1))\
> Type "help" for help.
> 
> postgres=# 
 
> ``#`` Создаем таблицу t_persons\
> postgres=# create table t_persons(id serial, first_name text, second_name text);\
> CREATE TABLE\

> ``#`` Добавляем записи в t_persons\
> postgres=# insert into t_persons(first_name, second_name) values('ivan', 'ivanov');\
> INSERT 0 1
> insert into t_persons(first_name, second_name) values('petr', 'petrov');\
> INSERT 0 1

> ``#`` Проверяем записи в t_persons\
> postgres=# select * from t_persons;\
>id | first_name | second_name\
>---+--------------+-------------\
>1 | ivan       | ivanov\
>2 | petr       | petrov\
>(2 rows)

6. Подключится к контейнеру с сервером с ноутбука/компьютера извне инстансов ЯО/места установки докера

``#`` Успешно подключился с ноутбука в DBeaver строкой jdbc:postgresql://89.169.177.130:5432/postgres

7. Удалить контейнер с сервером
> ``#`` Останавливаем контейнер
> postgres@postgres-vm-2-2-20-hdd:~$ sudo docker stop pg-docker\
>pg-docker

> ``#`` Удаляем контейнер
> postgres@postgres-vm-2-2-20-hdd:~$ sudo docker rm pg-docker\
>pg-docker

8. Cоздать его заново

> ``#`` Создаем контейнер
> postgres@postgres-vm-2-2-20-hdd:~$ sudo docker run --name pg-docker --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:17\
> 7a91fe960829e838a6cc81230280e3cf12c2f4f03b105afe33783c586d354965

> ``#`` Убеждаемся что контейнер стартовал
> postgres@postgres-vm-2-2-20-hdd:~$ sudo docker ps\
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                         NAMES\
7a91fe960829   postgres:17   "docker-entrypoint.s…"   44 seconds ago   Up 44 seconds   0.0.0.0:5432->5432/tcp, [::]:5432->5432/tcp   pg-docker\
639d629da226   postgres:17   "docker-entrypoint.s…"   22 minutes ago   Up 22 minutes   5432/tcp                                      pg-client

9. Подключится снова из контейнера с клиентом к контейнеру с сервером

> ``#`` Запускаем контейнер pg-client\
> postgres@postgres-vm-2-2-20-hdd:~$ sudo docker run -it --rm --network pg-net --name pg-client postgres:17 psql -h pg-docker -U postgres\
> Password for user postgres:\
> psql (17.6 (Debian 17.6-2.pgdg13+1))\
> Type "help" for help.
>
> postgres=#

10. Проверить, что данные остались на месте

> ``#`` Проверяем записи в t_persons\
> postgres=# select * from t_persons;\
>id | first_name | second_name\
>---+--------------+-------------\
>1 | ivan       | ivanov\
>2 | petr       | petrov\
>(2 rows) 