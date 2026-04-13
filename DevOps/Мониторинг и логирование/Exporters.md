### NodeExporter:

**Node Exporter** — это официальный экспортер метрик для Prometheus, предназначенный для сбора аппаратных и операционных метрик с Unix-подобных систем (Linux, FreeBSD, macOS и др.).

Node Exporter извлекает метрики о состоянии хоста, такие как:

- **CPU**: загрузка по ядрам, время в режимах user/system/iowait.
- **Память**: использование RAM, swap, page faults.
- **Диск**: I/O операции, latency, использование места (`node_filesystem_*`).
- **Сеть**: трафик (bytes sent/received), ошибки, drop packets (`node_network_*`).
- **Система**: uptime, load average, количество процессов, температурные датчики (если есть `hwmon`).

Node Exporter написан на Go, поставляется как один бинарный файл. Он не требует установки зависимостей.

**Установка на Linux:**

1. Скачать архив с [github.com/prometheus/node_exporter](https://github.com/prometheus/node_exporter/releases);
2. Распаковать в `/usr/local/bin/`;
3. Создать пользователя `node_exporter` (без возможности залогинится) для безопасности;
4. Настроить systemd unit file.

Пример systemd юнита (`/etc/systemd/system/node_exporter.service`):

```vim
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter \
    --web.listen-address=":9100" \
    --collector.systemd \
    --collector.processes

[Install]
WantedBy=multi-user.target
```

> **Важно:** По умолчанию Node Exporter слушает на порту **9100**.

**Ключевые флаги запуска:**

- `--web.listen-address=":9100"`: Адрес и порт для скрапинга.
- `--path.procfs="/proc"` и `--path.sysfs="/sys"`: Пути к псевдофайловым системам. Полезно, если ты запускаешь exporter внутри контейнера или chroot, где эти пути смонтированы иначе.
- `--collector.disable-defaults`: Отключает все коллекторы по умолчанию. Используется вместе с явным включением нужных через `--collector.<name>`. Это хорошая практика для снижения нагрузки и шума в метриках.
- `--log.level=info`: Уровень логирования (debug, info, warn, error).

**Запуск сервиса node_exporter.service**

```bash
systemctl enable --now node_exporter.service
```

**Проверка работоспособности:**

```bash
curl 127.0.0.1:9100/metrics
```

На экране мы должны увидеть метрики, на подобие:

```
...
# TYPE process_virtual_memory_max_bytes gauge  
process_virtual_memory_max_bytes 1.8446744073709552e+19  
# HELP promhttp_metric_handler_errors_total Total number of internal errors encountered by the promhttp metric handler.  
# TYPE promhttp_metric_handler_errors_total counter  
promhttp_metric_handler_errors_total{cause="encoding"} 0  
promhttp_metric_handler_errors_total{cause="gathering"} 0  
# HELP promhttp_metric_handler_requests_in_flight Current number of scrapes being served.  
# TYPE promhttp_metric_handler_requests_in_flight gauge  
promhttp_metric_handler_requests_in_flight 1  
# HELP promhttp_metric_handler_requests_total Total number of scrapes by HTTP status code.  
# TYPE promhttp_metric_handler_requests_total counter  
promhttp_metric_handler_requests_total{code="200"} 0  
promhttp_metric_handler_requests_total{code="500"} 0  
promhttp_metric_handler_requests_total{code="503"} 0
...
```

Также можно открыть веб-браузер и перейти по адресу **`http://<IP-адрес сервера или клиента>:9100/metrics`** — мы увидим метрики, собранные node_exporter:

![[prometheus-linux-06.jpg|697]]

### NGINX exporter:

!!!TODO

### PostgreSQL Exporter:

!!!TODO

### Blackbox exporter:

!!!TODO