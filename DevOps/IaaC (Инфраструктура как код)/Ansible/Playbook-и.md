
Плейбук — это YAML-файл, который описывает **сценарий**: что делать, на каких хостах, в каком порядке и при каких условиях.

**Структура простейшего плейбука.** Минимальный плейбук состоит из трех обязательных элементов:

1. **`name`**: Описание того, что делает этот блок (видно в логах).
2. **`hosts`**: На каких машинах запускать (группа из инвентаря).
3. **`tasks`**: Список задач (действий).

**Пример playbook-а:**

```yaml
---
# Три тире в начале обязательны для YAML.

- name: Базовая настройка веб-сервера  # Название "плея" (сценария)
  hosts: webservers                     # Группа хостов из инвентаря
  become: yes                           # Выполнять задачи от root (sudo)

  tasks:                                # Начало списка задач
    - name: Установить пакет nginx
      apt:                              # Имя модуля (для Debian/Ubuntu)
        name: nginx
        state: present                  # desired state: установлен

    - name: Запустить сервис nginx
      service:
        name: nginx
        state: started
        enabled: yes                    # Добавить в автозагрузку
```

**Запуск playbook-а:**

```bash
ansible-playbook -i inventory/hosts.ini site.yml
```

**Что произойдет:**

1. Ansible подключится ко всем хостам группы `webservers`.
2. Выполнит задачу 1: проверит, стоит ли nginx. Если нет — установит. Если есть — ничего не сделает (идемпотентность).
3. Выполнит задачу 2: проверит статус сервиса. Если не запущен — запустит.

## Основные модули:

### Управление пакетами (Package Management):

#### Package:

Универсальный. Сам выбирает менеджер пакетов (`apt`, `yum`, `dnf`) в зависимости от ОС. Пример:

```yaml
- name: Установить необходимые пакеты
  package:
    name:
      - vim
      - git
      - curl
    state: present
```

#### Apt:

Специфичен для Debian/Ubuntu. Позволяет делать `update_cache: yes`. Пример:

```yaml
- name: Установить необходимые пакеты
  apt:
    name:
      - vim
      - git
      - curl
    state: present
```

#### Yum / Dnf:

Для RHEL/CentOS/Fedora. Пример:

```yaml
- name: Установить необходимые пакеты
  yum:
    name:
      - vim
      - git
      - curl
    state: present
```

```yaml
- name: Установить необходимые пакеты
  dnf:
    name:
      - vim
      - git
      - curl
    state: present
```

#### PIP:

 Установка пакетов Python. Пример:

```yaml
- name: Установить необходимые пакеты
  pip:
    name:
      - vim
      - git
      - curl
    state: present
```

### Управление файлами и директориями:

#### File:

Создание файлов, папок, ссылок. Изменение прав (`mode`), владельца (`owner`).

**Создание директории с правами**

```yaml
- name: Создать директорию для логов приложения
  file:
    path: /var/log/myapp
    state: directory       # Гарантирует, что это папка
    owner: www-data        # Владелец
    group: www-data        # Группа
    mode: '0755'           # Права (rwxr-xr-x)
    recurse: yes           # Применить рекурсивно, если путь уже существует
```

**Создание пустого файла (touch):**

```yaml
- name: Создать пустой файл флага
  file:
    path: /opt/app/init.done
    state: touch           # Аналог команды touch
    owner: appuser
    mode: '0644'
```

**Создание символической ссылки (Symlink):**

```yaml
- name: Создать симлинк на текущую версию приложения
  file:
    src: /opt/app/releases/v2.5.0      # Цель (куда ссылается)
    dest: /opt/app/current             # Сама ссылка
    state: link                        # Тип: символическая ссылка
    force: yes                         # Перезаписать, если dest уже существует
```

**Удаление файла или директории:**

```yaml
- name: Удалить старую директорию кэша
  file:
    path: /var/cache/myapp
    state: absent          # Удаляет файл или директорию (рекурсивно)
```

**Изменение прав существующего файла:**

```yaml
- name: Исправить права на конфиг
  file:
    path: /etc/myapp/config.json
    mode: '0600'           # Только чтение/запись владельцу
    owner: root
    group: root
```

**Переместить файл или директорию (Move):**

```yaml
- name: Переместить директорию с данными
  file:
    src: /var/data/old_location
    dest: /var/data/new_location
    state: file # Или можно не указывать, Ansible поймет сам
    # Если dest - существующая директория, файл будет перемещен ВНУТРЬ неё.
    # Если dest не существует, old_location будет переименован в new_location.
```

#### Copy: 

