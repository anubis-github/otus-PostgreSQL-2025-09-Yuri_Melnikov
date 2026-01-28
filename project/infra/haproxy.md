> Ставим HAProxy на хост otus-prj-3
```shell
sudo apt update && sudo apt upgrade -y -q && \
sudo apt -y install haproxy
```
> Настраиваем конфиг HAProxy из файла configs/haproxy.cfg
```shell
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
sudo nano /etc/haproxy/haproxy.cfg
```

> Валидируем конфиг
```shell
sudo haproxy -f /etc/haproxy/haproxy.cfg -c
```

> Перезагружаем HAProxy
```shell
sudo systemctl restart haproxy
```

