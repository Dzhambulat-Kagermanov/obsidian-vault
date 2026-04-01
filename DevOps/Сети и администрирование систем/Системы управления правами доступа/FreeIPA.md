
**FreeIPA** (Free Identity, Policy, Audit) — это открытое решение для централизованного управления идентификацией, аутентификацией и авторизацией в Linux/UNIX-средах. По функциональности оно приближено к Active Directory, но ориентировано на мир открытого ПО.

#### Основные компоненты архитектуры:

| Компонент                     | Назначение                                                               |
| ----------------------------- | ------------------------------------------------------------------------ |
| **389 Directory Server**      | LDAP-сервер — основное хранилище данных о пользователях, группах, хостах |
| **MIT Kerberos KDC**          | Служба аутентификации и единого входа (SSO)                              |
| **Dogtag Certificate System** | Управление SSL/TLS-сертификатами (CA/RA)                                 |
| **BIND (ISC DNS)**            | Управление доменными именами (опционально)                               |
| **NTP**                       | Синхронизация времени — критично для работы Kerberos                     |
| **SSSD**                      | Клиентский компонент для интеграции систем с FreeIPA                     |
| **Веб-интерфейс + CLI**       | Управление через браузер или команду `ipa`                               |

#### Ключевые возможности:

- **Централизованное управление**: пользователи, группы, хосты, сервисы, политики паролей; 
- **Kerberos-аутентификация**: единый вход для пользователей и служб;
- **HBAC (Host-Based Access Control)**: гибкие правила доступа к хостам и сервисам;
- **Управление sudo**: централизованное распространение правил sudo;
- **PKI-инфраструктура**: выпуск и отзыв сертификатов через встроенный CA;
- **Доверие с Active Directory**: интеграция с доменами Windows через Samba (с версии 3.0);
- **Масштабируемость**: поддержка нескольких серверов в режиме multi-master для отказоустойчивости.

#### Сценарии использования:

1. **Корпоративная инфраструктура на Linux** — замена или дополнение к AD для управления доступом;
2. **DevOps / Cloud-среды** — централизованная аутентификация для кластеров, контейнеров, CI/CD;
3. **Образовательные и исследовательские организации** — управление доступом к вычислительным ресурсам;
4. **Гибридные среды** — интеграция Linux-хостов в существующую Windows-инфраструктуру через доверие.

#### Особенности безопасности:

- Все коммуникации защищены TLS;
- Поддержка двухфакторной аутентификации через OTP;
- Делегирование прав администрирования без передачи пароля администратора;
- Аудит действий через интеграцию с rsyslog/journald;
- Возможность отключения анонимного доступа к LDAP.

## Настройка FreeIPA (ALT Linux):

### Настройка сервера FreeIPA:

#### Системные требования:

Сервер FreeIPA следует устанавливать в чистой системе, в которой отсутствуют какие-либо пользовательские настройки для служб: DNS, Kerberos, Apache и Directory Server. FreeIPA при настройке домена перезаписывает системные файлы. FreeIPA создает резервные копии исходных системных файлов в /var/lib/ipa/sysrestore/. При удалении сервера FreeIPA, эти файлы восстанавливаются.

##### RAM:

Для установки с CA требуется минимально 1,2 ГБ RAM. Типичные требования к оперативной памяти:
- для 10 000 пользователей и 100 групп: не менее 4 ГБ RAM и 4 ГБ Swap.
- для 100 000 пользователей и 50 000 групп: не менее 16 ГБ RAM и 4 ГБ Swap.

##### DNS:

Без правильно настроенной конфигурации DNS не будут работать должным образом Kerberos и SSL.

> **Внимание!** Домен DNS не может быть изменен после установки.


Установщик FreeIPA довольно требователен к настройке DNS. Установщик выполняет следующие проверки:
- имя узла не может быть localhost или localhost6.
- имя узла должно быть полным (ipa.example.test)
- имя узла должно быть разрешимым.
- обратный адрес должен совпадать с именем хоста.