Копирование файла **с контроллера** на удаленный сервер.

**Простое копирование файла:**

```yaml
- name: Скопировать статический конфиг
  copy:
    src: files/nginx.conf          # Путь относительно playbook или абсолютный на контроллере
    dest: /etc/nginx/nginx.conf    # Путь на удаленном сервере
    owner: root
    group: root
    mode: '0644'
```

**Копирование с созданием бэкапа**:

```yaml
- name: Обновить конфиг с бэкапом оригинала
  copy:
    src: files/hosts
    dest: /etc/hosts
    backup: yes                    # Создаст копию вида /etc/hosts.ansible_timestamp перед заменой
    validate: '/usr/sbin/visudo -cf %s' # Проверить синтаксис перед заменой (для sudoers)
```

**Запись текста прямо из плейбука (параметр `content`):**

```yaml
- name: Создать файл приветствия
  copy:
    content: |
      Hello from {{ ansible_hostname }}!
      Deployed at: {{ ansible_date_time.iso8601 }}
    dest: /var/www/html/index.txt
    mode: '0644'
```

**Копирование директории:**

```yaml
- name: Скопировать всю папку со скриптами
  copy:
    src: scripts/                  # Слэш в конце важен! Копирует СОДЕРЖИМОЕ папки
    dest: /opt/app/scripts/
    mode: '0755'
    preserve: yes                  # Сохранить даты модификации и права исходных файлов
```

#### Template:

Как `copy`, но обрабатывает файл как шаблон **Jinja2** (подставляет переменные).

**Применение шаблона с переменными:**

_Предварительно создаём файл `templates/nginx.conf.j2`:_

```jinja2
server {
    listen {{ http_port }};
    server_name {{ domain_name }};
    
    {% if enable_ssl %}
    ssl_certificate /etc/ssl/certs/{{ domain_name }}.crt;
    {% endif %}
    
    location / {
        proxy_pass http://{{ backend_ip }}:{{ backend_port }};
    }
}
```

```yaml
- name: Развернуть конфиг nginx из шаблона
  template:
    src: templates/nginx.conf.j2
    dest: /etc/nginx/sites-available/default
    owner: root
    group: root
    mode: '0644'
    backup: yes
  vars:
    http_port: 80
    domain_name: "example.com"
    enable_ssl: true
    backend_ip: "127.0.0.1"
    backend_port: 8080
  notify: Restart Nginx  # Уведомление хендлеру при изменении файла
```

**Шаблон с циклами (генерация списка upstream):**

```jinja2
upstream backend_pool {
    {% for server in backend_servers %}
    server {{ server.ip }}:{{ server.port }} weight={{ server.weight }};
    {% endfor %}
}
```

```yaml
- name: Generate upstream config
  template:
    src: upstream.j2
    dest: /etc/nginx/conf.d/upstream.conf
  vars:
    backend_servers:
      - { ip: "192.168.1.10", port: 80, weight: 5 }
      - { ip: "192.168.1.11", port: 80, weight: 3 }
```

#### Lineinfile:

Гарантирует наличие одной строки в файле. Удаляет дубликаты. Идеально для правки конфигов.

**Добавить строку, если её нет:**

```yaml
- name: Добавить запись в hosts
  lineinfile:
    path: /etc/hosts
    line: "192.168.1.50 db-master.local"
    state: present
```

**Заменить существующую строку (по регулярке):**

```yaml
- name: Изменить порт SSH в конфиге
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?Port\s+\d+'      # Ищем строку типа "Port 22" или "#Port 22"
    line: 'Port 2222'            # Заменяем на эту
    state: present
  notify: Restart SSHD
```

**Удалить строку:**

```yaml
- name: Удалить небезопасную настройку
  lineinfile:
    path: /etc/security/limits.conf
    regexp: '^.*soft.*nofile.*'
    state: absent
```

**Вставить строку после определенной линии:**

```yaml
- name: Добавить переменную окружения после PATH
  lineinfile:
    path: /etc/environment
    line: 'MY_APP_VAR="production"'
    insertafter: '^PATH='        # Вставить сразу после строки начинающейся с PATH=
```

**Редактирование sudo:**

```yaml
- name: Разрешить группе 'developers' выполнять любые команды без пароля
  lineinfile:
    path: /etc/sudoers.d/developers
    line: '%developers ALL=(ALL) NOPASSWD: ALL'
    create: yes
    mode: '0440'  # Права должны быть строгими (чтение владельцем и группой)
    owner: root
    group: root
    validate: 'visudo -cf %s' # ПРОВЕРКА СИНТАКСИСА перед записью!
                              # %s заменяется на временный файл проверки
```

