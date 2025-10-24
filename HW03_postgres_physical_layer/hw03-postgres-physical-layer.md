# Физический уровень PostgreSQL

1. создайте виртуальную машину c Ubuntu 20.04/22.04 LTS в ЯО/Virtual Box/докере
> VM cоздана в UI консоли Я.облако с параметрами 2CPU, 2Gb RAM, 20Gb HDD

2. Поставьте на нее PostgreSQL 18 через sudo apt

```shell
# Использовались команды из официальной документации postgres https://www.postgresql.org/download/linux/ubuntu/ 
# Делаем автоматическую настройку репозитория apt PostgreSQL

postgres@postgres-vm-2-2-20-hdd:~$ sudo apt install -y postgresql-common
Reading package lists... Done
Building dependency tree... Done
...
Running kernel seems to be up-to-date.
No services need to be restarted.
No containers need to be restarted.
No user sessions are running outdated binaries.
No VM guests are running outdated hypervisor (qemu) binaries on this host.

postgres@postgres-vm-2-2-20-hdd:~$ sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
This script will enable the PostgreSQL APT repository on apt.postgresql.org on
your system. The distribution codename used will be noble-pgdg.

Press Enter to continue, or Ctrl-C to abort.

Using keyring /usr/share/postgresql-common/pgdg/apt.postgresql.org.gpg
Writing /etc/apt/sources.list.d/pgdg.sources ...

Running apt-get update ...
Hit:1 http://mirror.yandex.ru/ubuntu noble InRelease
Hit:2 http://mirror.yandex.ru/ubuntu noble-updates InRelease
Hit:3 http://mirror.yandex.ru/ubuntu noble-backports InRelease
Hit:4 http://security.ubuntu.com/ubuntu noble-security InRelease
Hit:5 https://download.docker.com/linux/ubuntu noble InRelease
Get:6 https://apt.postgresql.org/pub/repos/apt noble-pgdg InRelease [107 kB]
Get:7 https://apt.postgresql.org/pub/repos/apt noble-pgdg/main amd64 Packages [350 kB]
Fetched 457 kB in 1s (657 kB/s)
Reading package lists... Done

You can now start installing packages from apt.postgresql.org.

Have a look at https://wiki.postgresql.org/wiki/Apt for more information;
most notably the FAQ at https://wiki.postgresql.org/wiki/Apt/FAQ
```
```shell
# Ставим PostgreSQL 18
postgres@postgres-vm-2-2-20-hdd:~$ sudo apt install postgresql-18
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
...
Running kernel seems to be up-to-date.
No services need to be restarted.
No containers need to be restarted.
No user sessions are running outdated binaries.
No VM guests are running outdated hypervisor (qemu) binaries on this host.
```

3. Проверьте что кластер запущен через sudo -u postgres pg_lsclusters

```shell
# Проверяем статус кластера
postgres@postgres-vm-2-2-20-hdd:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 online postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```
4. Зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым

```shell
# Пользуемся знанием параметров по умолчанию: psql без параметров подсоединится к БД postgres через unix-сокет на том же хосте localhost под пользователем postgres 
postgres@postgres-vm-2-2-20-hdd:~$ psql
psql (18.0 (Ubuntu 18.0-1.pgdg24.04+3))
Type "help" for help.

postgres=#
```

``#`` Создаем таблицу с данным
>postgres=# create table test(c1 text);\
> CREATE TABLE\
> postgres=# insert into test values('1');\
> INSERT 0 1

5. Остановите postgres например через sudo -u postgres pg_ctlcluster 18 main stop

```shell
# Останавливаем кластер
postgres@postgres-vm-2-2-20-hdd:~$ sudo -u postgres pg_ctlcluster 18 main stop
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@18-main

# Проверяем что он остановлен
postgres@postgres-vm-2-2-20-hdd:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 down   postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```

6. Создайте новый диск к ВМ размером 10GB
> Делаем по инструкции https://yandex.cloud/ru/docs/compute/operations/disk-create/empty \
> Создаем диск размером 10Gb типа HDD с размером блока 4Kb в зоне ru-central1-b

7. Добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk
> Диск подключен в консоли ВМ с именем устройства disk-hdd-10gb

8. Проинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на актуальное, в вашем случае это скорее всего будет /dev/sdb - https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux

