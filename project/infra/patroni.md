Установка на каждой ноде (вариант 2): 
Новый механизмом безопасности в Ubuntu 22.04+ (и Debian 12+) блокирует глобальную установку Python-пакетов через pip, чтобы не повредить системные пакеты. 

1. Ставим модуль для создания виртуальных окружений
```shell
sudo apt update && sudo apt install python3.12-venv
```
 
2. Создаём каталог для Patroni.
```shell
sudo mkdir -p /opt/patroni
```

3. Передаём владение каталогом пользователю postgres 
```shell
sudo chown postgres:postgres /opt/patroni
```

4. Создаём виртуальное окружение от имени postgres. 
```shell
sudo -u postgres python3 -m venv /opt/patroni/venv
```

5. Устанавливаем Patroni с поддержкой etcd3. 
```shell
sudo -u postgres /opt/patroni/venv/bin/pip install 'patroni[etcd3]'
```

6. Устанавливаем драйвер для работы Patroni с PostgreSQL. 
```shell
sudo -u postgres /opt/patroni/venv/bin/pip install 'psycopg2-binary'
```

7. Конфигурация Patroni на каждой ноде - взять конфиг из файлов configs/patroni-1.yml и configs/patroni-2.yml
> - редактируем конфиг
```shell
sudo nano /etc/patroni.yml
```
> - валидируем конфиг
```shell
sudo /opt/patroni/venv/bin/patroni --validate-config /etc/patroni.yml -i
```

8. Конфигурируем сервис Patroni на каждой ноде - взять конфиг из файла configs/patroni.service
```shell
sudo nano /etc/systemd/system/patroni.service
```
9. Перевод Patroni в автозапуск, старт и проверка
> - Обновляем конфиг, включаем автозапуск и стартуем сервис
```shell
sudo systemctl daemon-reload
sudo systemctl enable patroni
sudo systemctl start patroni
```
>  Проверка работы кластера patroni
```shell
sudo systemctl status patroni
sudo /opt/patroni/venv/bin/patronictl -c /etc/patroni.yml list
```
> - Диагностика, которой пользовался для исследования проблем
```shell
sudo journalctl -xeu patroni.service
sudo journalctl -u patroni
```
