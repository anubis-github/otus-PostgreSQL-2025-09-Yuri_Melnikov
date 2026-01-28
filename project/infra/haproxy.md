```shell
sudo apt update && sudo apt upgrade -y -q && \
sudo apt -y install haproxy
```

```shell
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
sudo nano /etc/haproxy/haproxy.cfg
```

```shell
sudo haproxy -f /etc/haproxy/haproxy.cfg -c
```

```shell
sudo systemctl restart haproxy
```