#### Blockinfile:

Вставка блока текста в файл (между маркерами).

**Вставка блока конфигурации:**

```yaml
- name: Добавить настройки ядра в sysctl
  blockinfile:
    path: /etc/sysctl.conf
    block: |
      net.ipv4.ip_forward = 1
      net.ipv4.conf.all.send_redirects = 0
      vm.swappiness = 10
    marker: "# {mark} ANSIBLE MANAGED BLOCK"
    create: yes
```

_Результат в файле_

```Text
# BEGIN ANSIBLE MANAGED BLOCK
net.ipv4.ip_forward = 1
net.ipv4.conf.all.send_redirects = 0
vm.swappiness = 10
# END ANSIBLE MANAGED BLOCK
```

Если запустить тот же плейбук с измененным содержимым `block`, Ansible найдет маркеры и заменит только текст внутри них, не трогая остальной файл.

**Удаление блока:**

```yaml
- name: Удалить блок настроек
  blockinfile:
    path: /etc/sysctl.conf
    block: ""                  # Пустой блок
    state: absent              # Или просто указать state: absent без block
    marker: "# {mark} ANSIBLE MANAGED BLOCK"
```

#### Replace:

Замена текста по регулярному выражению.

**Простая замена слова:**

```yaml
- name: Заменить старое имя домена на новое
  replace:
    path: /var/www/html/config.php
    regexp: 'old-domain\.com'
    replace: 'new-domain.com'
```

**Замена значения параметра:**

```yaml
- name: Изменить максимальное количество соединений
  replace:
    path: /etc/postgresql/14/main/postgresql.conf
    regexp: '^max_connections\s*=\s*\d+'
    replace: 'max_connections = 200'
```

**Удаление комментариев или лишних строк:**

```bash
- name: Удалить все пустые строки в файле
  replace:
    path: /tmp/cleanup.txt
    regexp: '^\n'
    replace: ''
```

#### Archive / Unarchive:

Создание и распаковка архивов (tar, gz, zip, bz2).

**Создание архива (Backup):**

```yaml
- name: Создать архив с логами
  archive:
    path:
      - /var/log/myapp/*.log
      - /var/log/myapp/error.log
    dest: /backups/myapp_logs_{{ ansible_date_time.date }}.tar.gz
    format: gz
    owner: backup_user
    mode: '0640'
```

**Распаковка архива (с контроллера):**

```yaml
- name: Распаковать приложение
  unarchive:
    src: files/app-release-v1.0.tar.gz  # Лежит на машине Ansible
    dest: /opt/app/                     # Распаковать сюда
    owner: appuser
    group: appuser
    creates: /opt/app/bin/start.sh      # Не распаковывать, если этот файл уже есть (идемпотентность)
```

**Распаковка архива (уже лежащего на сервере):**

```yaml
- name: Распаковать архив скачанный ранее
  unarchive:
    src: /tmp/downloaded-package.zip    # Лежит на удаленном сервере
    dest: /opt/software/
    remote_src: yes                     # ВАЖНО: сказать Ansible, что источник локальный для хоста
    creates: /opt/software/readme.txt
```

**Распаковка с исключением файлов:**

```yaml
- name: Распаковать без тестовых данных
  unarchive:
    src: dataset.tar.gz
    dest: /data/
    remote_src: yes
    exclude:
      - "*.tmp"
      - "tests/"
```

### Управление сервисами и процессами:

#### Service:

Универсальное управление сервисами (SysV, Upstart, systemd). Ansible сам определяет, какая система инициализации используется (systemd, SysVinit, Upstart), и вызывает нужную команду.

**Запуск сервиса и добавление в автозагрузку:**

```yaml
- name: Убедиться, что Nginx запущен и в автозагрузке
  service:
    name: nginx
    state: started       # started, stopped, restarted, reloaded
    enabled: yes         # Добавить в автозагрузку (analog systemctl enable)
```

**Перезагрузка сервиса (Restart):**

```yaml
- name: Перезапустить SSH при изменении конфига
  service:
    name: sshd
    state: restarted
  # Примечание: Будьте осторожны при удаленном подключении! 
  # Лучше использовать 'reloaded', если конфиг позволяет.
```

**Мягкая перезагрузка (Reload):**

```yaml
- name: Применить новые настройки Nginx без разрыва соединений
  service:
    name: nginx
    state: reloaded
```

**Остановка сервиса и удаление из автозагрузки:**

```yaml
- name: Отключить и остановить старый сервис Telnet
  service:
    name: telnetd
    state: stopped
    enabled: no
```

**Проверка статуса (через register):**