Не используйте существующий домен, если вы не являетесь владельцем домена. Рекомендуется использовать зарезервированный домен верхнего уровня из RFC2606 для частных тестовых установок, например ipa.test.


#### Подготовка сервера:

Для корректной работы сервера необходимо, задать ему полное доменное имя (FQDN). Имя серверу можно назначить командой:

```bash
hostnamectl set-hostname ipa.example.test
```

> **Примечание:** IP-адрес сервера не должен изменяться.

Нужно отключить работающий на порту 8080 ahttpd во избежание конфликтов с разворачиваемым tomcat и отключить HTTPS в Apache2:

```bash
# service ahttpd stop
# a2dissite 000-default_https
# a2disport https
# service httpd2 condreload
```

**Установка пакетов:**

```bash
apt-get update && apt-get install freeipa-server

# Если сервер FreeIPA должен включать DNS-сервер, необходимо также установить пакет freeipa-server-dns:
apt-get install freeipa-server-dns
```

**Настройка файла /etc/hosts:**

FreeIPA требует, чтобы сервер мог разрешить свое имя в IP-адрес.

```bash
192.168.1.10 ipa.example.com ipa
```

**Настройка chronyd:**

FreeIPA использует `chronyd` (или `ntpd`). Убедитесь, что служба активна.

```bash
systemctl enable --now chronyd
timedatectl status
```

> Убедитесь, что время точное (разница не более 5 минут, лучше секунды).

#### Установка сервера с интегрированным DNS:

Преимущества установки сервера FreeIPA со встроенным DNS:

- Можно автоматизировать большую часть обслуживания и управления записями DNS, используя инструменты FreeIPA. Например, записи DNS SRV создаются во время установки, а затем автоматически обновляются;
- Можно иметь стабильное соединение с остальной частью Интернета, настроив глобальные серверы пересылки во время установки сервера FreeIPA. Глобальные серверы пересылки также полезны для доверительных отношений с Active Directory;
- Можно настроить обратную зону DNS, чтобы электронные письма из вашего домена не считались спамом почтовыми серверами за пределами домена FreeIPA.

Ограничения установки сервера FreeIPA со встроенным DNS:

- FreeIPA DNS не предназначен для использования в качестве DNS-сервера общего назначения. Некоторые расширенные функции DNS не поддерживаются.

**Интерактивная установка:**

Для запуска интерактивной установки выполнить команду:

```bash
ipa-server-install
```

> **Примечание:** Если в команде ipa-server-install в конфигурации по умолчанию не указаны CA параметры, например, --external-ca или --ca-less, сервер FreeIPA устанавливается с интегрированным CA.


На первый вопрос, нужно ли сконфигурировать DNS-сервер BIND, следует ответить утвердительно:

```bash
Do you want to configure integrated DNS (BIND)? [no]: yes
```

Далее нужно указать имя узла, на котором будет установлен сервер FreeIPA, доменное имя и пространство Kerberos (можно нажать Enter для выбора значений по умолчанию):

```bash
Server host name [ipa.example.test]:
Please confirm the domain name [example.test]:
Please provide a realm name [EXAMPLE.TEST]:
```

> **Внимание!** Эти имена нельзя изменить после завершения установки.

Задаем пароль для Directory Manager (cn=Directory Manager):

```bash
Directory Manager password:
Password (confirm):
```

Задаем пароль для администратора FreeIPA (будет создана учетная запись admin с правами администратора):

```bash
IPA admin password:
Password (confirm):
```

Для настройки DNS на первый запрос, нужно ли настроить перенаправления, необходимо ответить «да»:

```bash
Do you want to configure DNS forwarders? [yes]:
```

Система предложит сначала использовать DNS-серверы из настроек сети (если они прописаны) — если это устроит, можно оставить значение по умолчанию:

```bash
Do you want to configure these servers as DNS forwarders? [yes]:
```

Также можно добавить дополнительные серверы:

```bash
Enter an IP address for a DNS forwarder, or press Enter to skip: 8.8.8.8
```

Оставляем значение по умолчанию для попытки найти обратные зоны:

```bash
Do you want to search for missing reverse zones? [yes]
```

