```shell
yc-user@otus-prj-2:~$ sudo systemctl stop postgresql
yc-user@otus-prj-2:~$ sudo -su postgres
postgres@otus-prj-2:/home/yc-user$ rm -rf /var/lib/postgresql/18/main/*
postgres@otus-prj-2:/home/yc-user$ ls -l /var/lib/postgresql/18/main/
total 0
postgres@otus-prj-2:/home/yc-user$ exit
exit
yc-user@otus-prj-2:~$ sudo systemctl start patroni
yc-user@otus-prj-2:~$ sudo systemctl status patroni
● patroni.service - High availability PostgreSQL Cluster
Loaded: loaded (/etc/systemd/system/patroni.service; enabled; preset: enabled)
Active: active (running) since Fri 2026-01-23 19:17:30 UTC; 6s ago
Main PID: 1563 (patroni)
Tasks: 5 (limit: 9483)
Memory: 25.6M (peak: 26.1M)
CPU: 424ms
CGroup: /system.slice/patroni.service
└─1563 /opt/patroni/venv/bin/python3 /opt/patroni/venv/bin/patroni /etc/patroni.yml

Jan 23 19:17:30 otus-prj-2 systemd[1]: Started patroni.service - High availability PostgreSQL Cluster.
Jan 23 19:17:30 otus-prj-2 patroni[1563]: 2026-01-23 19:17:30,758 INFO: Selected new etcd server http://otus-prj-3:2379
Jan 23 19:17:30 otus-prj-2 patroni[1563]: 2026-01-23 19:17:30,766 INFO: No PostgreSQL configuration items changed, nothing to reload.
Jan 23 19:17:30 otus-prj-2 patroni[1563]: 2026-01-23 19:17:30,769 INFO: Systemd integration is not supported
Jan 23 19:17:30 otus-prj-2 patroni[1563]: 2026-01-23 19:17:30,781 INFO: Lock owner: None; I am patroni02
Jan 23 19:17:30 otus-prj-2 patroni[1563]: 2026-01-23 19:17:30,789 INFO: PAUSE: running with empty data directory
yc-user@otus-prj-2:~$ sudo systemctl status patroni
● patroni.service - High availability PostgreSQL Cluster
Loaded: loaded (/etc/systemd/system/patroni.service; enabled; preset: enabled)
Active: active (running) since Fri 2026-01-23 19:17:30 UTC; 56s ago
Main PID: 1563 (patroni)
Tasks: 5 (limit: 9483)
Memory: 25.6M (peak: 26.1M)
CPU: 445ms
CGroup: /system.slice/patroni.service
└─1563 /opt/patroni/venv/bin/python3 /opt/patroni/venv/bin/patroni /etc/patroni.yml

Jan 23 19:17:40 otus-prj-2 patroni[1563]: 2026-01-23 19:17:40,772 INFO: Lock owner: None; I am patroni02
Jan 23 19:17:40 otus-prj-2 patroni[1563]: 2026-01-23 19:17:40,778 INFO: PAUSE: running with empty data directory
Jan 23 19:17:50 otus-prj-2 patroni[1563]: 2026-01-23 19:17:50,771 INFO: Lock owner: None; I am patroni02
Jan 23 19:17:50 otus-prj-2 patroni[1563]: 2026-01-23 19:17:50,777 INFO: PAUSE: running with empty data directory
Jan 23 19:18:00 otus-prj-2 patroni[1563]: 2026-01-23 19:18:00,771 INFO: Lock owner: None; I am patroni02
Jan 23 19:18:00 otus-prj-2 patroni[1563]: 2026-01-23 19:18:00,784 INFO: PAUSE: running with empty data directory
Jan 23 19:18:10 otus-prj-2 patroni[1563]: 2026-01-23 19:18:10,771 INFO: Lock owner: None; I am patroni02
Jan 23 19:18:10 otus-prj-2 patroni[1563]: 2026-01-23 19:18:10,776 INFO: PAUSE: running with empty data directory
Jan 23 19:18:20 otus-prj-2 patroni[1563]: 2026-01-23 19:18:20,771 INFO: Lock owner: None; I am patroni02
Jan 23 19:18:20 otus-prj-2 patroni[1563]: 2026-01-23 19:18:20,789 INFO: PAUSE: running with empty data directory
yc-user@otus-prj-2:~$ sudo /opt/patroni/venv/bin/patronictl -c /etc/patroni.yml list
+ Cluster: patroni (7595321595766063155) -----+----+-------------+-----+------------+-----+
  | Member    | Host        | Role    | State   | TL | Receive LSN | Lag | Replay LSN | Lag |
  +-----------+-------------+---------+---------+----+-------------+-----+------------+-----+
  | patroni01 | 10.129.0.4  | Leader  | running |  1 |             |     |            |     |
  | patroni02 | 10.129.0.32 | Replica | stopped |    |     unknown |     |    unknown |     |
  +-----------+-------------+---------+---------+----+-------------+-----+------------+-----+
  yc-user@otus-prj-2:~$ sudo systemctl status patroni
  ● patroni.service - High availability PostgreSQL Cluster
  Loaded: loaded (/etc/systemd/system/patroni.service; enabled; preset: enabled)
  Active: active (running) since Fri 2026-01-23 19:17:30 UTC; 2min 55s ago
  Main PID: 1563 (patroni)
  Tasks: 5 (limit: 9483)
  Memory: 25.7M (peak: 26.1M)
  CPU: 495ms
  CGroup: /system.slice/patroni.service
  └─1563 /opt/patroni/venv/bin/python3 /opt/patroni/venv/bin/patroni /etc/patroni.yml

Jan 23 19:19:40 otus-prj-2 patroni[1563]: 2026-01-23 19:19:40,771 INFO: Lock owner: None; I am patroni02
Jan 23 19:19:40 otus-prj-2 patroni[1563]: 2026-01-23 19:19:40,776 INFO: PAUSE: running with empty data directory
Jan 23 19:19:50 otus-prj-2 patroni[1563]: 2026-01-23 19:19:50,772 INFO: Lock owner: None; I am patroni02
Jan 23 19:19:50 otus-prj-2 patroni[1563]: 2026-01-23 19:19:50,776 INFO: PAUSE: running with empty data directory
Jan 23 19:20:00 otus-prj-2 patroni[1563]: 2026-01-23 19:20:00,772 INFO: Lock owner: None; I am patroni02
Jan 23 19:20:00 otus-prj-2 patroni[1563]: 2026-01-23 19:20:00,782 INFO: PAUSE: running with empty data directory
Jan 23 19:20:10 otus-prj-2 patroni[1563]: 2026-01-23 19:20:10,771 INFO: Lock owner: None; I am patroni02
Jan 23 19:20:10 otus-prj-2 patroni[1563]: 2026-01-23 19:20:10,776 INFO: PAUSE: running with empty data directory
Jan 23 19:20:20 otus-prj-2 patroni[1563]: 2026-01-23 19:20:20,771 INFO: Lock owner: None; I am patroni02
Jan 23 19:20:20 otus-prj-2 patroni[1563]: 2026-01-23 19:20:20,785 INFO: PAUSE: running with empty data directory

```
