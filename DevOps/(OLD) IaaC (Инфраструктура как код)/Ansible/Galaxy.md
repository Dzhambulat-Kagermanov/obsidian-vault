
**Ansible Galaxy** — это центральный репозиторий (хаб) для обмена контентом Ansible. Можно провести аналогию:

- Если **Ansible** — это язык программирования (как Python).
- То **Galaxy** — это **PyPI** (для Python) или **npm** (для Node.js).

Это место, где сообщество и вендоры публикуют готовые **роли**, **коллекции** и **плейбуки**, чтобы вы не писали всё с нуля.

**Роли (Roles):**

- **Что это:** Структурированный набор задач, переменных, файлов и шаблонов для выполнения одной конкретной функции (например, "установить Nginx", "настроить Docker"). Это «разбитый на части плейбук» (набор задач, переменных, файлов), который можно переиспользовать.
- **Структура:** `tasks/`, `vars/`, `templates/`, `handlers/`.
- **Использование:** Подключаются в плейбуке через ключ `roles:`.
- **Статус:** Сейчас считаются устаревающим форматом для сложного софта, но всё еще популярны для простых задач. Новые проекты лучше делать в виде Коллекций.

**Коллекции (Collections):**

- **Что это:** Упакованный контент более высокого уровня. Коллекция может содержать:
    - Несколько ролей.
    - Модули (плагины для управления конкретным оборудованием или облаком, например `cisco.ios`, `community.aws`).
    - Плагины (lookup, filter, test).
    - Документацию и тесты.
- **Именование:** Всегда имеют пространство имен: `namespace.collection_name` (например, `community.general`, `ansible.netcommon`).
- **Использование:** Подключаются через ключ `collections:` в плейбуке или устанавливаются глобально.

Основная команда для работы — `ansible-galaxy`.

### Установка коллекций

Чаще всего зависимости проекта описывают в файле `requirements.yml`.

```yaml
collections:
  - name: community.general       # Имя коллекции
    version: ">=6.0.0"            # Версия (рекомендуется фиксировать!)
  
  - name: ansible.posix           # Еще одна коллекция
  
  - name: cisco.ios               # Коллекция от вендора
    source: https://galaxy.ansible.com # Источник (по умолчанию этот)

roles:
  - name: geerlingguy.nginx       # Старая добрая роль от Jeff Geerling
    version: 3.1.0
  - name: geerlingguy.docker
```

Запускается в корне проекта:

```bash
ansible-galaxy install -r requirements.yml
```

_Где будет установлено?_ По умолчанию: `~/.ansible/collections/ansible_collections/` и `~/.ansible/roles/`. Можно изменить путь в `ansible.cfg` (`collections_paths`, `roles_path`) или указать флаг `-p ./collections`.

#### Поиск контента:

```bash
# Поиск коллекций
ansible-galaxy collection search docker

# Поиск ролей
ansible-galaxy role search nginx

# Просмотр информации о коллекции
ansible-galaxy collection info community.general
```

#### Инициализация своего проекта

Вы можете создать скелет своей роли или коллекции для разработки:

```bash
# Создать структуру новой роли
ansible-galaxy role init my_custom_role

# Создать структуру новой коллекции
ansible-galaxy collection init my_namespace.my_collection
```

### Популярные коллекции и роли, которые нужно знать:

| Название                | Тип       | Описание                                                                                                                                                                               |
| ----------------------- | --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`community.general`** | Коллекция | Огромный набор общих модулей, фильтров и ролей, которые раньше входили в ядро Ansible. **Обязательна к установке.**                                                                    |
| **`ansible.posix`**     | Коллекция | Модули для управления Linux/Unix системами (iptables, cron, known_hosts и т.д.).                                                                                                       |
| **`community.docker`**  | Коллекция | Все модули для работы с Docker и Docker Compose (`docker_container`, `docker_image`).                                                                                                  |
| **`kubernetes.core`**   | Коллекция | Управление кластерами K8s.                                                                                                                                                             |
| **`geerlingguy.*`**     | Роли      | Набор ролей от Джеффа Гирлинга (автора книги "Ansible for DevOps"). Очень качественные роли для установки ПО (nginx, mysql, java, docker). Отличный пример того, как надо писать роли. |
| **`aws.*` / `azure.*`** | Коллекции | Официальные коллекции для управления облаками AWS и Azure.                                                                                                                             |

