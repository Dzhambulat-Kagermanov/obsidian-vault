### Базовые команды:
---
##### Просмотр запущенных контейнеров:
```bash
docker ps
```

##### Информация о контейнерах:
```bash
docker ps -a
```

##### Удаление контейнера:
```bash
docker rm {{container_id}}
```

##### Список образов:
```bash
docker images
```

##### Удаление образа:
```bash
docker rmi {{image_id}}
```

##### Создание и запуск контейнера с выключением через 5 секунд:
```bash
docker run ubuntu sleep 5
```

##### Создание и запуск контейнера в фоне:
```bash
docker run -d ubuntu
```

##### Запуск существующего контейнера:
```bash
docker start {{container_id}}
```

##### Остановка контейнера:
```bash
docker stop {{container_id}}
```

##### Жесткая остановка контейнера:
```bash
docker kill {{container_id}}
```

##### Уничтожение контейнера после остановки:
```bash
docker run --rm -d ubuntu
```

##### Информация о контейнере:
```bash
docker inspect {{container_id}}
```

##### Статистика потребления ресурсов контейнером:
```bash
docker stats {{container_id}}
```

##### Создание и запуск контейнера с указанием имени и выполнением команды echo после запуска:
```bash
docker run -d --rm --name MyContainer ubuntu echo "TEST:TEST"
```

##### Просмотр логов контейнера:
```bash
docker logs {{container_id}}

# Для динамических логов
docker logs -f {{container_id}}
```

##### Вход в контейнер:
```bash
docker exec -it {{container_id}} /bin/bash
```

##### Удаление всех образов с локального хранилища:
```bash
docker system prune -a --volumes
```

##### Проброс порта к контейнеру:
```bash
docker run -p {{внешний_порт}}:{{порт_контейнера}} {{образ}}

# Все порты указанные в образе пробрасываются к случайным не занятым портам хоста
docker run -P {{образ}}
```


### Volumes:
---
##### Монтирование host volumes при запуске контейнера:
```bash
# Сохранится на HOST-е по пути {{host_path}}
docker run -v {{host_path}}:{{container_path}} {{image}}

# Сохранится на HOST-е по пути /opt/mysql
docker run -v /opt/mysql:/var/lib/mysql mysql
```

##### Монтирование anonymous volumes при запуске контейнера:
```bash
# Сохранится на HOST-е по пути /var/lib/mysql/volumes/{{RANDOM_HASH}}
docker run -v {{container_path}} mysql

# Сохранится на HOST-е по пути /var/lib/mysql/volumes/{{RANDOM_HASH}}
docker run -v /var/lib/mysql mysql
```

##### Монтирование named volumes при запуске контейнера:
```bash
# Сохранится на HOST-е по пути /var/lib/mysql/volumes/{{name}}
docker run -v {{name}}:{{host_path}} mysql

# Сохранится на HOST-е по пути /var/lib/mysql/volumes/mysql_data
docker run -v mysql_data:/var/lib/mysql mysql
```

##### Запуск контейнера в read-only режиме:
```bash
# У контейнера нет прав менять volume
docker run -v {{name}}:{{host_path}}:ro mysql
```

##### Список volume-ов;
```bash
docker volume ls
```

### Сети:
---
При первоначальном запуске docker-а на хосте создаётся новый интерфейс `docker0`'. После установки docker предоставляет несколько типов сетей:
* **bridge** - сеть по умолчанию (172.16.0.0/16). Использует драйвер `bridge`. ip адрес получается случайным образом в своей подсети. Есть доступ к контейнеру извне:
```bash
docker run {{image}}
# или
docker run {{image}} -p {{внешний_порт}}:{{порт_контейнера}}
```

* **host** - получает ip хоста. Есть доступ к контейнеру извне. Использует драйвер `host`. Нельзя создавать больше одной сети типа `host`.
```bash
docker run {{image}} --network=host
```

* **none** - отключает доступ к контейнеру по сети. Использует драйвер `null`. Доступ возможен только при прямом входе через докер на хосте. Нельзя создавать больше одной сети типа `null`.
```bash
docker run {{image}} --network=none
```

* **macvlan** - ???;

* **ipvlan** - ???;

* **overlay** - для кластерной сети Docker swarm;

##### Создание сети типа bridge:
```bash
docker network create --drive bridge {{имя_сети}}

# С указанием подсети и шлюза
docker network create --drive bridge --subnet {{x.x.x.x/x}} --gateway {{x.x.x.x}} {{имя_сети}}
```

##### Удаление сети:
```bash
docker network rm {{имя_сети}}
```

