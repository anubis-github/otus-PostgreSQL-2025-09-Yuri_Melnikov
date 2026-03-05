0. Создаем новую ВМ с параметрами как в предыдущем задании и заново ставим Postgres

```shell
# создаем ВМ через cli Я.Облако
yc compute instance create \
  --name otus-postgres-hw12-1 \
  --hostname otus-postgres-hw12-1 \
  --cores 2 \
  --core-fraction 50 \
  --memory 2 \
  --create-boot-disk size=10G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2404-lts \
  --network-interface subnet-name=default-ru-central1-b,nat-ip-version=ipv4 \
  --ssh-key ~/.ssh/ssh-key-postgres.pub
```

```shell
# Ставим Postgres, пользуясь apt
yc-user@otus-postgres-hw12-1:~$ sudo apt update && sudo apt upgrade -y -q && \
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && \
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && \
sudo apt-get update && \
sudo apt -y install postgresql

# Проверяем что кластер запущен
yc-user@otus-postgres-hw12-1:~$ sudo pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 online postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```

1. Скрипт и развернутое описание задачи – в ЛК (файл hw_triggers.sql) или по ссылке: https://disk.yandex.ru/d/l70AvknAepIJXQ
В БД создана структура, описывающая товары (таблица goods) и продажи (таблица sales).
Есть запрос для генерации отчета – сумма продаж по каждому товару.
БД была денормализована, создана таблица (витрина), структура которой повторяет структуру отчета.

```shell
# Развернем БД на ВМ из файла hw_triggers.sql
# Копируем файл hw_triggers.sql c локальной машины на ВМ с Postgres чтобы не настраивать подключение к Postgres c локальной машины
❯ scp -i ~/.ssh/ssh-key-postgres ~/Documents/obsidian-vaults/IT/Education/DBs/Postgres/OTUS/lesson-16\ -\ procedures\ and\ functions/scripts/hw_triggers.sql yc-user@51.250.20.230:/home/yc-user/hw_triggers.sql
hw_triggers.sql                              100% 1928    83.5KB/s   00:00

#
yc-user@otus-postgres-hw12-1:~$ sudo chown postgres:postgres hw_triggers.sql

#
yc-user@otus-postgres-hw12-1:~$ sudo -u postgres psql -c "alter user postgres with password 'postgres';"
ALTER ROLE

# Пытаемся создать БД скриптом из файла и падаем с ошибкой. Удобно, что в консоль выводится лог выполнения и диагностика об ошибках
yc-user@otus-postgres-hw12-1:~$ psql -U postgres -h localhost -p 5432 -f ~/hw_triggers.sql
Password for user postgres:
psql:/home/yc-user/hw_triggers.sql:3: NOTICE:  drop cascades to 3 other objects
DETAIL:  drop cascades to table pract_functions.goods
drop cascades to table pract_functions.sales
drop cascades to table pract_functions.good_sum_mart
DROP SCHEMA
CREATE SCHEMA
psql:/home/yc-user/hw_triggers.sql:14: ERROR:  syntax error at or near "CREATE"
LINE 4: CREATE TABLE goods
        ^
psql:/home/yc-user/hw_triggers.sql:17: ERROR:  relation "goods" does not exist
LINE 1: INSERT INTO goods (goods_id, good_name, good_price)
                    ^
psql:/home/yc-user/hw_triggers.sql:26: ERROR:  relation "goods" does not exist
psql:/home/yc-user/hw_triggers.sql:28: ERROR:  relation "sales" does not exist
LINE 1: INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1...
                    ^
psql:/home/yc-user/hw_triggers.sql:34: ERROR:  relation "goods" does not exist
LINE 2: FROM goods G
             ^
CREATE TABLE

# Правим скрипт (; после "SET search_path = pract_functions, publ") и запускаем снова, теперь успешно
yc-user@otus-postgres-hw12-1:~$ psql -U postgres -h localhost -p 5432 -f ~/hw_triggers.sql
Password for user postgres:
DROP SCHEMA
CREATE SCHEMA
SET
CREATE TABLE
INSERT 0 2
CREATE TABLE
INSERT 0 4
        good_name         |     sum
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)

CREATE TABLE
```

2. Создать триггер на таблице продаж, для поддержки данных в витрине в актуальном состоянии (вычисляющий при каждой продаже сумму и записывающий её в витрину). Подсказка: не забыть, что кроме INSERT есть еще UPDATE и DELETE

> Все работы в этом пункте делаем в DBeaver так как с таким количеством кода неудобно работать из CLI

> Пересоздадим таблицу pract_functions.good_sum_mart - добавим поле total_qty для проверки и unique ограничение на good_name и переинициализируем ее
```postgresql
drop table if exists pract_functions.good_sum_mart cascade;

create table pract_functions.good_sum_mart
(
    good_name   varchar(63) not null,
    sum_sale    numeric(16, 2) not null,
    total_qty   integer not null default 0, -- добавим для проверки
    unique(good_name)
);

insert into pract_functions.good_sum_mart (good_name, sum_sale, total_qty)
select 
    g.good_name, 
    sum(g.good_price * s.sales_qty) as sum_sale,
    sum(s.sales_qty) as total_qty
from pract_functions.goods g
join pract_functions.sales s on s.good_id = g.goods_id
group by g.good_name;
```

