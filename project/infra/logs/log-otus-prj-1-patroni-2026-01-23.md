```shell
yc-user@otus-prj-1:~$ etcdctl endpoint status --cluster -w table
+------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|        ENDPOINT        |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| http://otus-prj-1:2379 | 694d56d42326b4e4 |  3.4.30 |   20 kB |     false |      false |        59 |      11537 |              11537 |        |
| http://otus-prj-2:2379 | b7fc32347a545ede |  3.4.30 |   20 kB |      true |      false |        59 |      11537 |              11537 |        |
| http://otus-prj-3:2379 | f62d14776096ab00 |  3.4.30 |   20 kB |     false |      false |        59 |      11537 |              11537 |        |
+------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+

yc-user@otus-prj-1:~$ sudo /opt/patroni/venv/bin/patroni --validate-config /etc/patroni.yml -i
yc-user@otus-prj-1:~$ sudo nano /etc/systemd/system/patroni.service
yc-user@otus-prj-1:~$ sudo systemctl daemon-reload
yc-user@otus-prj-1:~$ sudo systemctl enable patroni
Created symlink /etc/systemd/system/multi-user.target.wants/patroni.service → /etc/systemd/system/patroni.service.
yc-user@otus-prj-1:~$ sudo systemctl start patroni
yc-user@otus-prj-1:~$ sudo systemctl status patroni
● patroni.service - High availability PostgreSQL Cluster
     Loaded: loaded (/etc/systemd/system/patroni.service; enabled; preset: enabled)
     Active: active (running) since Fri 2026-01-23 18:41:21 UTC; 5s ago
   Main PID: 1194 (patroni)
      Tasks: 5 (limit: 9483)
     Memory: 32.1M (peak: 35.1M)
        CPU: 518ms
     CGroup: /system.slice/patroni.service
             └─1194 /opt/patroni/venv/bin/python3 /opt/patroni/venv/bin/patroni /etc/patroni.yml

Jan 23 18:41:22 otus-prj-1 patroni[1194]: 2026-01-23 18:41:22,843 INFO: Changed ssl_cert_file from '/etc/ssl/certs/ssl-cert-snakeoil.pem' to 'None'
Jan 23 18:41:22 otus-prj-1 patroni[1194]: 2026-01-23 18:41:22,843 INFO: Changed ssl_key_file from '/etc/ssl/private/ssl-cert-snakeoil.key' to 'None'
Jan 23 18:41:22 otus-prj-1 patroni[1194]: 2026-01-23 18:41:22,843 INFO: Changed TimeZone from 'Etc/UTC' to 'None'
Jan 23 18:41:22 otus-prj-1 patroni[1194]: 2026-01-23 18:41:22,843 INFO: Changed unix_socket_directories from '/var/run/postgresql' to 'None' (restart might be required)
Jan 23 18:41:22 otus-prj-1 patroni[1194]: 2026-01-23 18:41:22,843 INFO: Changed wal_keep_size from '0' to '128MB'
Jan 23 18:41:22 otus-prj-1 patroni[1194]: 2026-01-23 18:41:22,843 INFO: Changed wal_log_hints from 'off' to 'on' (restart might be required)
Jan 23 18:41:22 otus-prj-1 patroni[1194]: 2026-01-23 18:41:22,857 INFO: Reloading PostgreSQL configuration.
Jan 23 18:41:22 otus-prj-1 patroni[1202]: server signaled
Jan 23 18:41:23 otus-prj-1 patroni[1194]: 2026-01-23 18:41:23,909 INFO: Systemd integration is not supported
Jan 23 18:41:24 otus-prj-1 patroni[1194]: 2026-01-23 18:41:24,040 INFO: acquired session lock as a leader
yc-user@otus-prj-1:~$ sudo /opt/patroni/venv/bin/patronictl -c /etc/patroni.yml list
+ Cluster: patroni (7595321595766063155) ---+----+-------------+-----+------------+-----+-----------------+-------------------------------------------+
| Member    | Host       | Role   | State   | TL | Receive LSN | Lag | Replay LSN | Lag | Pending restart | Pending restart reason                    |
+-----------+------------+--------+---------+----+-------------+-----+------------+-----+-----------------+-------------------------------------------+
| patroni01 | 10.129.0.4 | Leader | running |  1 |             |     |            |     | *               | cluster_name: 18/main->patroni            |
|           |            |        |         |    |             |     |            |     |                 | listen_addresses: *->127.0.0.1,10.129.0.4 |
|           |            |        |         |    |             |     |            |     |                 | wal_log_hints: off->on                    |
+-----------+------------+--------+---------+----+-------------+-----+------------+-----+-----------------+-------------------------------------------+

```

