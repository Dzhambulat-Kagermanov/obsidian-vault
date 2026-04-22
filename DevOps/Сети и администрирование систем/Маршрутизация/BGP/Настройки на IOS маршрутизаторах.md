
При настройке BGP (например, в FRR, Bird или Cisco) вы оперируете следующими ключевыми понятиями:
* **ASN (Autonomous System Number):** Ваш номер и номер соседа. _Конфиг:_ `router bgp 65001` (ваш AS), `neighbor 10.0.0.2 remote-as 65002` (сосед).
* **Network Advertisement:** Какие сети вы объявляете миру. _Важно:_ В BGP нельзя просто написать `network 0.0.0.0`. Маршрут должен уже существовать в таблице routingu (часто создают статический маршрут в null0 или loopback), чтобы BGP мог его анонсировать.

### ebgp-requires-policy:

Отключает защиту, которая по умолчанию запрещает обмен маршрутами с eBGP-соседями, если не настроены явные политики входа/выхода

**Зачем нужно:**

- **Безопасность по умолчанию:** В современных версиях FRR (и Cisco IOS XR) включена защита: «Не верь никому, пока не напишешь фильтр». Если вы настроили соседа, но забыли фильтр, маршруты не пройдут. Это спасает от случайных утечек.
- **Лабораторные условия / Быстрый старт:** В учебной среде или при быстром тестировании вам может быть лень писать фильтры для каждого соседа. Вы хотите, чтобы «просто работало». Эта команда отключает требование обязательного фильтра.

```Router
router bgp 65000
  ! Отключаем требование обязательной политики для всех eBGP соседей
  no bgp ebgp-requires-policy
  
  neighbor 10.0.0.1 remote-as 65001
  ! Теперь маршруты пойдут даже без "neighbor ... route-map ..."
```

### Атрибуты влияния на выбор пути (Traffic Engineering):

#### LOCAL_PREF:

**LOCAL_PREF (Local Preference):** Самый важный атрибут **внутри** вашей AS. Показывает, какой выход во внешний мир предпочтительнее. Задается вами и не передается соседям. Чем выше значение, тем лучше. Пример конфигурации:

```Router
! Создаем карту маршрутов с именем SET_HIGH_PREF
route-map SET_HIGH_PREF permit 10
  ! Действие: установить LOCAL_PREF в 200 (по умолчанию 100)
  set local-preference 200

route-map SET_LOW_PREF permit 10
  set local-preference 50

router bgp 65001
  ! Применяем высокую метку ко всем маршрутам ОТ Провайдера А (IP 10.0.0.1)
  neighbor 10.0.0.1 route-map SET_HIGH_PREF in
  
  ! Применяем низкую метку ко всем маршрутам ОТ Провайдера Б (IP 10.0.0.2)
  neighbor 10.0.0.2 route-map SET_LOW_PREF in
```

#### AS_PATH Prepending

**AS_PATH Prepending:** Искусственное удлинение пути. Вы добавляете свой номер AS несколько раз (например, `65001 65001 65001`), чтобы соседи считали этот путь «длинным» и шли через другого провайдера. Используется для входящего трафика.  Пример конфигурации:

```Router
route-map PREPEND_BACKUP permit 10
  ! Добавляем наш номер AS (65001) три раза. 
  ! Итоговый путь будет выглядеть как: ... 65001 65001 65001 65001
  set as-path prepend 65001 65001 65001

router bgp 65001
  ! Применяем при отправке маршрутов НАРУЖУ через резервного провайдера
  neighbor 10.0.0.2 (резервный) route-map PREPEND_BACKUP out
  
  ! Через основного ничего не делаем (путь короткий, его выберут)
  neighbor 10.0.0.1 (основной) route-map NONE out
```

#### MED:

**MED (Multi-Exit Discriminator):** Подсказка соседней AS, через какой вход к вам лучше заходить. Передается соседям, но они могут её игнорировать. Чем **меньше** значение, тем лучше. Пример конфигурации:

```Router
route-map SET_MED_LOW permit 10
  set metric 10  ! Лучший путь

route-map SET_MED_HIGH permit 10
  set metric 500 ! Худший путь

router bgp 65001
  neighbor 10.0.0.1 route-map SET_MED_LOW out
  neighbor 10.0.0.2 route-map SET_MED_HIGH out
```

#### Communities:

