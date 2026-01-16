0. Создаем новую ВМ с параметрами как в предыдущем задании и заново ставим Postgres

```shell
# создаем ВМ через cli Я.Облако
yc compute instance create \
  --name otus-postgres-hw11-1 \
  --hostname otus-postgres-hw11-1 \
  --cores 2 \
  --core-fraction 50 \
  --memory 4 \
  --create-boot-disk size=20G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2404-lts \
  --network-interface subnet-name=default-ru-central1-b,nat-ip-version=ipv4 \
  --ssh-key ~/.ssh/ssh-key-postgres.pub
```

```shell
# Ставим Postgres, пользуясь apt
yc-user@otus-postgres-hw11-1:~$ sudo apt update && sudo apt upgrade -y -q && \
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && \
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && \
sudo apt-get update && \
sudo apt -y install postgresql

# Проверяем что кластер запущен
yc-user@otus-postgres-hw11-1:~$ sudo pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 online postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log

# Качаем БД из https://edu.postgrespro.ru/demo-20250901-2y.sql.gz
yc-user@otus-postgres-hw11-1:~$ wget https://edu.postgrespro.ru/demo-20250901-2y.sql.gz
--2025-11-30 13:12:50--  https://edu.postgrespro.ru/demo-20250901-2y.sql.gz
Resolving edu.postgrespro.ru (edu.postgrespro.ru)... 213.171.56.196
Connecting to edu.postgrespro.ru (edu.postgrespro.ru)|213.171.56.196|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1170883248 (1.1G) [application/x-gzip]
Saving to: ‘demo-20250901-2y.sql.gz’

demo-20250901-2y.sql.gz                               100%[=======================================================================================================================>]   1.09G  79.3MB/s    in 15s

2025-11-30 13:13:05 (74.5 MB/s) - ‘demo-20250901-2y.sql.gz’ saved [1170883248/1170883248]

# Распаковываем
yc-user@otus-postgres-hw11-1:~$ gzip -d demo-20250901-2y.sql.gz
yc-user@otus-postgres-hw11-1:~$ ls -l
total 4001004
-rw-rw-r-- 1 yc-user yc-user 4097021542 Sep 21 19:23 demo-20250901-2y.sql

# Импорт БД в кластер main по инструкции со страницы https://postgrespro.ru/education/demodb
yc-user@otus-postgres-hw11-1:~$ gunzip -c demo-20250901-2y.sql.gz | sudo -u postgres psql -U postgres

# Проверяем БД
yc-user@otus-postgres-hw11-1:~$ sudo -u postgres psql
psql (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
Type "help" for help.
```
```postgresql
postgres=# \l
                                                     List of databases
   Name    |  Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | Locale | ICU Rules |   Access privileges
-----------+----------+----------+-----------------+-------------+-------------+--------+-----------+-----------------------
 demo      | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           |
 postgres  | postgres | UTF8     | libc            | C.UTF-8     | C.UTF-8     |        |           |
 template0 | postgres | UTF8     | libc            | C.UTF-8     | C.UTF-8     |        |           | =c/postgres          +
           |          |          |                 |             |             |        |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | C.UTF-8     | C.UTF-8     |        |           | =c/postgres          +
           |          |          |                 |             |             |        |           | postgres=CTc/postgres
(4 rows)

postgres=# \c demo
You are now connected to database "demo" as user "postgres".

demo=# \dt+
                                                         List of tables
  Schema  |      Name       | Type  |  Owner   | Persistence | Access method |  Size   |              Description
----------+-----------------+-------+----------+-------------+---------------+---------+----------------------------------------
 bookings | airplanes_data  | table | postgres | permanent   | heap          | 16 kB   | Airplanes (internal multilingual data)
 bookings | airports_data   | table | postgres | permanent   | heap          | 1280 kB | Airports (internal multilingual data)
 bookings | boarding_passes | table | postgres | permanent   | heap          | 1713 MB | Boarding passes
 bookings | bookings        | table | postgres | permanent   | heap          | 484 MB  | Bookings
 bookings | flights         | table | postgres | permanent   | heap          | 11 MB   | Flights
 bookings | routes          | table | postgres | permanent   | heap          | 1016 kB | Routes
 bookings | seats           | table | postgres | permanent   | heap          | 120 kB  | Seats
 bookings | segments        | table | postgres | permanent   | heap          | 1797 MB | Flight segment (leg)
 bookings | tickets         | table | postgres | permanent   | heap          | 1703 MB | Tickets
(9 rows)
```