> Создадим функцию триггера
```postgresql
create or replace function pract_functions.maintain_good_sum_mart()
returns trigger as $$
declare
    v_good_name varchar(63);
    v_price_diff numeric(16, 2);
    v_qty_diff integer;
begin
    -- получаем название товара (оно не меняется)
    select good_name into v_good_name
    from pract_functions.goods
    where goods_id = coalesce(new.good_id, old.good_id);

    -- обработка разных операций
    if tg_op = 'INSERT' then
        -- при вставке: добавляем сумму продажи
        insert into pract_functions.good_sum_mart (good_name, sum_sale, total_qty)
        values (
            v_good_name, 
            (select good_price from pract_functions.goods where goods_id = new.good_id) * new.sales_qty,
            new.sales_qty
        )
        on conflict (good_name) do update
        set 
            sum_sale = good_sum_mart.sum_sale + excluded.sum_sale,
            total_qty = good_sum_mart.total_qty + excluded.total_qty;

    elsif tg_op = 'DELETE' then
        -- при удалении: вычитаем сумму продажи
        update pract_functions.good_sum_mart 
        set 
            sum_sale = sum_sale - (
                select good_price from pract_functions.goods where goods_id = old.good_id
            ) * old.sales_qty,
            total_qty = total_qty - old.sales_qty
        where good_name = v_good_name;

    elsif tg_op = 'UPDATE' then
        -- при обновлении: корректируем разницу
        if old.good_id != new.good_id then
            -- товар изменился - это сложный случай, лучше запретить
            raise exception 'cannot change good_id in sales, delete and insert instead';
        end if;
        
        -- корректируем только если изменилось количество
        if old.sales_qty != new.sales_qty then
            v_price_diff = (select good_price from pract_functions.goods where goods_id = new.good_id);
            v_qty_diff = new.sales_qty - old.sales_qty;
            
            update pract_functions.good_sum_mart 
            set 
                sum_sale = sum_sale + (v_price_diff * v_qty_diff),
                total_qty = total_qty + v_qty_diff
            where good_name = v_good_name;
        end if;
    end if;

    -- удаляем записи с нулевыми продажами
    delete from pract_functions.good_sum_mart where total_qty = 0;

    return coalesce(new, old);
end;
$$ language plpgsql;
```

> Создаем триггеры
```postgresql
create trigger trg_sales_insert
    after insert on pract_functions.sales
    for each row
    execute function pract_functions.maintain_good_sum_mart();

create trigger trg_sales_update
    after update of sales_qty on pract_functions.sales
    for each row
    execute function pract_functions.maintain_good_sum_mart();

create trigger trg_sales_delete
    after delete on pract_functions.sales
    for each row
    execute function pract_functions.maintain_good_sum_mart();
```

