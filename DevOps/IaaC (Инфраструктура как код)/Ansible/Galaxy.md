
**Ansible Galaxy** — это центральный репозиторий (хаб) для обмена контентом Ansible. Можно провести аналогию:

- Если **Ansible** — это язык программирования (как Python).
- То **Galaxy** — это **PyPI** (для Python) или **npm** (для Node.js).

Это место, где сообщество и вендоры публикуют готовые **роли**, **коллекции** и **плейбуки**, чтобы вы не писали всё с нуля.

**Роли (Roles):**

- **Что это:** Структурированный набор задач, переменных, файлов и шаблонов для выполнения одной конкретной функции (например, "установить Nginx", "настроить Docker").
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

### Приватный Galaxy (Automation Hub):

В крупных компаниях нельзя использовать публичный Galaxy из соображений безопасности и стабильности.

- **Ansible Automation Hub:** Корпоративное решение от Red Hat (платное). Позволяет создавать приватные репозитории, сертифицировать контент и контролировать версии.
- **Pulp / Galaxy NG:** Open-source решения, которые можно развернуть у себя в инфраструктуре, чтобы иметь свой внутренний "магазин" ролей и коллекций.