1. Анализ структуры данных:

Ознакомьтесь с таблицами базы данных, особенно с таблицами bookings, tickets, ticket_flights, flights, boarding_passes, seats, airports, aircrafts.
Определите, какие данные в таблице bookings или других таблицах имеют логическую привязку к диапазонам, по которым можно провести секционирование (например, дата бронирования, рейсы).

> Посмотрим структуру таблиц с большим количеством данных - bookings, boarding_passes, segments, tickets
> - В таблице ```bookings``` мало колонок и значения в них занимают мало места, ее объем обусловлен большим количеством записей.
> Для секционирования могут быть интересны столбцы ```book_ref``` (по хешированию) и ```book_date``` (по диапазону)
```postgresql
-- структура таблицы
demo=# \d+ bookings
                                                        Table "bookings.bookings"
    Column    |           Type           | Collation | Nullable | Default | Storage  | Compression | Stats target |     Description
--------------+--------------------------+-----------+----------+---------+----------+-------------+--------------+----------------------
 book_ref     | character(6)             |           | not null |         | extended |             |              | Booking number
 book_date    | timestamp with time zone |           | not null |         | plain    |             |              | Booking date
 total_amount | numeric(10,2)            |           | not null |         | main     |             |              | Total booking amount
Indexes:
    "bookings_pkey" PRIMARY KEY, btree (book_ref)
Referenced by:
    TABLE "tickets" CONSTRAINT "tickets_book_ref_fkey" FOREIGN KEY (book_ref) REFERENCES bookings(book_ref)
Not-null constraints:
    "bookings_book_ref_not_null" NOT NULL "book_ref"
    "bookings_book_date_not_null" NOT NULL "book_date"
    "bookings_total_amount_not_null" NOT NULL "total_amount"
Access method: heap

-- количество записей
demo=# select count(*) from bookings;
  count
---------
 9706657
(1 row)

-- пример данных
demo=# select * from bookings limit 10;
 book_ref |           book_date           | total_amount
----------+-------------------------------+--------------
 2EW1SQ   | 2025-09-01 00:00:12.557744+00 |      8125.00
 3ZY3O5   | 2025-09-03 00:25:35.278453+00 |      7500.00
 756UAS   | 2025-09-10 16:24:02.978559+00 |      6325.00
 GC7I6S   | 2025-09-03 00:26:37.085293+00 |     18000.00
 RBUUIQ   | 2025-09-01 00:00:33.126206+00 |     13000.00
 GB7E53   | 2025-09-10 15:58:23.948989+00 |     11000.00
 6TAS39   | 2025-09-01 00:00:06.265219+00 |      5000.00
 UEX925   | 2025-09-03 00:27:39.815145+00 |     19550.00
 GGY5EU   | 2025-09-01 00:00:07.565193+00 |      5750.00
 MS148V   | 2025-09-03 00:29:12.70515+00  |      6750.00
```

> В таблице ```boarding_passes``` для секционирования могут быть интересны столбцы ```ticket_no``` (по диапазону) и ```boarding_time``` (по диапазону).\
> Хотя значения столбца ```ticket_no``` выглядят как элементы возрастающей последовательности, но тип у столбца ```text``` и этот столбец не является первичным ключом.
> То есть, вообще говоря, там могут быть любые значения не равные null и удовлетворяющие ограничению ```boarding_passes_pkey```. Следовательно, не стоит выбирать этот столбец для секционирования таблицы

