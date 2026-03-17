
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

### Установка сервера FreeIPA:

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

#### Проверка работоспособности:

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

