> Создаем хотст 1 с нуля
```shell
yc compute instance create \
--name otus-prj-1 \
--hostname otus-prj-1 \
--cores 4 \
--core-fraction 100 \
--memory 8 \
--create-boot-disk name=otus-prj-1,size=20G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2404-lts \
--network-interface subnet-name=default-ru-central1-b,nat-ip-version=ipv4 \
--ssh-key ~/.ssh/ssh-key-postgres.pub
```

> Подключение хост 1
```shell
vm_ip_address=$(yc compute instance show --name otus-prj-1 | grep -E ' +address' | tail -n 1 | awk '{print $2}') && \
ssh -o StrictHostKeyChecking=no -i ~/.ssh/ssh-key-postgres yc-user@$vm_ip_address
```

```shell
yc compute snapshot create \
  --name otus-prj-1-etcd \
  --description "otus-prj-1-etcd" \
  --disk-name otus-prj-1
```

```shell
yc compute snapshot create \
  --name otus-prj-1-etcd-pg \
  --description "otus-prj-1-etcd-pg" \
  --disk-name otus-prj-1
```

```shell
yc compute snapshot create \
  --name otus-prj-1-patroni \
  --description "otus-prj-1-patroni" \
  --disk-name otus-prj-1
```

```shell
yc compute instance create \
--name otus-prj-1 \
--hostname otus-prj-1 \
--cores 4 \
--core-fraction 100 \
--memory 8 \
--create-boot-disk name=otus-prj-1,snapshot-name=otus-prj-1-patroni \
--network-interface subnet-name=default-ru-central1-b,nat-ip-version=ipv4 \
--ssh-key ~/.ssh/ssh-key-postgres.pub
```


> Создаем хост 2
```shell
yc compute instance create \
--name otus-prj-2 \
--hostname otus-prj-2 \
--cores 4 \
--core-fraction 100 \
--memory 8 \
--create-boot-disk name=otus-prj-2,size=20G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2404-lts \
--network-interface subnet-name=default-ru-central1-b,nat-ip-version=ipv4 \
--ssh-key ~/.ssh/ssh-key-postgres.pub
```
> Подключение хост 2
```shell
vm_ip_address=$(yc compute instance show --name otus-prj-2 | grep -E ' +address' | tail -n 1 | awk '{print $2}') && \
ssh -o StrictHostKeyChecking=no -i ~/.ssh/ssh-key-postgres yc-user@$vm_ip_address
```

```shell
yc compute snapshot create \                                                                  ✔ ╱ took 5s  ╱ at 23:43:23  ▓▒░
  --name otus-prj-2-etcd \
  --description "otus-prj-2-etcd" \
  --disk-name otus-prj-2
```

```shell
yc compute snapshot create \
  --name otus-prj-2-etcd-pg \
  --description "otus-prj-2-etcd-pg" \
  --disk-name otus-prj-2
```
```shell
yc compute instance create \
--name otus-prj-2 \
--hostname otus-prj-2 \
--cores 4 \
--core-fraction 100 \
--memory 8 \
--create-boot-disk name=otus-prj-2,snapshot-name=otus-prj-2-etcd-pg \
--network-interface subnet-name=default-ru-central1-b,nat-ip-version=ipv4 \
--ssh-key ~/.ssh/ssh-key-postgres.pub
```


> Создаем хост 3
```shell
yc compute instance create \
--name otus-prj-3 \
--hostname otus-prj-3 \
--cores 4 \
--core-fraction 100 \
--memory 8 \
--create-boot-disk name=otus-prj-3,size=20G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2404-lts \
--network-interface subnet-name=default-ru-central1-b,nat-ip-version=ipv4 \
--ssh-key ~/.ssh/ssh-key-postgres.pub
```
> Подключение хост 3
```shell
vm_ip_address=$(yc compute instance show --name otus-prj-3 | grep -E ' +address' | tail -n 1 | awk '{print $2}') && \
ssh -o StrictHostKeyChecking=no -i ~/.ssh/ssh-key-postgres yc-user@$vm_ip_address
```

```shell
yc compute snapshot create \                                                                  ✔ ╱ took 5s  ╱ at 23:43:23  ▓▒░
  --name otus-prj-3-etcd \
  --description "otus-prj-3-etcd" \
  --disk-name otus-prj-3
```

```shell
yc compute snapshot create \
  --name otus-prj-3-etcd-pg \
  --description "otus-prj-3-etcd-pg" \
  --disk-name otus-prj-3
```