Модуль `service` возвращает статус в переменную, если использовать `register`.

```yaml
- name: Проверить статус Docker
  service:
    name: docker
  register: docker_status

- name: Вывести состояние сервиса
  debug:
    msg: "Docker is {{ docker_status.status.ActiveState }}"
  when: docker_status.status is defined
```

#### Systemd:

Прямое управление `systemctl`. Дает больше контроля (например, просмотр логов юнита).

**Применение изменений в юнит-файлах (Daemon Reload):**

Если вы изменили файл `.service` в `/etc/systemd/system/`, нужно сообщить systemd об изменениях перед рестартом.

```yaml
- name: Обновить список юнитов systemd после изменения файлов
  systemd:
    daemon_reload: yes   # Выполняет systemctl daemon-reload
```

**Комплексное управление (Enable + Start + Reload):**

```yaml
- name: Развернуть кастомный сервис myapp
  systemd:
    name: myapp
    enabled: yes
    state: restarted
    daemon_reload: yes   # Можно объединить в одной задаче!
```

**Получение статуса и логов (Debug):**

```yaml
- name: Получить полную информацию о сервисе PostgreSQL
  systemd:
    name: postgresql
  register: pg_info

- name: Вывести статус и последние логи
  debug:
    msg: |
      Status: {{ pg_info.unit.ActiveState }}
      SubStatus: {{ pg_info.unit.SubState }}
      Recent Logs: {{ pg_info.unit.StatusText }}
```

**Маскировка сервиса (Mask):**

Запрещает запуск сервиса даже вручную (создает симлинк на `/dev/null`). Полезно для конфликтовующих сервисов (например, `firewalld` vs `ufw`).

```yaml
- name: Навсегда запретить запуск firewalld
  systemd:
    name: firewalld
    masked: yes
    state: stopped
```

#### Command / Shell

Запуск произвольных команд. Использовать только если нет спец. модуля. 
- **`command`**: Запускает команду напрямую, без оболочки. Безопаснее, но нельзя использовать пайпы (`|`), перенаправление (`>`, `>>`), переменные среды `$VAR`;
- **`shell`**: Запускает команду через `/bin/sh`. Позволяет использовать всю мощь шелла (пайпы, редиректы), но менее безопасен.

Используйте только если нет готового модуля (`service`, `yum`, `user` и т.д.). Команды по умолчанию **не идемпотентны** (выполняются каждый раз).

**Простая команда без шелла (`command`):**

```yaml
- name: Проверить версию ядра
  command: uname -r
  register: kernel_version

- name: Вывести версию ядра
  debug:
    var: kernel_version.stdout
```

**Сложная команда с пайпами (`shell`):**

```yaml
- name: Посчитать количество процессов nginx
  shell: ps aux | grep nginx | grep -v grep | wc -l
  register: nginx_count
  changed_when: false # Эта команда ничего не меняет в системе, поэтому всегда ok
```

**Изменение переменных окружения:**

```yaml
- name: Запустить скрипт с конкретной переменной окружения
  shell: source /etc/profile && ./deploy.sh
  args:
    chdir: /opt/app/deploy
    executable: /bin/bash
  environment:
    APP_ENV: production
    DB_HOST: 192.168.1.50
```

### Пользователи и группы:

#### User:

Создание, удаление, модификация пользователей.

**Создание обычного пользователя с домашней директорией:**

```yaml
- name: Создать пользователя alice
  user:
    name: alice
    comment: "Alice Developer"      # Поле GECOS (полное имя)
    home: /home/alice               # Домашняя директория (по умолчанию /home/name)
    create_home: yes                # Создать директорию, если нет
    shell: /bin/bash                # Оболочка по умолчанию
    groups: developers              # Дополнительные группы (через запятую можно список)
    append: yes                     # Добавить в группы, не удаляя из существующих
```

**Создание системного пользователя (для сервиса):**

```yaml
- name: Создать системного пользователя для Nginx
  user:
    name: nginx
    system: yes                     # UID из системного диапазона
    home: /var/www                  # Служебная папка
    shell: /sbin/nologin            # Запрет входа в шелл
    create_home: no                 # Не создавать /var/www, если нет (или создайте отдельно)
    group: nginx                    # Основная группа
```

**Генерация SSH-ключа при создании:**

```yaml
- name: Создать пользователя deployer с SSH ключами
  user:
    name: deployer
    generate_ssh_key: yes
    ssh_key_bits: 4096
    ssh_key_file: .ssh/id_rsa       # Относительно home директории
    ssh_key_comment: "deployer@ansible"
```