```shell
yc-user@otus-prj-1:~$ sudo /opt/patroni/venv/bin/patronictl -c /etc/patroni.yml list
2026-01-23 20:06:12,524 - ERROR - Request to server http://otus-prj-3:2379 failed: MaxRetryError('HTTPConnectionPool(host=\'otus-prj-3\', port=2379): Max retries exceeded with url: /v2/keys/db/patroni/?recursive=true&quorum=true (Caused by ReadTimeoutError("HTTPConnectionPool(host=\'otus-prj-3\', port=2379): Read timed out. (read timeout=3.3310338943333497)"))')
+ Cluster: patroni (7595321595766063155) -----+----+-------------+-----+------------+-----+
| Member    | Host        | Role    | State   | TL | Receive LSN | Lag | Replay LSN | Lag |
+-----------+-------------+---------+---------+----+-------------+-----+------------+-----+
| patroni01 | 10.129.0.4  | Replica | running |  1 |   0/30002E8 |   0 |  0/3000398 |   0 |
| patroni02 | 10.129.0.24 | Leader  | running |  2 |             |     |            |     |
+-----------+-------------+---------+---------+----+-------------+-----+------------+-----+
```

```shell
yc-user@otus-prj-1:~$ sudo /opt/patroni/venv/bin/patronictl -c /etc/patroni.yml switchover
Current cluster topology
+ Cluster: patroni (7595321595766063155) -----+----+-------------+-----+------------+-----+
| Member    | Host        | Role    | State   | TL | Receive LSN | Lag | Replay LSN | Lag |
+-----------+-------------+---------+---------+----+-------------+-----+------------+-----+
| patroni01 | 10.129.0.4  | Replica | running |  1 |   0/30002E8 |   0 |  0/3000398 |   0 |
| patroni02 | 10.129.0.24 | Leader  | running |  2 |             |     |            |     |
+-----------+-------------+---------+---------+----+-------------+-----+------------+-----+
Primary [patroni02]:
Candidate ['patroni01'] []:
When should the switchover take place (e.g. 2026-01-23T21:07 )  [now]:
Are you sure you want to switchover cluster patroni, demoting current leader patroni02? [y/N]: y
2026-01-23 20:07:20.68116 Successfully switched over to "patroni01"
+ Cluster: patroni (7595321595766063155) -----+----+-------------+-----+------------+-----+
| Member    | Host        | Role    | State   | TL | Receive LSN | Lag | Replay LSN | Lag |
+-----------+-------------+---------+---------+----+-------------+-----+------------+-----+
| patroni01 | 10.129.0.4  | Leader  | running |  1 |             |     |            |     |
| patroni02 | 10.129.0.24 | Replica | stopped |    |     unknown |     |    unknown |     |
+-----------+-------------+---------+---------+----+-------------+-----+------------+-----+
```

```shell
yc-user@otus-prj-1:~$ sudo /opt/patroni/venv/bin/patronictl -c /etc/patroni.yml list
2026-01-23 20:08:07,782 - ERROR - Failed to get list of machines from http://otus-prj-3:2379/v2: MaxRetryError('HTTPConnectionPool(host=\'otus-prj-3\', port=2379): Max retries exceeded with url: /v2/machines (Caused by ReadTimeoutError("HTTPConnectionPool(host=\'otus-prj-3\', port=2379): Read timed out. (read timeout=3.3309773243333516)"))')
2026-01-23 20:08:11,126 - ERROR - Request to server http://otus-prj-3:2379 failed: MaxRetryError('HTTPConnectionPool(host=\'otus-prj-3\', port=2379): Max retries exceeded with url: /v2/keys/db/patroni/?recursive=true&quorum=true (Caused by ReadTimeoutError("HTTPConnectionPool(host=\'otus-prj-3\', port=2379): Read timed out. (read timeout=3.3327786573333547)"))')
+ Cluster: patroni (7595321595766063155) -----+----+-------------+-----+------------+-----+
| Member    | Host        | Role    | State   | TL | Receive LSN | Lag | Replay LSN | Lag |
+-----------+-------------+---------+---------+----+-------------+-----+------------+-----+
| patroni01 | 10.129.0.4  | Leader  | running |  3 |             |     |            |     |
| patroni02 | 10.129.0.24 | Replica | running |  2 |   0/3000000 |   0 |  0/3000570 |   0 |
+-----------+-------------+---------+---------+----+-------------+-----+------------+-----+
```