```postgresql
-- структура таблицы
demo=# \d+  boarding_passes
                                                     Table "bookings.boarding_passes"
    Column     |           Type           | Collation | Nullable | Default | Storage  | Compression | Stats target |     Description
---------------+--------------------------+-----------+----------+---------+----------+-------------+--------------+----------------------
 ticket_no     | text                     |           | not null |         | extended |             |              | Ticket number
 flight_id     | integer                  |           | not null |         | plain    |             |              | Flight ID
 seat_no       | text                     |           | not null |         | extended |             |              | Seat number
 boarding_no   | integer                  |           |          |         | plain    |             |              | Boarding pass number
 boarding_time | timestamp with time zone |           |          |         | plain    |             |              | Boarding time
Indexes:
    "boarding_passes_pkey" PRIMARY KEY, btree (ticket_no, flight_id)
    "boarding_passes_flight_id_boarding_no_key" UNIQUE CONSTRAINT, btree (flight_id, boarding_no)
    "boarding_passes_flight_id_seat_no_key" UNIQUE CONSTRAINT, btree (flight_id, seat_no)
Foreign-key constraints:
    "boarding_passes_ticket_no_flight_id_fkey" FOREIGN KEY (ticket_no, flight_id) REFERENCES segments(ticket_no, flight_id)
Not-null constraints:
    "boarding_passes_ticket_no_not_null" NOT NULL "ticket_no"
    "boarding_passes_flight_id_not_null" NOT NULL "flight_id"
    "boarding_passes_seat_no_not_null" NOT NULL "seat_no"
Access method: heap

-- количество записей
demo=# select count(1) from boarding_passes;
count
----------
26299160

-- пример данных
demo=# select * from boarding_passes limit 10;
   ticket_no   | flight_id | seat_no | boarding_no |         boarding_time
---------------+-----------+---------+-------------+-------------------------------
 0005432007794 |        39 | 36E     |         324 | 2025-10-01 06:53:03.887374+00
 0005432003788 |        39 | 35F     |          50 | 2025-10-01 06:39:49.651496+00
 0005432012756 |        39 | 5G      |         150 | 2025-10-01 06:44:41.55811+00
 0005432003895 |        39 | 26B     |         165 | 2025-10-01 06:45:24.74352+00
 0005432018476 |        39 | 29K     |          81 | 2025-10-01 06:41:17.913314+00
 0005441785893 |     59246 | 11B     |          47 | 2026-09-01 00:00:04.05653+00
 0005432033689 |        39 | 38B     |         152 | 2025-10-01 06:44:44.52662+00
 0005432009302 |        39 | 13F     |         397 | 2025-10-01 06:56:35.315773+00
 0005432038037 |       105 | 39E     |         297 | 2025-10-01 14:01:03.171027+00
 0005432019566 |       105 | 35E     |          93 | 2025-10-01 13:47:53.102948+00
(10 rows)
```

> - в таблице ```segments``` для секционирования может быть интересен столбец ```ticket_no``` (по диапазону).\
> Однако, как и в таблице ```boarding_passes```, все аргументы против секционирования по колонке ```ticket_no``` применимы и в данном случае.\
> С другой стороны, в этой таблице это единственный осмысленный кандидат для секционирования.
```postgresql

-- структура таблицы
demo=# \d+ segments
                                                Table "bookings.segments"
     Column      |     Type      | Collation | Nullable | Default | Storage  | Compression | Stats target |  Description
-----------------+---------------+-----------+----------+---------+----------+-------------+--------------+---------------
 ticket_no       | text          |           | not null |         | extended |             |              | Ticket number
 flight_id       | integer       |           | not null |         | plain    |             |              | Flight ID
 fare_conditions | text          |           | not null |         | extended |             |              | Travel class
 price           | numeric(10,2) |           | not null |         | main     |             |              | Travel price
Indexes:
    "segments_pkey" PRIMARY KEY, btree (ticket_no, flight_id)
    "segments_flight_id_idx" btree (flight_id)
Check constraints:
    "segments_fare_conditions_check" CHECK (fare_conditions = ANY (ARRAY['Economy'::text, 'Comfort'::text, 'Business'::text]))
    "segments_price_check" CHECK (price >= 0::numeric)
Foreign-key constraints:
    "segments_flight_id_fkey" FOREIGN KEY (flight_id) REFERENCES flights(flight_id)
    "segments_ticket_no_fkey" FOREIGN KEY (ticket_no) REFERENCES tickets(ticket_no)
Referenced by:
    TABLE "boarding_passes" CONSTRAINT "boarding_passes_ticket_no_flight_id_fkey" FOREIGN KEY (ticket_no, flight_id) REFERENCES segments(ticket_no, flight_id)
Not-null constraints:
    "segments_ticket_no_not_null" NOT NULL "ticket_no"
    "segments_flight_id_not_null" NOT NULL "flight_id"
    "segments_fare_conditions_not_null" NOT NULL "fare_conditions"
    "segments_price_not_null" NOT NULL "price"
Access method: heap

-- количество записей
demo=# select count(1) from segments;
  count
----------
 27580257
(1 row)

-- пример данных
demo=# select * from segments limit 10;
   ticket_no   | flight_id | fare_conditions |  price
---------------+-----------+-----------------+---------
 0005432000000 |       108 | Economy         | 3250.00
 0005432000003 |      1637 | Economy         | 3250.00
 0005432000001 |       164 | Comfort         | 8125.00
 0005432000006 |       174 | Economy         | 5000.00
 0005432000011 |       108 | Economy         | 3250.00
 0005432000002 |       139 | Economy         | 4500.00
 0005432000005 |        88 | Economy         | 2500.00
 0005432000004 |        64 | Economy         | 2500.00
 0005432000013 |      1637 | Economy         | 3250.00
 0005432000016 |      2041 | Economy         | 5000.00
```