**Изменение существующего пользователя (смена оболочки, групп):**

```yaml
- name: Повысить права пользователя bob (добавить в sudo)
  user:
    name: bob
    groups: sudo,developers         # Перезаписывает список доп. групп!
    append: no                      # Важно: no означает замену списка групп, 
								    # yes просто добавить группы к существующим
    shell: /bin/bas
```

**Блокировка учетной записи:**

```yaml
- name: Заблокировать уволенного сотрудника
  user:
    name: ex_employee
    password_lock: yes              # Блокирует пароль (добавляет ! в начало хеша)
```

**Удаление пользователя и его домашней директории:**

```yaml
- name: Полностью удалить пользователя
  user:
    name: temp_user
    state: absent
    remove: yes                     # Удаляет также /home/temp_user и почтовый ящик
```

**Задание пароля для пользователя:**

Модуль `user` не принимает пароль в открытом виде (plain text). Ему нужен **хеш** пароля. Мы используем фильтр Jinja2 `password_hash`, чтобы создать хеш прямо во время выполнения плейбука.

```yaml
- name: Создать пользователя с паролем
  user:
    name: new_user
    password: "{{ 'MySuperSecretPassword123' | password_hash('sha512', 'unique_salt_string') }}"
    update_password: on_create  # Критически важно!
    # Если поставить 'always', пароль будет сбрасываться при КАЖДОМ запуске плейбука.
    # 'on_create' меняет пароль только если пользователя еще нет.
```

**Пароль из переменной (для Ansible Vault)**

```yaml
# В файле group_vars/all/vault.yml (зашифровано):
# db_password_plain: "SecretPass!"

- name: Установить пароль из переменной
  user:
    name: db_admin
    password: "{{ db_password_plain | password_hash('sha512') }}"
    update_password: on_create
```

**Принудительная смена пароля при первом входе:**

```yaml
- name: Создать пользователя с временным паролем
  user:
    name: intern
    password: "{{ 'TempPass123' | password_hash('sha512') }}"
    password_expire_max: 0  # Пароль истекает немедленно
    password_expire_min: 0
```

#### Group:

Взаимодействие с группами

**Создание обычной группы:**

```yaml
- name: Создать группу для разработчиков
  group:
    name: developers
    state: present
```

**Создание системной группы (GID < 1000):**

```yaml
- name: Создать системную группу для приложения
  group:
    name: app_runtime
    system: yes        # Выделяет GID из системного диапазона
    state: present
```

**Удаление группы:**

```yaml
- name: Удалить устаревшую группу
  group:
    name: temp_workers
    state: absent
```

#### Authorized_key:

Добавление SSH-ключей пользователю.

**Добавление ключа из файла на контроллере:**

```yaml
- name: Добавить мой публичный ключ пользователю alice
  authorized_key:
    user: alice
    state: present
    key: "{{ lookup('file', '/home/myuser/.ssh/id_rsa.pub') }}"
    # key можно передать и строкой напрямую, но lookup удобнее
```

**Добавление ключа с комментарием и типом:**

```yaml
- name: Добавить ключ с меткой
  authorized_key:
    user: deployer
    key: "ssh-ed25519 AAAAC3Nza... user@laptop"
    comment: "Deploy key from CI/CD server"
    exclusive: no                   # Не удалять другие ключи (добавить к существующим)
```

**Управление ключами через URL (GitHub/GitLab):**

```yaml
- name: Добавить все ключи пользователя с GitHub
  authorized_key:
    user: devops_user
    key: "https://github.com/username.keys"
    state: present
```

**Очистка всех ключей пользователя (Exclusive mode):**

```yaml
- name: Строго задать единственный разрешенный ключ
  authorized_key:
    user: root
    key: "ssh-rsa AAAAB3... admin@secure-pc"
    exclusive: yes                  # Удаляет ВСЕ остальные ключи из authorized_keys
    manage_dir: yes                 # Убедиться, что .ssh существует с правильными правами
```

**Удаление конкретного ключа:**

```yaml
- name: Отозвать доступ у старого ноутбука
  authorized_key:
    user: alice
    key: "ssh-rsa AAAAB3... old-laptop@office"
    state: absent
```


### Сеть и загрузка файлов:

#### Get_url:

Скачивание файла из интернета (HTTP/HTTPS/FTP). Аналог `wget`/`curl`.

**Простое скачивание файла:**

```yaml
- name: Скачать последнюю версию Node.js
  get_url:
    url: https://nodejs.org/dist/v20.10.0/node-v20.10.0-linux-x64.tar.xz
    dest: /tmp/nodejs.tar.xz
    mode: '0644'
```

