##### Установка nginx через apt:
```bash
apt-get update
apt-get install nginx
```

##### Установка nginx из исходного кода:
```
wget https://nginx.org/download/nginx-1.9.2.tar.gz
unzip nginx-1.9.2.tar.gz -d ./my-nginx
```


##### Запуск служб nginx:
```bash
systemctl enable --now nginx
```

##### Файлы конфигурации находятся по пути `/etc/nginx`