> - в таблице ```tickets``` для секционирования могут быть интересны столбцы ```ticket_no``` (по диапазону) и ```passenger_id``` (по диапазону).\
> Аналогично, все аргументы против секционирования по колонке ```ticket_no``` применимы и в данном случае.\
> Столбец ```passanger_id``` интересен тем, что значения в нем можно разделить на две части - код страны и идентификатор пассажира и секционировать таблицу по странам
```postgresql
-- структура таблицы
demo=# \d+ tickets
                                                 Table "bookings.tickets"
     Column     |     Type     | Collation | Nullable | Default | Storage  | Compression | Stats target |   Description
----------------+--------------+-----------+----------+---------+----------+-------------+--------------+------------------
 ticket_no      | text         |           | not null |         | extended |             |              | Ticket number
 book_ref       | character(6) |           | not null |         | extended |             |              | Booking number
 passenger_id   | text         |           | not null |         | extended |             |              | Passenger ID
 passenger_name | text         |           | not null |         | extended |             |              | Passenger name
 outbound       | boolean      |           | not null |         | plain    |             |              | Outbound flight?
Indexes:
    "tickets_pkey" PRIMARY KEY, btree (ticket_no)
    "tickets_book_ref_passenger_id_outbound_key" UNIQUE CONSTRAINT, btree (book_ref, passenger_id, outbound)
Foreign-key constraints:
    "tickets_book_ref_fkey" FOREIGN KEY (book_ref) REFERENCES bookings(book_ref)
Referenced by:
    TABLE "segments" CONSTRAINT "segments_ticket_no_fkey" FOREIGN KEY (ticket_no) REFERENCES tickets(ticket_no)
Not-null constraints:
    "tickets_ticket_no_not_null" NOT NULL "ticket_no"
    "tickets_book_ref_not_null" NOT NULL "book_ref"
    "tickets_passenger_id_not_null" NOT NULL "passenger_id"
    "tickets_passenger_name_not_null" NOT NULL "passenger_name"
    "tickets_outbound_not_null" NOT NULL "outbound"
Access method: heap

-- количество записей
demo=# select count(1) from tickets;
  count
----------
 21095265
(1 row)

-- пример данных
demo=# select * from tickets limit 10;
   ticket_no   | book_ref |   passenger_id   |  passenger_name   | outbound
---------------+----------+------------------+-------------------+----------
 0005432000000 | RBUUIQ   | IT 2425980678984 | Ilaria Cazzaniga  | t
 0005432000001 | 2EW1SQ   | CL 7793846138778 | Claudio Gonzalez  | t
 0005432000003 | RBUUIQ   | IT 2425980678984 | Ilaria Cazzaniga  | f
 0005432000002 | Y4KNLB   | IN 6055838167316 | Sunita Mal        | t
 0005432000004 | GGY5EU   | DE 4384341116867 | Karl-Heinz Arndt  | t
 0005432000005 | 6TAS39   | DE 0926458219327 | Dietmar Franz     | t
 0005432000006 | KOS1KJ   | US 1903569888046 | Jamie Jackson     | t
 0005432000007 | MJUZ2D   | IN 9845886166493 | Smita Chand       | t
 0005432000009 | CFSWW6   | RU 2408782196542 | Natalya Filippova | t
 0005432000008 | YY3YVH   | CN 3545368122753 | Zhiming Hou       | t
(10 rows)
```


2. Выбор таблицы для секционирования:
Основной акцент делается на секционировании таблицы bookings. Но вы можете выбрать и другие таблицы, если видите в этом смысл для оптимизации производительности (например, flights, boarding_passes).
Обоснуйте свой выбор: почему именно эта таблица требует секционирования? Какой тип данных является ключевым для секционирования?

> Да, ```bookings``` один из лучших кандидатов для секционирования, так как в OLTP БД, хранящей операционные данные рано или поздно встает проблема с ухудшением производительности из-за постоянного роста объема данных.\
> А главными кандидатами на секционирование, как раз и являются постоянно растущие в объеме таблицы, в которых хранятся эти самые операционные данные. В отличие от таблиц со справочными данными, объем которых медленно растет со временем\
> Самими удобными для секционирования таблиц c операционными данных являются типы данных:
> - значения в которых монотонно растут; 
> - имеют значения из определенного списка и более-менее равномерно распределены;
> - имеют произвольные значения
> 
> Это позволяет разбивать значения на заранее известные диапазоны и создавать секции по этим диапазонам