**Скачивание с проверкой целостности (Checksum):**

```yaml
- name: Скачать бинарник Terraform с проверкой SHA256
  get_url:
    url: https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_linux_amd64.zip
    dest: /usr/local/bin/terraform.zip
    checksum: sha256:https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_SHA256SUMS
    # Можно указать прямую строку хеша: sha256:abc123...
    mode: '0755'
```

**Скачивание с авторизацией (Basic Auth):**

```yaml
- name: Скачать приватный артефакт из Nexus/Artifactory
  get_url:
    url: https://repo.company.com/artifacts/myapp-1.0.jar
    dest: /opt/app/myapp.jar
    url_username: "deploy_user"
    url_password: "secret_pass"
    force_basic_auth: yes
    mode: '0640'
    owner: appuser
```

**Обновление файла только если он изменился:**

```yaml
- name: Обновить скрипт мониторинга только если версия новая
  get_url:
    url: https://scripts.example.com/monitor.sh
    dest: /usr/local/bin/monitor.sh
    force: no          # Не скачивать, если файл уже существует и совпадает по времени/размеру (эвристика)
    # force: yes       # Всегда перезаписывать (по умолчанию)
    mode: '0755'
```

#### Uri:

Выполнение произвольных HTTP/HTTPS запросов. Идеален для проверки API, веб-хуков, получения токенов.

**Проверка доступности API (Health Check):**

```yaml
- name: Проверить статус приложения
  uri:
    url: http://localhost:8080/health
    method: GET
    status_code: 200   # Ожидаем код 200, иначе ошибка
    return_content: yes
  register: health_result

- name: Вывести ответ API
  debug:
    var: health_result.json # Если ответ в JSON, он автоматически распарсится
```

**Отправка POST запроса с JSON телом (Создание ресурса):**

```yaml
- name: Создать пользователя через API
  uri:
    url: https://api.example.com/users
    method: POST
    body_format: json
    body:
      username: "alice"
      email: "alice@example.com"
      role: "admin"
    headers:
      Authorization: "Bearer {{ api_token }}"
      Content-Type: "application/json"
    status_code: 201
  register: create_user_result
```

**Получение токена OAuth:**

```yaml
- name: Получить access token
  uri:
    url: https://auth.example.com/oauth/token
    method: POST
    body_format: form-urlencoded
    body:
      grant_type: client_credentials
      client_id: "{{ client_id }}"
      client_secret: "{{ client_secret }}"
    status_code: 200
  register: auth_response

- name: Сохранить токен в переменную
  set_fact:
    api_token: "{{ auth_response.json.access_token }}"
```

**Игнорирование ошибок SSL (для самоподписанных сертификатов):**

```yaml
- name: Запрос к внутреннему API с самоподписанным cert
  uri:
    url: https://internal-api/status
    validate_certs: no # ⚠️ Используйте только в доверенной сети!
    status_code: 200
```

#### Wait_for:

Приостановить выполнение плейбука до наступления условия (порт открыт, файл появился, процесс запущен).

**Ожидание запуска базы данных (по порту):**

```yaml
- name: Ждать пока PostgreSQL станет доступен
  wait_for:
    host: db.example.com
    port: 5432
    timeout: 60        # Максимальное время ожидания (сек)
    delay: 5           # Подождать 5 сек перед первой проверкой
    state: started     # Ждать пока порт откроется (или 'stopped')
```

**Ожидание появления файла:**

```yaml
- name: Ждать создания флага инициализации
  wait_for:
    path: /var/lib/app/init.done
    state: present
    timeout: 120
```

**Ожидание исчезновения процесса (блокировочного файла):**

```yaml
- name: Ждать удаления lock-файла после обновления
  wait_for:
    path: /var/run/yum.pid
    state: absent
    timeout: 300
```

**Ожидание конкретного текста в ответе порта (Advanced):**

```
- name: Ждать пока SSH выдаст баннер
  wait_for:
    host: localhost
    port: 22
    search_regex: OpenSSH
    timeout: 30
```

#### iptables / ufw:

Управление фаерволами.
- `ufw`: Проще, удобнее для Ubuntu/Debian.
- `iptables`: Мощнее, универсальнее, но сложнее синтаксис.

**Разрешить порты через UFW (Ubuntu):**

```yaml
- name: Разрешить HTTP и HTTPS через UFW
  ufw:
    rule: allow
    port: "{{ item }}"
    proto: tcp
  loop:
    - 80
    - 443

- name: Разрешить доступ к порту приложения только из подсети
  ufw:
    rule: allow
    port: 8080
    proto: tcp
    from_ip: 192.168.1.0/24
```

