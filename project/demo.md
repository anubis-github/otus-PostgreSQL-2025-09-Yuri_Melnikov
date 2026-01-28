1. Создаем ВМ otus-prj-1, otus-prj-2, otus-prj-3
> Описано в infra/vm.md

2. Устанавливаем etcd на ВМ otus-prj-1, otus-prj-2, otus-prj-3
> Описано в infra/etcd.md

3. Устанавливаем Postgres на ВМ otus-prj-1, otus-prj-2
> Описано в infra/pg.md

4. Устанавливаем Patroni на ВМ otus-prj-1, otus-prj-2
> Описано в infra/patroni.md

5. Устанавливаем HAProxy на ВМ otus-prj-3
> Описано в infra/haproxy.md

6. Демонстрация
> Проверяем статус patroni на otus-prj-1
```shell
yc-user@otus-prj-1:~$ sudo /opt/patroni/venv/bin/patronictl -c /etc/patroni.yml list
+ Cluster: patroni (7595321595766063155) -------+----+-------------+-----+------------+-----+
| Member    | Host        | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+-----------+-------------+---------+-----------+----+-------------+-----+------------+-----+
| patroni01 | 10.129.0.4  | Leader  | running   |  7 |             |     |            |     |
| patroni02 | 10.129.0.16 | Replica | streaming |  7 |   0/50014F0 |   0 |  0/50014F0 |   0 |
+-----------+-------------+---------+-----------+----+-------------+-----+------------+-----+
```

> Првоеряем статус HAProxy на otus-prj-3. otus-prj-2 DOWN

```shell
yc-user@otus-prj-3:~$ sudo systemctl status haproxy
● haproxy.service - HAProxy Load Balancer
     Loaded: loaded (/usr/lib/systemd/system/haproxy.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-01-28 18:20:19 UTC; 10s ago
       Docs: man:haproxy(1)
             file:/usr/share/doc/haproxy/configuration.txt.gz
   Main PID: 3211 (haproxy)
     Status: "Ready."
      Tasks: 5 (limit: 9483)
     Memory: 39.7M (peak: 40.3M)
        CPU: 133ms
     CGroup: /system.slice/haproxy.service
             ├─3211 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -S /run/haproxy-master.sock
             └─3215 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -S /run/haproxy-master.sock

Jan 28 18:20:19 otus-prj-3 systemd[1]: Starting haproxy.service - HAProxy Load Balancer...
Jan 28 18:20:19 otus-prj-3 haproxy[3211]: [NOTICE]   (3211) : New worker (3215) forked
Jan 28 18:20:19 otus-prj-3 haproxy[3211]: [NOTICE]   (3211) : Loading success.
Jan 28 18:20:19 otus-prj-3 systemd[1]: Started haproxy.service - HAProxy Load Balancer.
Jan 28 18:20:20 otus-prj-3 haproxy[3215]: [WARNING]  (3215) : Server postgres/otus-prj-2 is DOWN, reason: Layer7 wrong status, code: 503, info: "Service Unavailable", check duration: 26ms. 1 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
```

> Делаем switchover на otus-prj-1
```shell
yc-user@otus-prj-1:~$ sudo /opt/patroni/venv/bin/patronictl -c /etc/patroni.yml switchover
Current cluster topology
+ Cluster: patroni (7595321595766063155) -------+----+-------------+-----+------------+-----+
| Member    | Host        | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+-----------+-------------+---------+-----------+----+-------------+-----+------------+-----+
| patroni01 | 10.129.0.4  | Leader  | running   |  7 |             |     |            |     |
| patroni02 | 10.129.0.16 | Replica | streaming |  7 |   0/50014F0 |   0 |  0/50014F0 |   0 |
+-----------+-------------+---------+-----------+----+-------------+-----+------------+-----+
Primary [patroni01]:
Candidate ['patroni02'] []:
When should the switchover take place (e.g. 2026-01-28T19:24 )  [now]:
Are you sure you want to switchover cluster patroni, demoting current leader patroni01? [y/N]: y
2026-01-28 18:24:31.62377 Successfully switched over to "patroni02"
+ Cluster: patroni (7595321595766063155) -----+----+-------------+-----+------------+-----+
| Member    | Host        | Role    | State   | TL | Receive LSN | Lag | Replay LSN | Lag |
+-----------+-------------+---------+---------+----+-------------+-----+------------+-----+
| patroni01 | 10.129.0.4  | Replica | stopped |    |     unknown |     |    unknown |     |
| patroni02 | 10.129.0.16 | Leader  | running |  7 |             |     |            |     |
+-----------+-------------+---------+---------+----+-------------+-----+------------+-----+
```

> Првоеряем статус HAProxy на otus-prj-3. otus-prj-1 стал DOWN, а otus-prj-2 стал UP
```shell
yc-user@otus-prj-3:~$ sudo systemctl status haproxy
● haproxy.service - HAProxy Load Balancer
     Loaded: loaded (/usr/lib/systemd/system/haproxy.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-01-28 18:20:19 UTC; 4min 29s ago
       Docs: man:haproxy(1)
             file:/usr/share/doc/haproxy/configuration.txt.gz
   Main PID: 3211 (haproxy)
     Status: "Ready."
      Tasks: 5 (limit: 9483)
     Memory: 39.8M (peak: 40.4M)
        CPU: 185ms
     CGroup: /system.slice/haproxy.service
             ├─3211 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -S /run/haproxy-master.sock
             └─3215 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -S /run/haproxy-master.sock

Jan 28 18:20:19 otus-prj-3 systemd[1]: Starting haproxy.service - HAProxy Load Balancer...
Jan 28 18:20:19 otus-prj-3 haproxy[3211]: [NOTICE]   (3211) : New worker (3215) forked
Jan 28 18:20:19 otus-prj-3 haproxy[3211]: [NOTICE]   (3211) : Loading success.
Jan 28 18:20:19 otus-prj-3 systemd[1]: Started haproxy.service - HAProxy Load Balancer.
Jan 28 18:20:20 otus-prj-3 haproxy[3215]: [WARNING]  (3215) : Server postgres/otus-prj-2 is DOWN, reason: Layer7 wrong status, code: 503, info: "Service Unavailable", check duration: 26ms. 1 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
Jan 28 18:24:36 otus-prj-3 haproxy[3215]: [WARNING]  (3215) : Server postgres/otus-prj-2 is UP, reason: Layer7 check passed, code: 200, check duration: 2ms. 2 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
Jan 28 18:24:37 otus-prj-3 haproxy[3215]: [WARNING]  (3215) : Server postgres/otus-prj-1 is DOWN, reason: Layer7 wrong status, code: 503, info: "Service Unavailable", check duration: 2ms. 1 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
```

> Проверяем статус patroni на otus-prj-1
```shell
yc-user@otus-prj-1:~$ sudo /opt/patroni/venv/bin/patronictl -c /etc/patroni.yml list
+ Cluster: patroni (7595321595766063155) -------+----+-------------+-----+------------+-----+
| Member    | Host        | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+-----------+-------------+---------+-----------+----+-------------+-----+------------+-----+
| patroni01 | 10.129.0.4  | Replica | streaming |  8 |   0/5001740 |   0 |  0/5001740 |   0 |
| patroni02 | 10.129.0.16 | Leader  | running   |  8 |             |     |            |     |
+-----------+-------------+---------+-----------+----+-------------+-----+------------+-----+
```