3. Определение типа секционирования:
Определитесь с типом секционирования, которое наилучшим образом подходит для ваших данных:

По диапазону (например, по дате бронирования или дате рейса).
По списку (например, по пунктам отправления или по номерам рейсов).
По хэшированию (для равномерного распределения данных).

> Сделаем секционирование таблицы ```bookings``` по колонке ```book_date```.\
> Посмотрим сколько данных содержится в таблице `bookings` по годам

```potgresql
demo=# select extract(year from book_date) as year, count(1) as count from bookings group by year order by year;
 year |  count
------+---------
 2025 | 1703689
 2026 | 4811451
 2027 | 3191517
(3 rows)
```

> Меньше 5 миллионов в год. Секционирования по годам должно хватить для заметного увеличения производительности запросов

4. Создание секционированной таблицы:
Преобразуйте таблицу в секционированную с выбранным типом секционирования.
Например, если вы выбрали секционирование по диапазону дат бронирования, создайте секции по месяцам или годам.

> Воспользуемся утилитой ```pg_dump``` для получения DDL скрипта таблицы ```bookings```
```shell
yc-user@otus-postgres-hw11-1:~$ sudo -u postgres pg_dump -s -t bookings demo > ~/bookings.sql
```
> Модифицируем его необходимым образом, создадим DDL-операторы для создания главной таблицы и секций и запустим их

```postgresql
demo=# create table bookings.bookings_main (
    book_ref character(6) not null,
    book_date timestamp with time zone not null,
    total_amount numeric(10,2) not null,

    -- ВАЖНЫЙ МОМЕНТ: мы должны добавить в первичный ключ таблицы не только основную колонку, 
    -- но и колонку, по которой делаем секционирование
    -- это, к сожалению, снижает строгость ограничения на данные в столбце book_ref
    primary key (book_ref, book_date)
) partition by range (book_date);
CREATE TABLE

-- ВАЖНЫЙ МОМЕНТ 2: диапазоны указываются с учетом неравенства date1 <= value < date2
demo=# create table bookings.bookings_2025 partition of bookings.bookings_main for values from ('2025-01-01') to ('2026-01-01');
CREATE TABLE
demo=# create table bookings.bookings_2026 partition of bookings.bookings_main for values from ('2026-01-01') to ('2027-01-01');
CREATE TABLE
demo=# create table bookings.bookings_2027 partition of bookings.bookings_main for values from ('2027-01-01') to ('2028-01-01');
CREATE TABLE
demo=# create table bookings.bookings_default partition of bookings.bookings_main default;
CREATE TABLE

-- проверяем что получилось
demo=# \d+ bookings_main
                                           Partitioned table "bookings.bookings_main"
    Column    |           Type           | Collation | Nullable | Default | Storage  | Compression | Stats target | Description
--------------+--------------------------+-----------+----------+---------+----------+-------------+--------------+-------------
 book_ref     | character(6)             |           | not null |         | extended |             |              |
 book_date    | timestamp with time zone |           | not null |         | plain    |             |              |
 total_amount | numeric(10,2)            |           | not null |         | main     |             |              |
Partition key: RANGE (book_date)
Indexes:
    "bookings_main_pkey" PRIMARY KEY, btree (book_ref, book_date)
Not-null constraints:
    "bookings_main_book_ref_not_null" NOT NULL "book_ref"
    "bookings_main_book_date_not_null" NOT NULL "book_date"
    "bookings_main_total_amount_not_null" NOT NULL "total_amount"
Partitions: bookings_2025 FOR VALUES FROM ('2025-01-01 00:00:00+00') TO ('2026-01-01 00:00:00+00'),
            bookings_2026 FOR VALUES FROM ('2026-01-01 00:00:00+00') TO ('2027-01-01 00:00:00+00'),
            bookings_2027 FOR VALUES FROM ('2027-01-01 00:00:00+00') TO ('2028-01-01 00:00:00+00'),
            bookings_default DEFAULT
```

5. Миграция данных:

Перенесите существующие данные из исходной таблицы в секционированную структуру.
Убедитесь, что все данные правильно распределены по секциям.