```shell
# Ищем имя неинициализированного диска
postgres@postgres-vm-2-2-20-hdd:~$ sudo parted -l | grep Error
Error: /dev/vdb: unrecognised disk label

# Убеждаемся что на нет тома
postgres@postgres-vm-2-2-20-hdd:~$ lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
vda     253:0    0   30G  0 disk
├─vda1  253:1    0 29.4G  0 part /
├─vda14 253:14   0    4M  0 part
└─vda15 253:15   0  600M  0 part /boot/efi
vdb     253:16   0   10G  0 disk

# Создаем том типа gpt
postgres@postgres-vm-2-2-20-hdd:~$ sudo parted /dev/vdb mklabel gpt
Information: You may need to update /etc/fstab.

# Создаем партицию на весь том
postgres@postgres-vm-2-2-20-hdd:~$ sudo parted -a opt /dev/vdb mkpart primary ext4 0% 100%
Information: You may need to update /etc/fstab.

# Убеждаемся что партиция появилась
postgres@postgres-vm-2-2-20-hdd:~$ lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
vda     253:0    0   30G  0 disk
├─vda1  253:1    0 29.4G  0 part /
├─vda14 253:14   0    4M  0 part
└─vda15 253:15   0  600M  0 part /boot/efi
vdb     253:16   0   10G  0 disk
└─vdb1  253:17   0   10G  0 part

# Создаем файловую систему на партиции
postgres@postgres-vm-2-2-20-hdd:~$ sudo mkfs.ext4 -L datapartition /dev/vdb1
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 2619392 4k blocks and 655360 inodes
Filesystem UUID: 455833ba-fbfb-4d6c-962a-659ed7be565c
Superblock backups stored on blocks:
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done

# Монтируем файловую систему
postgres@postgres-vm-2-2-20-hdd:~$ sudo mkdir -p /mnt/data
postgres@postgres-vm-2-2-20-hdd:~$ sudo mount -o defaults /dev/vdb1 /mnt/data

# Прописываем партицию в fstab, чтобы диск монтировался каждый раз при старте ВМ
postgres@postgres-vm-2-2-20-hdd:~$ sudo vim /etc/fstab
LABEL=cloudimg-rootfs   /        ext4   discard,commit=30,errors=remount-ro     0 1
LABEL=UEFI /boot/efi vfat umask=0077 0 1
LABEL=datapartition /mnt/data ext4 defaults 0 2

# Проверяем что можем создавать и читать файлы с нового диска
postgres@postgres-vm-2-2-20-hdd:~$ df -h -x tmpfs
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        29G  3.4G   26G  12% /
/dev/vda15      599M  6.2M  593M   2% /boot/efi
/dev/vdb1       9.8G   24K  9.3G   1% /mnt/data
postgres@postgres-vm-2-2-20-hdd:~$ echo "success" | sudo tee /mnt/data/test_file
success
postgres@postgres-vm-2-2-20-hdd:~$ cat /mnt/data/test_file
success
```

9. Перезагрузите инстанс и убедитесь, что диск остается примонтированным (если не так смотрим в сторону fstab)
```shell
# Перезаходим на ВМ и убеждаемся, что можем прочитать содержимое ранее записаного файла
postgres@postgres-vm-2-2-20-hdd:~$ exit
logout
Connection to 84.252.141.73 closed.
❯ ssh -i ~/.ssh/ssh-key-postgres postgres@84.252.141.73
Welcome to Ubuntu 24.04.3 LTS (GNU/Linux 6.8.0-85-generic x86_64)

...

Last login: Sun Oct 19 16:41:12 2025 from 93.100.77.210
postgres@postgres-vm-2-2-20-hdd:~$

postgres@postgres-vm-2-2-20-hdd:~$ cat /mnt/data/test_file
success
```

10. Сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/
```shell
postgres@postgres-vm-2-2-20-hdd:~$ sudo chown -R postgres:postgres /mnt/data/
```

