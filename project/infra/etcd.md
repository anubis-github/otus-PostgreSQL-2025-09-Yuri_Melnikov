1. Устанавливаем etcd
> Аппаратные рекомендации https://etcd.io/docs/v3.6/op-guide/hardware/
> Оригинальная инструкция https://etcd.io/docs/v3.6/install/
> Инструкция с apt https://linuxconfig.org/how-to-install-etcd-on-ubuntu
> Инструкция из презентации

> На каждой ноде ставим etcd
```shell
sudo apt -y install etcd-server
sudo apt -y install etcd-client
``` 

2. Настраиваем etcd кластер

> На каждой ноде:
> - останавливаем etcd
```shell
sudo systemctl stop etcd
sudo systemctl disable etcd
```
> - проверяем что сервис остановлен
```shell
sudo systemctl status etcd.service
```

> - удаляем конфигурацию по умолчанию и БД
```shell
sudo rm -rf /var/lib/etcd/default
```

> - прописываем новую конфигурацию
```shell
sudo nano /etc/default/etcd
````
> Конфигурация otus-prj-1
```shell
ETCD_NAME="etcd1"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://otus-prj-1:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://otus-prj-1:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd_Name_Cluster"
ETCD_INITIAL_CLUSTER="etcd1=http://otus-prj-1:2380,etcd2=http://otus-prj-2:2380,etcd3=http://otus-prj-3:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_ELECTION_TIMEOUT="10000"
ETCD_HEARTBEAT_INTERVAL="2000"
ETCD_INITIAL_ELECTION_TICK_ADVANCE="false"
```

> Конфигурация otus-prj-2
```shell
ETCD_NAME="etcd2"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://otus-prj-2:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://otus-prj-2:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd_Name_Cluster"
ETCD_INITIAL_CLUSTER="etcd1=http://otus-prj-1:2380,etcd2=http://otus-prj-2:2380,etcd3=http://otus-prj-3:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_ELECTION_TIMEOUT="10000"
ETCD_HEARTBEAT_INTERVAL="2000"
ETCD_INITIAL_ELECTION_TICK_ADVANCE="false"
```

> Конфигурация otus-prj-3
```shell
ETCD_NAME="etcd3"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://otus-prj-3:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://otus-prj-3:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd_Name_Cluster"
ETCD_INITIAL_CLUSTER="etcd1=http://otus-prj-1:2380,etcd2=http://otus-prj-2:2380,etcd3=http://otus-prj-3:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_ELECTION_TIMEOUT="10000"
ETCD_HEARTBEAT_INTERVAL="2000"
ETCD_INITIAL_ELECTION_TICK_ADVANCE="false"
```

> Перезачитываем конфигурацию, включаем автозапуск etcd и стартуем сервис
```shell
sudo systemctl daemon-reload 
sudo systemctl enable etcd 
sudo systemctl start etcd 
```

3. Проверяем работу etcd кластера
```shell
sudo systemctl status etcd.service
etcdctl endpoint status --cluster -w table
```

4. Создаем снапшот ВМ
```shell
# otus-prj-1 ВМ
yc compute snapshot create \
  --name otus-prj-1-etcd \
  --description "otus-prj-1-etcd" \
  --disk-name otus-prj-1

# otus-prj-2 ВМ
yc compute snapshot create \
  --name otus-prj-2-etcd \
  --description "otus-prj-2-etcd" \
  --disk-name otus-prj-2

# otus-prj-3 ВМ
yc compute snapshot create \
  --name otus-prj-3-etcd \
  --description "otus-prj-3-etcd" \
  --disk-name otus-prj-3
```

5. Если что-то пошло не так
> Если запорота БД кластера
> - убиваем БД и перезапускаем сервис
```shell
sudo systemctl stop etcd
sudo rm -R /var/lib/etcd/member/
sudo systemctl start etcd
```
> - Диагностика
```shell
sudo systemctl status etcd.service
sudo journalctl -xeu etcd.service
sudo journalctl -e etcd.service
etcdctl endpoint health --cluster -w table
etcdctl endpoint status --cluster -w table
```