```postgresql
demo=# insert into bookings_main select * from bookings;
INSERT 0 9706657

-- повторим запрос, показывающий количество записей в таблице ```bookings``` по годам 
demo=# select extract(year from book_date) as year, count(1) as count from bookings group by year order by year;
 year |  count
------+---------
 2025 | 1703689
 2026 | 4811451
 2027 | 3191517
(3 rows)

-- и проверим что в соответствующих секциях bookings_* вставлено то же количество строк
demo=# select count(1) from bookings_2025;
  count
---------
 1703689 -- совпадает
(1 row)

demo=# select count(1) from bookings_2026;
  count
---------
 4811451 -- совпадает
(1 row)

demo=# select count(1) from bookings_2027;
  count
---------
 3191517 -- совпадает
(1 row)
```

6. Оптимизация запросов:

Проверьте, как секционирование влияет на производительность запросов. Выполните несколько выборок данных до и после секционирования для оценки времени выполнения.
Оптимизируйте запросы при необходимости (например, добавьте индексы на ключевые столбцы).

> Запросы, которые затронут данные в разных секциях
```postgresql
demo=# explain analyze select * from bookings where book_date between '2025-09-1' and '2027-06-30';
                                                                     QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------------------
 Seq Scan on bookings  (cost=0.00..207471.85 rows=8882115 width=21) (actual time=2.829..1031.690 rows=8895095.00 loops=1)
   Filter: ((book_date >= '2025-09-01 00:00:00+00'::timestamp with time zone) AND (book_date <= '2027-06-30 00:00:00+00'::timestamp with time zone))
   Rows Removed by Filter: 811562
   Buffers: shared hit=376 read=61496
 Planning Time: 0.087 ms
 JIT:
   Functions: 2
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 0.254 ms (Deform 0.079 ms), Inlining 0.000 ms, Optimization 0.262 ms, Emission 2.318 ms, Total 2.833 ms
 Execution Time: 1423.189 ms
(10 rows)

demo=# explain analyze select * from bookings_main where book_date between '2025-09-1' and '2027-06-30';
                                                                        QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------------------------
 Append  (cost=0.00..251882.14 rows=8890865 width=21) (actual time=4.304..1707.866 rows=8895095.00 loops=1)
   Buffers: shared hit=2068 read=59760
   ->  Seq Scan on bookings_2025 bookings_main_1  (cost=0.00..36407.33 rows=1703348 width=21) (actual time=4.303..217.305 rows=1703689.00 loops=1)
         Filter: ((book_date >= '2025-09-01 00:00:00+00'::timestamp with time zone) AND (book_date <= '2027-06-30 00:00:00+00'::timestamp with time zone))
         Buffers: shared hit=658 read=10194
   ->  Seq Scan on bookings_2026 bookings_main_2  (cost=0.00..102818.72 rows=4810486 width=21) (actual time=0.363..523.272 rows=4811451.00 loops=1)
         Filter: ((book_date >= '2025-09-01 00:00:00+00'::timestamp with time zone) AND (book_date <= '2027-06-30 00:00:00+00'::timestamp with time zone))
         Buffers: shared hit=1034 read=29613
   ->  Seq Scan on bookings_2027 bookings_main_3  (cost=0.00..68201.76 rows=2377031 width=21) (actual time=0.346..324.810 rows=2379955.00 loops=1)
         Filter: ((book_date >= '2025-09-01 00:00:00+00'::timestamp with time zone) AND (book_date <= '2027-06-30 00:00:00+00'::timestamp with time zone))
         Rows Removed by Filter: 811562
         Buffers: shared hit=376 read=19953
 Planning Time: 0.181 ms
 JIT:
   Functions: 6
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 0.510 ms (Deform 0.165 ms), Inlining 0.000 ms, Optimization 0.247 ms, Emission 3.939 ms, Total 4.696 ms
 Execution Time: 2063.285 ms
(18 rows)
```
> Запросы, которые затронут данные в одной секции
```postgresql
demo=# explain analyze select * from bookings where book_date between '2026-01-01' and '2026-06-30';
                                                                     QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------------------
 Seq Scan on bookings  (cost=0.00..207471.85 rows=2377934 width=21) (actual time=2.917..667.910 rows=2365065.00 loops=1)
   Filter: ((book_date >= '2026-01-01 00:00:00+00'::timestamp with time zone) AND (book_date <= '2026-06-30 00:00:00+00'::timestamp with time zone))
   Rows Removed by Filter: 7341592
   Buffers: shared hit=658 read=61214
 Planning Time: 0.075 ms
 JIT:
   Functions: 2
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 0.235 ms (Deform 0.063 ms), Inlining 0.000 ms, Optimization 0.286 ms, Emission 2.116 ms, Total 2.636 ms
 Execution Time: 765.846 ms
(10 rows)

demo=# explain analyze select * from bookings_main where book_date between '2026-01-02' and '2026-06-30';
                                                                     QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------------------
 Seq Scan on bookings_2026 bookings_main  (cost=0.00..102818.72 rows=2342079 width=21) (actual time=2.703..396.058 rows=2351145.00 loops=1)
   Filter: ((book_date >= '2026-01-02 00:00:00+00'::timestamp with time zone) AND (book_date <= '2026-06-30 00:00:00+00'::timestamp with time zone))
   Rows Removed by Filter: 2460306
   Buffers: shared hit=1316 read=29331
 Planning Time: 0.166 ms
 JIT:
   Functions: 2
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 0.221 ms (Deform 0.068 ms), Inlining 0.000 ms, Optimization 0.273 ms, Emission 2.192 ms, Total 2.686 ms
 Execution Time: 496.601 ms
(10 rows)
```
> Добавим индекс в обе таблицы
```postgresql
demo=# create index bookings_book_date_idx on bookings using btree (book_date);
CREATE INDEX

demo=# \d bookings
                        Table "bookings.bookings"
    Column    |           Type           | Collation | Nullable | Default
--------------+--------------------------+-----------+----------+---------
 book_ref     | character(6)             |           | not null |
 book_date    | timestamp with time zone |           | not null |
 total_amount | numeric(10,2)            |           | not null |
Indexes:
    "bookings_pkey" PRIMARY KEY, btree (book_ref)
    "bookings_book_date_idx" btree (book_date)
Referenced by:
    TABLE "tickets" CONSTRAINT "tickets_book_ref_fkey" FOREIGN KEY (book_ref) REFERENCES bookings(book_ref)

demo=# create index bookings_main_book_date_idx on bookings_main using btree (book_date);
CREATE INDEX

demo=# \d bookings_main
                Partitioned table "bookings.bookings_main"
    Column    |           Type           | Collation | Nullable | Default
--------------+--------------------------+-----------+----------+---------
 book_ref     | character(6)             |           | not null |
 book_date    | timestamp with time zone |           | not null |
 total_amount | numeric(10,2)            |           | not null |
Partition key: RANGE (book_date)
Indexes:
    "bookings_main_pkey" PRIMARY KEY, btree (book_ref, book_date)
    "bookings_main_book_date_idx" btree (book_date)
Number of partitions: 4 (Use \d+ to list them.)

demo=# \d bookings_2025
                      Table "bookings.bookings_2025"
    Column    |           Type           | Collation | Nullable | Default
--------------+--------------------------+-----------+----------+---------
 book_ref     | character(6)             |           | not null |
 book_date    | timestamp with time zone |           | not null |
 total_amount | numeric(10,2)            |           | not null |
Partition of: bookings_main FOR VALUES FROM ('2025-01-01 00:00:00+00') TO ('2026-01-01 00:00:00+00')
Indexes:
    "bookings_2025_pkey" PRIMARY KEY, btree (book_ref, book_date)
    "bookings_2025_book_date_idx" btree (book_date)
```
> Видно, что у секционированной таблицы индекс ```bookings_main_book_date_idx``` показывается и для главной таблицы и для секции\
> Попробуем запустить те же запросы и посмотреть на планы выполнения