##### Создание и запуск контейнера в конкретной сети:
```bash
docker run {{image}} --network={{имя_сети}}
```
> [!NOTE] Примечание
> * Контейнеры могут общаться только с контейнерами из той же сети;
> * Контейнеры могут общаться между собой по доменным именам. Доменное именем контейнера является имя данное при создании контейнера флагом --name или если не указано имя то сгенерированное docker-ом имя;

##### Список сетей:
```bash
docker network ls
```

### Dockerfile
---
##### Структура Dockerfile-а:
```Dockerfile
# Базовый образ
FROM ubuntu:22.04

# Описание образа
LABEL {{ключ}}={{значение}}
LABEL {{ключ}}={{значение}}
LABEL {{ключ}}={{значение}}

# Команды для подготовки базового образа, которые запускаются до основных команд CMD и ENTRYPOINT
RUN {{команда}}

# Рабочик директории
WORKDIR {{путь_к_директории_в_контейнере}}

# Копирование файлов из локальной машины в контейнер
COPY {{from_path}} {{to_path}}

# Переменные окружения по-умолчанию
ENV {{ключ}}={{значение}}

# Информационное поле о том какой порт прокидывается из контейнера наружу
EXPOSE {{порт}}
EXPOSE {{порт}}
EXPOSE {{порт}}

# Выполнение команды на базовом образе, которые МОГУТ БЫТЬ ПЕРЕЗАПИСАНЫ во время сборки с помощью параметра
CMD ["echo", "Hello world 1"]
CMD ["echo", "Hello world 2"]

# Выполнение команды на базовом образе, которые НЕ МОГУТ БЫТЬ ПЕРЕЗАПИСАНЫ во время сборки с помощью параметра
ENTRYPOINT ["echo", "Hello world 1"]
ENTRYPOINT ["echo", "Hello world 2"]
```

##### Сборка образа по Dockerfile-у:
```bash
docker build {{путь_к_файлу}}

# С указанием имени и тега
docker build -t {{имя}}:{{версия_или_другой_тег}} {{путь_к_файлу}}
```

##### Изменение информации о образе:
```bash
docker tag {{image_id}} {{имя}}:{{версия_или_другой_тег}}
```

##### Информация о сборке образа:
```bash
docker image inspect {{имя_образа}}
```

### Docker Compose:
---

```yml
# Версия docker-compose
version: "3.5"

# Контейнеры
services:
	web-service:
		# Образ для контейнер. нельзя использовать вместе с build
		image: nginx:stable
		
		# Образ для контейнер. нельзя использовать вместе с image
		build:
			context: .
			dockerfile: Dockerfile
			
		# Имя контейнера
		container_name
		
		# Переменные окружения
		environment:
			- NGINX_PORT=80
			- NGINX_HOST=domain.com
		
		# Файл переменных окружения
		env_file:
			- .env
		
		# Пробрасываемые порты
		ports:
			- "80:80"
			- "443:443"
		
		# Сеть в которой запускается контейнер
		networks:
			- front
			- back
		  
		# Что нужно сделать с контейнером при остановке, перезагрузке сервера и др
		restart: unless-stopped # always / no / on-failure
		
		# Запуск контейнера только после того как запустятся контейнеры в списке
		depends_on:
			- test
		
		# Хранилища, тома
		volumes:
			- /opt/web/html:/var/www/html
			- /opt/web/pics:/var/www/html
			  
		

# Сети
networks:
	front:
		driver: bridge
		name: frontend
	back:
		driver: bridge
		name: backend
```

##### Запуск всех сервисов в фоне:
```bash
docker-compose up -d

# Пересоздание и перезапуск изменённых сервисов
docker-compose up -d --build

# Масштабирование сервиса
docker-compose -f {{путь}} up -d

# использовать кастомный .env файл:
docker-compose --env-file {{путь}} up -d
```

##### Удаление неиспользуемых томов, сетей и образов:
```bash
docker-compose down --volumes --remove-orphans
```

##### Остановка и удаление контейнеров, сетей (но не томов):
```bash
docker-compose down
```

##### Остановка сервисов (без удаления):
```bash
docker-compose stop
```

##### Запуск остановленных сервисов:
```bash
docker-compose start
```

##### Логи сервисов:
```bash
docker-compose logs

docker-compose logs {{service_name}}
```

##### Просмотр запущенных контейнеров:
```bash
docker-compose ps
```

##### Выполнение команды внутри контейнера:
```bash
docker-compose exec {{service}} /bin/bash
```

##### Проверка конфигурации на ошибки:
```bash
docker-compose config
```