**Communities:** Специальные метки (теги), которые вы вешаете на маршруты, чтобы договориться с провайдером о действиях (например, «не транслировать дальше», «предпочесть этот путь»).  Сам по себе атрибут Community **ничего не делает**. Он просто «путешествует» вместе с маршрутом. Магию совершает тот роутер, который **читает** эту метку и настроен реагировать на неё определенным образом. Обычно его записывают в формате `ASN:VALUE`.
- **ASN:** Номер автономной системы, которая придумала эту метку (обычно ваша).
- **VALUE:** Код действия или категории, который вы придумали сами.

Типы Community:
*  **Стандартные (Well-known):** Понятны любому оборудованию в мире без договоренностей.
	- `no-export` (0xFFFFFF01): «Не передавай этот маршрут дальше своей AS». (Идеально для пиринга).
	- `no-advertise` (0xFFFFFF02): «Не передавай этот маршрут вообще никому (даже внутри своей AS)».
	- `no-export-subconfed` (0xFFFFFF03): Специфично для конфедераций.
	- `blackhole` (часто `65535:666` или подобный, но есть стандарт RFC7999 `65535:666`): «Урони весь трафик к этому префиксу в никуда (защита от DDoS)».
*  **Приватные (Custom):** Выдуманные вами значения. Их смысл известен **только** вам и вашему провайдеру (или партнеру), если вы заранее договорились.
	- Пример: Вы договариваетесь с провайдером: «Если я повешу на маршрут метку `65001:500`, пожалуйста, установи для него `LOCAL_PREF=500` у себя».

Пример конфигурации **Community**:
```Router
route-map TAG_NO_EXPORT permit 10
  ! Добавляем стандартный коммьюнити no-export
  set community no-export
  
  ! Важно: иногда нужно явно разрешить отправку коммьюнити
  set community additive 

router bgp 65001
  neighbor 10.0.0.1 route-map TAG_NO_EXPORT out
  ! Разрешаем соседу принимать коммьюнити
  neighbor 10.0.0.1 send-community
```

Обработка **Community** атрибутов от других маршрутизаторов:

1. Разрешить передачу/прием Community:

```Router
router bgp 65000
  neighbor 10.0.0.1 remote-as 65001
  ! Эта команда критична:
  neighbor 10.0.0.1 send-community 
  ! Или send-community extended для больших меток (RFC 4360)
  neighbor 10.0.0.1 send-community extended
```

2. Написать логику обработки (Route-map на вход):

```Router
! Определяем списки community атрибутов для удобства
ip community-list standard CUST_HIGH permit 65001:100
ip community-list standard CUST_LOW permit 65001:200

! Создаем карту обработки
route-map HANDLE_CLIENT_COMMUNITIES permit 10
  ! Если видим метку 65001:100 -> ставим высокий приоритет
  match community CUST_HIGH
  set local-preference 200

route-map HANDLE_CLIENT_COMMUNITIES permit 20
  ! Если видим метку 65001:200 -> ставим низкий приоритет
  match community CUST_LOW
  set local-preference 50

route-map HANDLE_CLIENT_COMMUNITIES permit 30
  ! Все остальные маршруты пропускаем как есть
  ! (Важно оставить permit в конце, иначе всё остальное упадет!)

! Применяем к соседу на ВХОД (in)
router bgp 65000
  neighbor 10.0.0.1 route-map HANDLE_CLIENT_COMMUNITIES in
```

3. Игнорирование конкретных well-known community меток от других маршрутизаторов:

```Router
! Создаем список сообществ, которые хотим "вырезать"
ip community-list standard BAD_METRICS permit no-export
ip community-list standard BAD_METRICS permit no-advertise

route-map CLEAN_INCOMING permit 10
  match community BAD_METRICS
  ! Ключевая команда: удаляет ВСЕ коммьюнити у этого маршрута
  set community none
  ! Или, если хотите удалить только плохие и оставить хорошие:
  ! set community delete BAD_METRICS (поддерживается в новых версиях FRR)

route-map CLEAN_INCOMING permit 20
  ! Пропускаем остальные маршруты без изменений
  ! Но важно: если мы не чистили маршрут выше, он может сохранить no-export.
  ! Поэтому часто проще чистить всё и набирать заново, или использовать delete.

router bgp 65000
  neighbor 10.0.0.1 route-map CLEAN_INCOMING in
  ! Теперь, даже если клиент прислал no-export, у нас в таблице его нет.
  ! Мы можем смело анонсировать маршрут дальше.
```

!!TODO 
neighbor next-hop-self 
neighbor update-source
neighbor soft-reconfiguration inbound
no bgp ebgp-requires-policy
путь по умолчанию default information