> Запросы, которые затронут данные в разных секциях.\
> Тут заметен выигрыш около 30% при запросе, обращающемся к нескольким секциям, хотя обращение к одной из секций идет не через индекс (видимо, в связи с недостаточным количеством данных)
```postgresql
demo=# explain analyze select * from bookings where book_date between '2025-09-1' and '2026-06-30';
                                                                       QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------------------------
 Index Scan using bookings_book_date_idx on bookings  (cost=0.43..153193.22 rows=4098630 width=21) (actual time=0.038..2074.443 rows=4068754.00 loops=1)
   Index Cond: ((book_date >= '2025-09-01 00:00:00+00'::timestamp with time zone) AND (book_date <= '2026-06-30 00:00:00+00'::timestamp with time zone))
   Index Searches: 1
   Buffers: shared hit=3898781 read=37202
 Planning:
   Buffers: shared read=4
 Planning Time: 0.269 ms
 JIT:
   Functions: 2
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 0.231 ms (Deform 0.066 ms), Inlining 0.000 ms, Optimization 0.000 ms, Emission 0.000 ms, Total 0.231 ms
 Execution Time: 2285.277 ms
(12 rows)


demo=# explain analyze select * from bookings_main where book_date between '2025-09-1' and '2026-06-30';
                                                                                      QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Append  (cost=0.00..144905.82 rows=4059355 width=21) (actual time=3.198..1459.564 rows=4068754.00 loops=1)
   Buffers: shared hit=2270148 read=25200
   ->  Seq Scan on bookings_2025 bookings_main_1  (cost=0.00..36407.33 rows=1703689 width=21) (actual time=3.197..185.321 rows=1703689.00 loops=1)
         Filter: ((book_date >= '2025-09-01 00:00:00+00'::timestamp with time zone) AND (book_date <= '2026-06-30 00:00:00+00'::timestamp with time zone))
         Buffers: shared read=10852
   ->  Index Scan using bookings_2026_book_date_idx on bookings_2026 bookings_main_2  (cost=0.43..88201.71 rows=2355666 width=21) (actual time=0.071..966.608 rows=2365065.00 loops=1)
         Index Cond: ((book_date >= '2025-09-01 00:00:00+00'::timestamp with time zone) AND (book_date <= '2026-06-30 00:00:00+00'::timestamp with time zone))
         Index Searches: 1
         Buffers: shared hit=2270148 read=14348
 Planning:
   Buffers: shared hit=2 read=10
 Planning Time: 0.394 ms
 JIT:
   Functions: 4
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 0.317 ms (Deform 0.105 ms), Inlining 0.000 ms, Optimization 0.205 ms, Emission 2.876 ms, Total 3.397 ms
 Execution Time: 1635.120 ms
(17 rows)
```