**Включение UFW с политикой по умолчанию:**

```yaml
- name: Включить UFW и запретить всё входящее по умолчанию
  ufw:
    state: enabled
    direction: incoming
    policy: deny
```

**Правила через iptables (Прямое управление):**

```yaml
- name: Разрешить ping (ICMP) через iptables
  iptables:
    chain: INPUT
    protocol: icmp
    jump: ACCEPT
    comment: "Allow ICMP ping"

- name: Отклонить подключения к порту 23 (Telnet)
  iptables:
    chain: INPUT
    protocol: tcp
    destination_port: 23
    jump: REJECT
    reject_with: icmp-port-unreachable
```

**Сохранение правил iptables:**

```yaml
- name: Сохранить правила iptables (для Debian/Ubuntu)
  command: iptables-save > /etc/iptables/rules.v4
  # Лучше использовать модуль copy/template для файла rules.v4, но это быстрый способ
```

#### Hostname:

Изменение имени хоста.

**Установка постоянного имени хоста:**

Этот модуль меняет имя "на лету" и в конфигурационных файлах (`/etc/hostname`), но не меняет запись в `/etc/hosts`.

```yaml
- name: Установить имя хоста
  hostname:
    name: web-server-01.example.com
    use: systemd # Явно указать систему инициализации (auto обычно работает)
```

**Комбинация с обновлением /etc/hosts:**

```yaml
- name: Обновить имя хоста и локальный hosts
  hosts: all
  tasks:
    - name: Set hostname
      hostname:
        name: "{{ inventory_hostname }}" # Берем имя из инвентаря Ansible

    - name: Update /etc/hosts with new hostname
      lineinfile:
        path: /etc/hosts
        regexp: '^127\.0\.1\.1'
        line: "127.0.1.1 {{ inventory_hostname }} {{ ansible_hostname }}"
        state: present
```

Чтобы команда `hostname` внутри сервера работала корректно, нужно добавить запись в hosts.

#### Getent:

Поиск в системных базах данных. Полезен для получения IP по имени хоста или проверки существования пользователя/группы без выполнения команд shell.

```yaml
- name: Получить IP адрес шлюза по имени
  getent:
    database: hosts
    key: gateway.local
    split: ':'

- name: Вывести IP
  debug:
    var: getent_hosts['gateway.local']
```

#### Sysctl:

Настройка параметров ядра сети. Настройка TCP/IP стека (например, включение IP forwarding для роутера/NAT).

```yaml
- name: Включить IP forwarding
  sysctl:
    name: net.ipv4.ip_forward
    value: '1'
    sysctl_set: yes
    state: present
    reload: yes # Применить сразу (sysctl -p)

- name: Увеличить размер очереди сокетов
  sysctl:
    name: net.core.somaxconn
    value: '65535'
    reload: yes
```

### Отладка и логика:

#### Debug:

Вывод сообщений или переменных на экран.

**Вывод простой строки или переменной:**

```yaml
- name: Показать имя хоста
  debug:
    msg: "Мы работаем на сервере {{ inventory_hostname }}"

- name: Показать значение переменной
  debug:
    var: ansible_default_ipv4.address
```

**Условный вывод (только при высокой детализации):**

```yaml
- name: Показать все факты сети (только при verbose режиме)
  debug:
    var: ansible_facts
  verbosity: 3  # Покажет вывод только если запущено с -vvv или выше
```

**Отладка цикла (показать каждый элемент):**

```yaml
- name: Показать список устанавливаемых пакетов
  debug:
    msg: "Устанавливаем пакет: {{ item }}"
  loop:
    - nginx
    - postgresql
    - redis
```

**Вывод сложной структуры (JSON):**

```yaml
- name: Показать детали пользователя
  debug:
    msg: >
      Пользователь {{ user_info.name }} 
      имеет UID {{ user_info.uid }} 
      и домашнюю директорию {{ user_info.home }}
  vars:
    user_info: "{{ ansible_user_id }}" # Пример использования факта
```

#### Set_fact:

Создание или изменение переменной прямо во время выполнения плейбука. Эта переменная становится доступной всем последующим задачам на этом хосте.

**Создание переменной на основе вычислений:**

```yaml
- name: Вычислить порт приложения
  set_fact:
    app_port: "{{ http_base_port + 100 }}"
  vars:
    http_base_port: 8000

- name: Использовать новую переменную
  debug:
    msg: "Приложение будет слушать порт {{ app_port }}" # Выведет 8100
```

**Извлечение данных из результата команды (register + set_fact):**

