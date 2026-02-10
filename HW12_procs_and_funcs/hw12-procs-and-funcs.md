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

2. Создать триггер на таблице продаж, для поддержки данных в витрине в актуальном состоянии (вычисляющий при каждой продаже сумму и записывающий её в витрину)

```postgresql
create or replace function tf_maintain_fff()
returns trigger as
$$
declare
begin
case TG_OP
    when 'INSERT' then
        -- обновляем запись на витрине для товара добавляя значение NEW в sales
        -- если записи на витрине нет - создаем ее и записываем то, что пришло в NEW  
    when 'UPDATE' then
        -- обновляем запись на витрине для товара, вычитая или прибавляя разницу между OLD и NEW в sales
        -- если записи на витрине нет - исключение
    when 'DELETE' then
        -- обновляем запись на витрине для товара, вычитая значение OLD из удаленной записи из sales
        -- если записи на витрине нет - исключение
        
end;
$$ language plpgsql;


create trigger trg_maintain_xxx
after insert or update or delete 
for each statement
execute procudure tf_maintain_fff();

```

Подсказка: не забыть, что кроме INSERT есть еще UPDATE и DELETE

3. Задание со звездочкой*

Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?
Подсказка: В реальной жизни возможны изменения цен.