> Запросы, которые затронут данные в одной секции. 
> Тут заметен выигрыш около 10% при запросе к одной секции
```postgresql
demo=# explain analyze select * from bookings where book_date between '2026-01-01' and '2026-06-30';
                                                                       QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------------------------
 Index Scan using bookings_book_date_idx on bookings  (cost=0.43..88941.80 rows=2377934 width=21) (actual time=0.072..1008.357 rows=2365065.00 loops=1)
   Index Cond: ((book_date >= '2026-01-01 00:00:00+00'::timestamp with time zone) AND (book_date <= '2026-06-30 00:00:00+00'::timestamp with time zone))
   Index Searches: 1
   Buffers: shared hit=2265720 read=23262
 Planning Time: 0.109 ms
 Execution Time: 1135.618 ms
(6 rows)

demo=# explain analyze select * from bookings_main where book_date between '2026-01-01' and '2026-06-30';
                                                                                  QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Index Scan using bookings_2026_book_date_idx on bookings_2026 bookings_main  (cost=0.43..88201.71 rows=2355666 width=21) (actual time=0.037..978.163 rows=2365065.00 loops=1)
   Index Cond: ((book_date >= '2026-01-01 00:00:00+00'::timestamp with time zone) AND (book_date <= '2026-06-30 00:00:00+00'::timestamp with time zone))
   Index Searches: 1
   Buffers: shared hit=2268022 read=16474
 Planning:
   Buffers: shared read=4
 Planning Time: 0.275 ms
 Execution Time: 1076.862 ms
(8 rows)
```

7. Тестирование решения:
Протестируйте секционирование, выполняя несколько запросов к секционированной таблице.
Проверьте, что операции вставки, обновления и удаления работают корректно.

> Проверяем запросы DML операции на секционированной таблице.

```postgresql
-- проверяем операцию вставки
demo=# insert into bookings_main (book_ref, book_date, total_amount) values ('TSTREF', '2026-05-01 00:00:00+00'::timestamp with time zone, 100.00);
INSERT 0 1
demo=# select * from bookings_main where book_ref = 'TSTREF';
 book_ref |       book_date        | total_amount
----------+------------------------+--------------
 TSTREF   | 2026-05-01 00:00:00+00 |       100.00
(1 row)

-- проверяем операцию обновления
demo=# update bookings_main set total_amount = 200.00 where book_ref = 'TSTREF';
UPDATE 1
demo=# select * from bookings_main where book_ref = 'TSTREF';
 book_ref |       book_date        | total_amount
----------+------------------------+--------------
 TSTREF   | 2026-05-01 00:00:00+00 |       200.00
(1 row)

-- проверяем операцию удаления
demo=# delete from bookings_main where book_ref = 'TSTREF';
DELETE 1
demo=# select * from bookings_main where book_ref = 'TSTREF';
 book_ref | book_date | total_amount
----------+-----------+--------------
(0 rows)
```

> Все DML-операции отработаил корректно на секционированной таблице 