> **Примечание:** Использование FreeIPA для управления обратными зонами не является обязательным. Для этой цели можно использовать внешнюю службу DNS.


Далее система выведет информацию о конфигурации и попросит ее подтвердить:

```bash
The IPA Master Server will be configured with:
Hostname:       ipa.example.test
IP address(es): 192.168.0.113
Domain name:    example.test
Realm name:     EXAMPLE.TEST

The CA will be configured with:
Subject DN:   CN=Certificate Authority,O=EXAMPLE.TEST
Subject base: O=EXAMPLE.TEST
Chaining:     self-signed

BIND DNS server will be configured to serve IPA domain with:
Forwarders:       8.8.8.8
Forward policy:   only
Reverse zone(s):  0.168.192.in-addr.arpa.

Continue to configure the system with these values? [no]: yes
```

> Установка займет 5–15 минут. В конце вы увидите сообщение:

```bash
==============================================================================
Setup complete

Next steps:
    1. You must make sure these network ports are open:
        TCP Ports:
          * 80, 443: HTTP/HTTPS
          * 389, 636: LDAP/LDAPS
          * 88, 464: kerberos
          * 53: bind
        UDP Ports:
          * 88, 464: kerberos
          * 53: bind
          * 123: ntp

    2. You can now obtain a kerberos ticket using the command: 'kinit admin'
       This ticket will allow you to use the IPA tools (e.g., ipa user-add)
       and the web user interface.

Be sure to back up the CA certificates stored in /root/cacert.p12
These files are required to create replicas. The password for these
files is the Directory Manager password
The ipa-server-install command was successful
```

##### Проверка работоспособности:

**Статус служб:**

Все службы должны быть в статусе `running`

```bash
ipactl status
```

**Тест аутентификации (Kerberos):**

```bash
kinit admin
# Введите пароль администратора
klist
# Должен показать билет в кэше
```

**Проверка DNS:**

```bash
ping -c 3 ipa.example.com

dig _ldap._tcp.example.com SRV
```


#### Дополнительные настройки DNS:

Все дополнительные настройки нужно вводить в файл `/etc/bind/ipa-options-ext.conf`. Пример:

```vim
forwarders: { 77.88.8.8; 8.8.8.8; };
allow-query: { any; };
```

После чего выполнить перезагрузку сервисов freeipa:

```bash
ipactl restart
```

### Подключение клиента  к  домену FreeIPA:

**Установка пакетов:**

```bash
apt-get update && apt-get install freeipa-client task-auth-freeipa
```

#### Ввод хоста через терминал:

Клиентские компьютеры должны быть настроены на использование DNS-сервера, который был сконфигурирован на сервере FreeIPA во время его установки. В сетевых настройках необходимо указать использовать сервер FreeIPA для разрешения имен. Эти настройки можно выполнить как в графическом интерфейсе, так и в консоли.

В сетевых настройках клиентского хоста необходимо добавить ip freeipa сервера в dns сервера.

**Интерактивная установка.** Запускаем скрипт настройки клиента в интерактивном режиме:

```bash
ipa-client-install --mkhomedir
```

Можно добавить параметр `--enable-dns-updates`, чтобы обновить записи DNS с помощью IP-адреса клиентской системы, если выполняется одно из следующих условий:

- Сервер FreeIPA, на котором будет зарегистрирован клиент, был установлен со встроенным DNS;
- DNS-сервер в сети принимает обновления записей DNS по протоколу GSS-TSIG.

Скрипт установки должен автоматически найти необходимые настройки на FreeIPA сервере, вывести их и спросить подтверждение для найденных параметров:

```bash
This program will set up IPA client.
Version 4.9.7

Discovery was successful!
Do you want to configure CHRONY with NTP server or pool address? [no]: 
Client hostname: comp08.example.test
Realm: EXAMPLE.TEST
DNS Domain: example.test
IPA Server: ipa.example.test
BaseDN: dc=example,dc=test

Continue to configure the system with these values? [no]: yes
```

Затем запрашивается имя пользователя, имеющего право вводить машины в домен, и его пароль (можно использовать администратора по умолчанию, который был создан при установке сервера):