11. Перенесите содержимое /var/lib/postgres/18 в /mnt/data - mv /var/lib/postgresql/18 /mnt/data
```shell
# Переносим данные
postgres@postgres-vm-2-2-20-hdd:~$ mv /var/lib/postgresql/18 /mnt/data

# Проверяем перенос
postgres@postgres-vm-2-2-20-hdd:~$ ls -l /mnt/data
total 24
drwxr-xr-x 3 postgres postgres  4096 Oct 19 17:06 18
drwx------ 2 postgres postgres 16384 Oct 19 19:03 lost+found
-rw-r--r-- 1 postgres postgres     8 Oct 19 19:15 test_file
```
12. Попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 18 main start и напишите получилось или нет и почему
```shell
# Пытаемся стартовать кластер и получаем ошибку 
# так как перенесли данные кластера на другой диск и не обновили конфигурацию
postgres@postgres-vm-2-2-20-hdd:~$ sudo -u postgres pg_ctlcluster 18 main start
Error: /var/lib/postgresql/18/main is not accessible or does not exist
```

13. Задание: найти конфигурационный параметр в файлах раcположенных в /etc/postgresql/18/main который надо поменять и поменяйте его
```shell
# Ищем конфигурационный параметр, содержащий старый путь к данным /var/lib/postgresql/18 в файлах в директории /etc/postgresql/18/main/
postgres@postgres-vm-2-2-20-hdd:~$ sudo grep -r "/var/lib/postgresql/18" /etc/postgresql/18/main/
/etc/postgresql/18/main/postgresql.conf:data_directory = '/var/lib/postgresql/18/main'		# use data in another directory

# Находим его в файле /etc/postgresql/18/main/postgresql.conf и создаем резервную копию файла
postgres@postgres-vm-2-2-20-hdd:~$ sudo cp /etc/postgresql/18/main/postgresql.conf /etc/postgresql/18/main/postgresql.conf.bak

# Убеждаемся, что теперь параметр находится в обоих файлах
postgres@postgres-vm-2-2-20-hdd:~$  sudo grep -r "/var/lib/postgresql" /etc/postgresql/18/main/
/etc/postgresql/18/main/postgresql.conf:data_directory = '/var/lib/postgresql/18/main'		# use data in another directory
/etc/postgresql/18/main/postgresql.conf.bak:data_directory = '/var/lib/postgresql/18/main'		# use data in another directory

 #  Меняем старое значение параметра data_directory на /mnt/data/18/main
postgres@postgres-vm-2-2-20-hdd:~$ sudo sed -i 's/var\/lib\/postgresql/mnt\/data/g' /etc/postgresql/18/main/postgresql.conf

# Убеждаемся, что значение параметра обновилось правильно
postgres@postgres-vm-2-2-20-hdd:~$ sudo grep -r "/mnt/data" /etc/postgresql/18/main/
/etc/postgresql/18/main/postgresql.conf:data_directory = '/mnt/data/18/main'		# use data in another directory

# Удаляем бэкап файла конфигурации
postgres@postgres-vm-2-2-20-hdd:~$ sudo rm -rf /etc/postgresql/18/main/postgresql.conf.bak

# Проверяем что старое значение параметра больше нигде не встречается
postgres@postgres-vm-2-2-20-hdd:~$ sudo grep -r "/var/lib/postgresql/18" /etc/postgresql/18/main/ 
```

14. Напишите что и почему поменяли
> Поменял путь к данным кластера в найденном параметре `data_directory' в файле `/etc/postgresql/18/main/postgresql.conf` на новый

15. Попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 18 main start
```shell
# Запускаем кластер
postgres@postgres-vm-2-2-20-hdd:~$ sudo -u postgres pg_ctlcluster 18 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@18-main
Removed stale pid file.

# Проверяем его статус
postgres@postgres-vm-2-2-20-hdd:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory    Log file
18  main    5432 online postgres /mnt/data/18/main /var/log/postgresql/postgresql-18-main.log
```

16. Напишите получилось или нет и почему
> Теперь получилось запустить кластер так как мы указали в конфигурационном файле `/etc/postgresql/18/main/postgresql.conf`\
> правильное значение параметра data_directory, 

17. Зайдите через через psql и проверьте содержимое ранее созданной таблицы
```shell
# Запускаем клиента
postgres@postgres-vm-2-2-20-hdd:~$ psql
psql (18.0 (Ubuntu 18.0-1.pgdg24.04+3))
Type "help" for help.
```

``#`` Проверяем содержимое таблицы test, все данные на месте
> postgres=# select * from test;\
> c1\
> ----\
> 1\
>(1 row)

18. Задание со звездочкой *: не удаляя существующий инстанс ВМ сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgres, перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.