### Жизненный цикл работы с Galaxy в проекте

1. **Создание файла зависимостей:** Вы создаете `requirements.yml` в корне репозитория вашего проекта.
2. **Фиксация версий:** Вы **обязательно** указываете конкретные версии (`version: 1.2.3`), а не `latest`. Это гарантирует, что обновление коллекции завтра не сломает ваш продакшен сегодня.
3. **CI/CD Интеграция:** На этапе сборки пайплайна (перед запуском плейбуков) выполняется команда:

```bash
ansible-galaxy install -r requirements.yml -p ./collections
```

4. **Настройка `ansible.cfg`:** Указываете Ansible искать коллекции в папке проекта:

```yaml
[defaults]
collections_paths = ./collections
roles_path = ./roles
```

5. **Использование в коде:**

```yaml
---
- hosts: all
  collections:
    - community.general
    - community.docker
  
  roles:
    - geerlingguy.nginx
  
  tasks:
    - name: Запустить контейнер
      community.docker.docker_container:
        name: web
        image: nginx
```

### Публикация своего контента:

1. **Подготовка:** Убедитесь, что структура папок правильная (используйте `ansible-galaxy init`).
2. **Galaxy YAML:** Заполните файл `galaxy.yml` (для коллекций) или `meta/main.yml` (для ролей) метаданными (автор, лицензия, зависимости, теги).
3. **Аккаунт:** Зарегистрируйтесь на [galaxy.ansible.com](https://galaxy.ansible.com/) через GitHub.
4. **Создание Namespace:** Создайте свое пространство имен (обычно совпадает с вашим логином GitHub).

```yaml
# Войти (получить токен на сайте)
ansible-galaxy login

# Сборка пакета (создает .tar.gz файл)
ansible-galaxy collection build 
# или для роли
ansible-galaxy role build

# Публикация
ansible-galaxy collection publish my_namespace-my_collection-1.0.0.tar.gz
```

Теперь вашу коллекцию может установить любой человек в мире командой:

```bash
ansible-galaxy collection install my_namespace.my_collection
```

#### Пример простой Кастомной Роли:

**Задача:** Создать роль `my_webserver`, которая устанавливает Nginx и кладет файл `index.html`.

##### Шаг 1: Генерация структуры:

Команда: `ansible-galaxy role init my_webserver` Создаст папку `my_webserver/` со следующей структурой (нам нужны только 3 файла):

```Structure
my_webserver/
├── tasks/
│   └── main.yml      <-- Сюда пишем задачи
├── defaults/
│   └── main.yml      <-- Сюда пишем переменные по умолчанию
└── files/
    └── index.html    <-- Статический файл
```

##### Шаг 2: Наполнение файлами:

**Файл: `my_webserver/defaults/main.yml`** (Переменные, которые легко переопределить снаружи)

```yaml
---
web_port: 80
web_user: www-data
welcome_message: "Hello from my custom Role!"
```

**Файл: `my_webserver/files/index.html`** (Просто HTML файл)

```yaml
<h1>{{ welcome_message }}</h1>
<p>Served by Ansible Role</p>
```

_Примечание:_ Так как это папка `files`, шаблонизация Jinja2 (`{{ }}`) здесь **не сработает**. Файл скопируется как есть. 

Если нужны переменные, этот файл надо перенести в папку `templates/` и использовать модуль `template`. Для примера оставим так или представим, что мы используем модуль `copy` без переменных внутри файла, либо заменим на шаблон. 

_Исправление для примера:_ Давайте лучше используем шаблон, чтобы показать силу Ansible. Переместим файл в `templates/index.html.j2`:

```Jinja2
<h1>{{ welcome_message }}</h1>
<p>Port: {{ web_port }}</p>
```

**Файл: `my_webserver/tasks/main.yml`** (Логика роли)

```yaml
---
- name: Установить Nginx
  apt:
    name: nginx
    state: present
  become: yes

- name: Разместить приветственную страницу
  template:
    src: index.html.j2
    dest: /var/www/html/index.html
    owner: "{{ web_user }}"
    mode: '0644'
  become: yes
  notify: Restart Nginx

- name: Убедиться, что Nginx запущен
  service:
    name: nginx
    state: started
    enabled: yes
  become: yes

# Хендлеры тоже могут быть внутри роли!
handlers:
  - name: Restart Nginx
    service:
      name: nginx
      state: restarted
    become: yes
```

##### Шаг 3. Использование роли в основном проекте:

В вашем главном плейбуке (`site.yml`):

```yaml
---
- hosts: webservers
  roles:
    - role: my_webserver
      vars:
        welcome_message: "Привет из моей первой роли!"
        web_port: 8080
```

#### Пример кастомной коллекции:

**Задача:** Создать коллекцию `mycompany.utils`, которая содержит:

1. Ту же самую роль (упакованную внутри);
2. Простой фильтр (функцию для обработки текста).

##### Шаг 1. Генерация структуры:

Команда: `ansible-galaxy collection init mycompany.utils` Создаст папку `mycompany-utils/`.

Структура, которую мы будем использовать:

```Structure
mycompany-utils/
├── galaxy.yml            <-- Метаданные (имя, версия, автор)
├── plugins/
│   └── filter/
│       └── my_filters.py <-- Наш кастомный фильтр
├── roles/
│   └── my_webserver/     <-- Наша роль внутри коллекции
│       ├── tasks/
│       │   └── main.yml
│       └── templates/
│           └── index.html.j2
└── README.md
```

##### Шаг 2. Наполнение файлами:

**Файл: `galaxy.yml`** (Самое важное: имя и версия)

```yaml
namespace: mycompany
name: utils
version: 1.0.0
readme: README.md
authors:
  - Your Name
description: "Моя первая коллекция с утилитами"
license:
  - MIT
dependencies: {}
```

**Файл: `plugins/filter/my_filters.py`** (Пишем код на Python для новой функции)

```python
def reverse_string(string):
    """Разворачивает строку задом наперед"""
    return string[::-1]

class FilterModule(object):
    def filters(self):
        return {
            'reverse_string': reverse_string
        }
```

_Теперь в шаблонах можно будет делать: `{{ 'hello' | mycompany.utils.reverse_string }}`_

**Файл: `roles/my_webserver/tasks/main.yml`** (Копируем сюда логику из примера роли выше, структура внутри коллекции такая же, как у обычной роли).

##### Шаг 3. Сборка и установка:

Чтобы использовать коллекцию, её нужно собрать в архив `.tar.gz`.

1. Зайти в папку коллекции: `cd mycompany-utils`
2. Собрать: `ansible-galaxy collection build`
    - Создастся файл `mycompany-utils-1.0.0.tar.gz`.
3. Установить локально для теста:

```bash
ansible-galaxy collection install mycompany-utils-1.0.0.tar.gz -p ./collections
```

##### Шаг 4. Использование коллекции в проекте:

В плейбуке (`site.yml`):

```yaml
---
- hosts: localhost
  gather_facts: no
  
  # Подключаем коллекцию
  collections:
    - mycompany.utils

  tasks:
    # 1. Использование фильтра из коллекции
    - name: Использовать кастомный фильтр
      debug:
        msg: "{{ 'Ansible Collection' | mycompany.utils.reverse_string }}"
      # Вывод: noitcelloC elbisnA

    # 2. Использование роли из коллекции
    # Обратите внимание на полное имя: namespace.collection.role_name
    - name: Запустить веб-роль из коллекции
      include_role:
        name: mycompany.utils.my_webserver
      vars:
        welcome_message: "Работает изнутри коллекции!"
```

### Приватный Galaxy (Automation Hub):

В крупных компаниях нельзя использовать публичный Galaxy из соображений безопасности и стабильности.

- **Ansible Automation Hub:** Корпоративное решение от Red Hat (платное). Позволяет создавать приватные репозитории, сертифицировать контент и контролировать версии.
- **Pulp / Galaxy NG:** Open-source решения, которые можно развернуть у себя в инфраструктуре, чтобы иметь свой внутренний "магазин" ролей и коллекций.