```bash
User authorized to enroll computers: admin
Password for admin@EXAMPLE.TEST:
```

#### Ввод хоста в домен через Центр управления системой:

Для ввода рабочей станции в домен FreeIPA, необходимо в Центре управления системой перейти в раздел «Пользователи» → «Аутентификация».

![[Freeipa2.png]]

В открывшемся окне необходимо ввести имя пользователя, имеющего право вводить машины в домен, и его пароль и нажать кнопку «ОК»:

![[Freeipa_pass.png]]

В случае успешного подключения, будет выведено соответствующее сообщение:

![[Freeipa3.png]]
#### Проверка работоспособности:

Создаем пользователя в веб-интерфейсе FreeIPA (например, `testuser`), затем на клиенте:

```bash
su - testuser
# Если вход прошел успешно — домен работает!
id testuser
```

### Основные команды для работы с FreeIPA:

Утилита `ipa` — это основной инструмент администрирования. Все команды имеют структуру:
`ipa <объект>-<действие> [параметры]`.

#### Аутентификация и сессия:

```bash
# Получить билет Kerberos (начать сессию)
kinit admin

# Посмотреть активный билет
klist

# Уничтожить билет (выйти)
kdestroy

# Проверить, аутентифицирован ли текущий пользователь
ipa ping
```

#### Управление пользователями:

```bash
# Создать пользователя
ipa user-add --first=Иван --last=Иванов --email=ivanov@example.com iivanov

# Информация о пользователе
ipa user-show iivanov

# Изменить атрибуты пользователя
ipa user-mod iivanov --email=new@example.com --title="System Admin"

# Сбросить пароль (требует прав администратора)
ipa passwd iivanov

# Смена пароля пользователя
ipa user-mod a.smith --password

# Деактивировать / активировать пользователя
ipa user-disable iivanov
ipa user-enable iivanov

# Удалить пользователя “в корзину”, т.е. с возможностью восстановления
ipa user-del --preserve a.smith

# Восстановить пользователя “из корзины”
ipa user-undel a.smith

# Поиск пользователей
ipa user-find --last=Иванов
ipa user-find --all | grep -A5 "uid:"

# Добавить пользователя в группу
ipa group-add-member developers --users=iivanov

# Просмотреть членство пользователя в группах
ipa user-show iivanov | grep -A10 "Member of groups"
```

#### Управление группами:

```bash
# Создать группу
ipa group-add --desc="Разработчики" developers

# Просмотреть группу
ipa group-show developers

# Добавить/удалить участников
ipa group-add-member developers --users=iivanov,petrov
ipa group-remove-member developers --users=petrov

# Вложенные группы (добавить одну группу в другую)
ipa group-add-member admins --groups=developers

# Найти группы
ipa group-find --desc="разработ"
```

#### Управление хостами:

```bash
# Добавить хост вручную (если не через ipa-client-install)
ipa host-add client1.example.com --desc="Dev workstation"

# Просмотреть хост
ipa host-show client1.example.com

# Сгенерировать одноразовый пароль для присоединения хоста
ipa host-add --random client2.example.com
# Пароль будет выведен в консоль — используйте его при ipa-client-install

# Удалить хост из домена
ipa host-del client1.example.com

# Найти хосты по шаблону
ipa host-find --fqdn=*.example.com

# Добавить сервис к хосту (для Kerberos SPN)
ipa service-add HTTP/client1.example.com
```

#### Управление сервисами и ключами:

```bash
# Добавить сервис (для аутентификации приложений)
ipa service-add postgres/db.example.com

# Получить keytab для сервиса
ipa-getkeytab -s ipa.example.com -p postgres/db.example.com -k /etc/krb5.keytab

# Просмотреть сервис
ipa service-show postgres/db.example.com

# Деактивировать сервис
ipa service-disable postgres/db.example.com
```

#### Управление паролями и политиками:

```bash
# Просмотреть глобальную политику паролей
ipa config-show

# Создать новую политику паролей
ipa pwpolicy-add --minlength=12 --maxlife=90 --minclasses=4 devs

# Применить политику к группе
ipa pwpolicy-add devs --minlength=14 --maxlife=60

# Просмотреть политику для пользователя
ipa pwpolicy-show iivanov
```

#### HBAC (Host-Based Access Control) — правила доступа к хостам:

```bash
# Создать правило доступа
ipa hbacrule-add --desc="Доступ разработчиков к dev-серверам" dev-access

# Добавить пользователей/группы в правило
ipa hbacrule-add-user dev-access --groups=developers

# Добавить целевые хосты
ipa hbacrule-add-host dev-access --hosts=dev1.example.com,dev2.example.com

# Разрешить все сервисы (или конкретные, например, ssh)
ipa hbacrule-add-service dev-access --services=sshd

# Включить правило
ipa hbacrule-enable dev-access

# Протестировать правило (проверка доступа)
ipa hbactest --user=iivanov --host=dev1.example.com --service=sshd
```

#### Управление sudo через FreeIPA:

```bash
# Создать команду для sudo
ipa sudocmd-add --desc="Перезапуск nginx" /usr/bin/systemctl
ipa sudocmd-add-member /usr/bin/systemctl --sudocmds="restart nginx"

# Создать группу команд
ipa sudocmdgroup-add --desc="Управление веб-серверами" web-admin

# Создать правило sudo
ipa sudorule-add --desc="Разработчики могут рестартить nginx" web-restart

# Добавить пользователей, хосты и команды в правило
ipa sudorule-add-user web-restart --groups=developers
ipa sudorule-add-host web-restart --hosts=web1.example.com
ipa sudorule-add-sudocmd web-restart --sudocmds=/usr/bin/systemctl

# Разрешить выполнение без пароля (опционально)
ipa sudorule-add-option web-restart --sudooption=!authenticate

# Включить правило
ipa sudorule-enable web-restart
```

#### Управление встроенным DNS (если включен `--setup-dns`):

```bash
# Изменить опцию в конфиге
ipa dnszone-mod au.team. --allow-query="10.1.0.0/16; 127.0.0.1"

# Просмотр информации о глобальной конфигурации
ip dnsconfig-show

# Установление forwarder-ов
ipa dnsconfig-mod --forward=only --forwarder=8.8.8.8 --forwarder=8.8.4.4

# Создать прямую зону
ipa dnszone-add example.com --admin-email=admin@example.com

# Добавить A-запись
ipa dnsrecord-add example.com www --a-rec=192.168.1.50

# Добавить CNAME
ipa dnsrecord-add example.com portal --cname-rec=www.example.com

# Добавить SRV-запись (для сервисов)
ipa dnsrecord-add example.com _ldap._tcp --srv-rec="0 100 389 ipa.example.com"

# Просмотреть записи зоны
ipa dnszone-show example.com
ipa dnsrecord-find example.com

# Удалить запись
ipa dnsrecord-del example.com www --a-rec=192.168.1.50

# Создать обратную зону (PTR)
ipa dnszone-add 1.168.192.in-addr.arpa.
ipa dnsrecord-add 1.168.192.in-addr.arpa. 50 --ptr-rec=www.example.com
```

#### Репликация и управление серверами:

```bash
# Показать топологию реплик
ipa topologysegment-find

# Добавить реплику (на новом сервере)
ipa-replica-install --principal=admin

# Принудительная синхронизация с другой репликой
ipa-replica-manage re-initialize --from=ipa2.example.com

# Проверить статус соглашения о репликации
ipa-replica-manage list

# Удалить реплику из топологии
ipa-replica-manage del ipa2.example.com
```

#### Резервное копирование и восстановление:

```bash
# Создать полный бэкап (включая LDAP, PKI, конфиги)
ipa-backup

# Восстановить из бэкапа (только в режиме offline!)
ipa-restore /var/lib/ipa/backup/ipa-backup-2024-03-17-10-30-00

# Список доступных бэкапов
ls -la /var/lib/ipa/backup/
```

> `ipa-restore` останавливает все службы — выполняйте только в окне обслуживания.