```yaml
- name: Получить версию Docker
  command: docker version --format '{{.Server.Version}}'
  register: docker_version_raw

- name: Очистить вывод от лишних пробелов и сохранить в переменную
  set_fact:
    docker_version: "{{ docker_version_raw.stdout | trim }}"

- name: Проверить версию
  debug:
    var: docker_version
```

**Накопление данных в списке (cacheable facts):**

```yaml
- name: Добавить текущий IP в общий список (в памяти)
  set_fact:
    all_web_ips: "{{ all_web_ips | default([]) + [ansible_default_ipv4.address] }}"
    cacheable: yes # Сделать факт доступным для других плейбуков в этой сессии
```

**Динамическое определение ОС:**

```yaml
- name: Определить менеджер пакетов
  set_fact:
    pkg_manager: "apt"
  when: ansible_os_family == "Debian"

- name: Переопределить для RedHat
  set_fact:
    pkg_manager: "yum"
  when: ansible_os_family == "RedHat"
```

#### Assert:

Проверка условий ("гарды"). Если условие ложно (`false`), плейбук останавливается с ошибкой. Идеально для валидации входных данных перед началом работы.

**Проверка версии ОС перед запуском:**

```yaml
- name: Убедиться, что это Ubuntu 20.04 или новее
  assert:
    that:
      - ansible_distribution == "Ubuntu"
      - ansible_distribution_major_version | int >= 20
    fail_msg: "Этот плейбук требует Ubuntu 20.04+. Найден {{ ansible_distribution }} {{ ansible_distribution_version }}"
    success_msg: "Версия ОС подходит: {{ ansible_distribution }} {{ ansible_distribution_version }}"
```

**Проверка наличия обязательных переменных:**

```yaml
- name: Проверить, что задан пароль базы данных
  assert:
    that:
      - db_password is defined
      - db_password | length > 0
    fail_msg: "Критическая ошибка: переменная 'db_password' не определена или пуста!"
```

**Проверка диапазона значений:**

```yaml
- name: Проверить корректность порта
  assert:
    that:
      - app_port | int >= 1024
      - app_port | int <= 65535
    fail_msg: "Порт {{ app_port }} недопустим. Должен быть между 1024 и 65535."
```

#### Fail:

Принудительная остановка плейбука с сообщением об ошибке. В отличие от `assert`, который проверяет условие, `fail` всегда останавливает выполнение (если не обернут в условие `when`).

**Остановка при неверном окружении:**

```yaml
- name: Запретить запуск на Production без подтверждения
  fail:
    msg: "ОШИБКА: Запуск на окружении 'production' запрещен в этом плейбуке! Используйте специальный плейбук для прода."
  when: environment_name == "production"
```

**Остановка, если важный сервис не найден:**

```yaml
- name: Проверить установку критического пакета
  command: which critical_app
  register: app_check
  ignore_errors: yes

- name: Остановить если приложение не найдено
  fail:
    msg: "Критическое приложение не установлено на {{ inventory_hostname }}. Прерывание."
  when: app_check.rc != 0
```

**Генерация понятной ошибки вместо системной:**

```yaml
- name: Попытка подключения к БД
  uri:
    url: "http://db:5432"
    status_code: 200
  register: db_conn
  ignore_errors: yes

- name: Явная ошибка при недоступности БД
  fail:
    msg: "Не удалось подключиться к базе данных. Проверьте сеть и статус СУБД."
  when: db_conn.failed
```

#### Pause:

Приостановка выполнения. Полезно для ручного подтверждения опасных действий или ожидания внешнего процесса.

**Пауза до нажатия Enter (Ручное подтверждение):**

```yaml
- name: Подтверждение перед перезагрузкой серверов
  pause:
    prompt: "ВНИМАНИЕ! Сейчас будут перезапущены все веб-серверы. Нажмите Enter для продолжения или Ctrl+C для отмены."
  
- name: Перезагрузка
  reboot:
    reboot_timeout: 300
```

**Пауза на фиксированное время:**

```yaml
- name: Остановить сервис
  service:
    name: myapp
    state: stopped

- name: Подождать 10 секунд перед удалением файлов
  pause:
    seconds: 10

- name: Очистить кэш
  file:
    path: /var/cache/myapp
    state: absent
```

**Пауза с таймаутом (автоматическое продолжение):**

```yaml
- name: Ждать подтверждения оператора (макс 60 сек)
  pause:
    prompt: "Подтвердите деплой (Enter). Авто-продолжение через 60 сек..."
    timeout: 60
    echo: no # Скрыть ввод пользователя (если бы мы ждали ввод текста)
```