> Проверяем что получилось
```shell
# состояние таблицы goods на начало теста
yc-user@otus-postgres-hw12-1:~$ sudo -u postgres psql -c "select * from pract_functions.goods;"
 goods_id |        good_name         |  good_price
----------+--------------------------+--------------
        2 | Автомобиль Ferrari FXX K | 185000000.01
        1 | Спички хозайственные     |         0.50
(2 rows)

# состояние таблицы sales на начало теста
yc-user@otus-postgres-hw12-1:~$ sudo -u postgres psql -c "select * from pract_functions.sales;"
 sales_id | good_id |          sales_time          | sales_qty
----------+---------+------------------------------+-----------
       34 |       1 | 2026-03-05 15:53:31.32634+00 |        10
       35 |       1 | 2026-03-05 15:53:31.32634+00 |         1
       36 |       1 | 2026-03-05 15:53:31.32634+00 |       120
       37 |       2 | 2026-03-05 15:53:31.32634+00 |         1
(4 rows)

# состояние витрины на начало теста
yc-user@otus-postgres-hw12-1:~$ sudo -u postgres psql -c "select * from pract_functions.good_sum_mart;"
        good_name         |   sum_sale   | total_qty
--------------------------+--------------+-----------
 Автомобиль Ferrari FXX K | 185000000.01 |         1
 Спички хозайственные     |        65.50 |       131
(2 rows)

# новая продажа 
yc-user@otus-postgres-hw12-1:~$ sudo -u postgres psql -c "insert into pract_functions.sales (good_id, sales_qty) values (1, 5);"
INSERT 0 1

# состояние витрины после теста вставки
yc-user@otus-postgres-hw12-1:~$ sudo -u postgres psql -c "select * from pract_functions.good_sum_mart;"
        good_name         |   sum_sale   | total_qty
--------------------------+--------------+-----------
 Автомобиль Ferrari FXX K | 185000000.01 |         1
 Спички хозайственные     |        68.00 |       136
(2 rows)

# изменение цены товара "Спички хозайственные"
yc-user@otus-postgres-hw12-1:~$ sudo -u postgres psql -c "update pract_functions.goods set good_price = 1.00 where goods_id = 1;"
UPDATE 1

# проверка - сумма не изменилась
yc-user@otus-postgres-hw12-1:~$ sudo -u postgres psql -c "select * from pract_functions.good_sum_mart;"
        good_name         |   sum_sale   | total_qty
--------------------------+--------------+-----------
 Автомобиль Ferrari FXX K | 185000000.01 |         1
 Спички хозайственные     |        68.00 |       136
(2 rows)

# новая продажа по новой цене
yc-user@otus-postgres-hw12-1:~$ sudo -u postgres psql -c "insert into pract_functions.sales (good_id, sales_qty) values (1, 3);"
INSERT 0 1

# проверка - добавилась продажа по новой цене
yc-user@otus-postgres-hw12-1:~$ sudo -u postgres psql -c "select * from pract_functions.good_sum_mart;"
        good_name         |   sum_sale   | total_qty
--------------------------+--------------+-----------
 Автомобиль Ferrari FXX K | 185000000.01 |         1
 Спички хозайственные     |        71.00 |       139
(2 rows)

# обновляем количество товара в существующей записи о продаже - смотрим sales_id для обновления
yc-user@otus-postgres-hw12-1:~$ sudo -u postgres psql -c "select * from pract_functions.sales s;"
 sales_id | good_id |          sales_time           | sales_qty
----------+---------+-------------------------------+-----------
       34 |       1 | 2026-03-05 15:53:31.32634+00  |        10
       35 |       1 | 2026-03-05 15:53:31.32634+00  |         1
       36 |       1 | 2026-03-05 15:53:31.32634+00  |       120
       37 |       2 | 2026-03-05 15:53:31.32634+00  |         1
       38 |       1 | 2026-03-05 15:55:43.187372+00 |         5
       39 |       1 | 2026-03-05 16:01:40.188302+00 |         3
(6 rows)

# обновляем количество товара в существующей записи о продаже - обновляем количество с 3 на 7 для записи с sales_id = 39
yc-user@otus-postgres-hw12-1:~$ sudo -u postgres psql -c "update pract_functions.sales s set sales_qty = 7 where s.sales_id = 39;"
UPDATE 1

# проверяем результат обновления в sales
yc-user@otus-postgres-hw12-1:~$ sudo -u postgres psql -c "select * from pract_functions.sales s;"
 sales_id | good_id |          sales_time           | sales_qty
----------+---------+-------------------------------+-----------
       34 |       1 | 2026-03-05 15:53:31.32634+00  |        10
       35 |       1 | 2026-03-05 15:53:31.32634+00  |         1
       36 |       1 | 2026-03-05 15:53:31.32634+00  |       120
       37 |       2 | 2026-03-05 15:53:31.32634+00  |         1
       38 |       1 | 2026-03-05 15:55:43.187372+00 |         5
       39 |       1 | 2026-03-05 16:01:40.188302+00 |         7
(6 rows)

# проверяем результат обновления в good_sum_mart: total_qty изменилось на 4 и sum_sale на 4
yc-user@otus-postgres-hw12-1:~$ sudo -u postgres psql -c "select * from pract_functions.good_sum_mart;"
        good_name         |   sum_sale   | total_qty
--------------------------+--------------+-----------
 Автомобиль Ferrari FXX K | 185000000.01 |         1
 Спички хозайственные     |        75.00 |       143
(2 rows)

# удаляем существующую запись о продаже с sales_id = 39
yc-user@otus-postgres-hw12-1:~$ sudo -u postgres psql -c "delete from pract_functions.sales where sales_id = 39;"
DELETE 1

# проверяем результат удаления в sales
yc-user@otus-postgres-hw12-1:~$ sudo -u postgres psql -c "select * from pract_functions.sales s;"
 sales_id | good_id |          sales_time           | sales_qty
----------+---------+-------------------------------+-----------
       34 |       1 | 2026-03-05 15:53:31.32634+00  |        10
       35 |       1 | 2026-03-05 15:53:31.32634+00  |         1
       36 |       1 | 2026-03-05 15:53:31.32634+00  |       120
       37 |       2 | 2026-03-05 15:53:31.32634+00  |         1
       38 |       1 | 2026-03-05 15:55:43.187372+00 |         5
(5 rows)

# проверяем результат обновления в good_sum_mart: total_qty изменилось на 7 и sum_sale на 7
yc-user@otus-postgres-hw12-1:~$ sudo -u postgres psql -c "select * from pract_functions.good_sum_mart;"
        good_name         |   sum_sale   | total_qty
--------------------------+--------------+-----------
 Автомобиль Ferrari FXX K | 185000000.01 |         1
 Спички хозайственные     |        68.00 |       136
(2 rows)
```


3. Задание со звездочкой*

Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?
Подсказка: В реальной жизни возможны изменения цен.

> В общем случае, такая схема лучше тем, что сохраняет историчность данных, как с примером в подсказке. Для того, чтобы на витрине хранить историю изменений не только значений но и связей, используются схемы построения витрин такие как DataVault 2.0, AnchorModeling