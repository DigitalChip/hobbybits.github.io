---
date: '2025-12-02T20:59:00+03:00'
draft: false
title: '1. Что такое Traefik'
description: "Traefik: Ваш умный и внимательный швейцар для Docker-контейнеров"
author: 'DigitalChip'
tags: ["traefik","https","proxy","docker"]
categories: ["traefik","devops"]
---

# Traefik 3: Ваш умный и внимательный швейцар для Docker-контейнеров

Представьте, что у вас есть огромный офисный центр (ваш сервер), в котором работают десятки разных компаний (ваши Docker-контейнеры). Каждая компания предоставляет свой сервис: одна — веб-сайт, другая — API, третья — панель управления, четвертая - еще что-либо. Если бы каждый приходящий клиент сам искал, куда ему идти, это был бы хаос.

**Traefik** — это профессиональный швейцар для вашего офисного центра. Его работа:

- Встречать входящих гостей (HTTP-запросы).
- Смотреть в их приглашение (Host, Path, заголовки).
- Точно знать, в какую компанию (контейнер) гостя нужно проводить.
- Быть невероятно быстрым и настраиваться прямо на ходу, без перерывов на перестройку всего здания.

В этой статье мы разберем, как нанять этого швейцара (установить Traefik), как обучить его правилам (настроить с помощью меток и файлов динамической конфигурации) и как он понимает связь между этими правилами.

## Часть 1. Устанавливаем швейцара на пост (Docker Compose для Traefik)

Давайте сначала запустим сам Traefik. Мы будем использовать Docker Compose, так как это уже стандарт для управления несколькими контейнерами.

В Traefik 3 статическую конфигурацию (базовые настройки самого Traefik) можно задавать через YAML-файл, а не через аргументы командной строки, как это было в предыдущих версиях. Это более современный и гибкий подход.

Давайте создадим структуру проекта:

```txt
traefik-stack/
├── traefik.yml           # Статическая конфигурация Traefik
└── docker-compose.yml    # Docker Compose для запуска Traefik
```

### 1. Создаем статическую конфигурацию Traefik (traefik.yml)

```yaml
# traefik.yml
api:
  dashboard: true        # Включаем панель управления (пока для отладки)
  insecure: true         # Разрешаем доступ к API без аутентификации (ТОЛЬКО ДЛЯ ТЕСТА!)

entryPoints:
  web:
    address: ":80"       # Создаем точку входа для HTTP на порту 80

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"  # Как подключаться к Docker
    exposedByDefault: false  # Игнорировать контейнеры без явных меток traefik
```

### 2. Создаем Docker Compose файл (docker-compose.yml)

```yaml
# docker-compose.yml
services:
  traefik:
    image: traefik:3.6
    container_name: traefik
    ports:
      # "Открываем дверь" для входящего HTTP-трафика
      - "80:80"
      # Дверь для Dashboard (обычно не открывают наружу, но это для тестирования)
      - "8080:8080"
    volumes:
      # Даем доступ к Docker-сокету, чтобы Traefik мог "видеть" другие контейнеры
      - /var/run/docker.sock:/var/run/docker.sock
      # Монтируем файл статической конфигурации
      - ./traefik.yml:/etc/traefik/traefik.yml
    networks:
      - web

# Создаем сеть, чтобы контейнеры могли общаться друг с другом
networks:
  web:
    name: web
```

### Что здесь происходит?

`traefik.yml` — это "должностная инструкция" (статическая конфигурация) для швейцара. Мы говорим ему:

- Где и как принимать запросы (entryPoints)
- Где брать информацию о сервисах (providers.docker)
- Включить панель управления для мониторинга

`volumes` — самый важный момент! Мы монтируем:

- Docker-сокет — дает Traefik волшебную способность "видеть" события Docker
- Файл конфигурации — передаем "должностную инструкцию"

`networks` — помещаем Traefik в сеть `web`. Все сервисы, которыми он будет управлять, должны быть в этой же сети.

Запустите командой `docker-compose up -d`. Теперь ваш "швейцар" на посту. Он пока никого не знает, но уже слушает на порту `80`. Панель управления доступна по адресу `http://localhost:8080`.

## Часть 2. Знакомим швейцара с первой компанией (Метки Traefik)

Теперь давайте запустим какой-нибудь простой веб-сервис и научим Traefik его "знать". Мы сделаем это с помощью меток (labels). Whoami - специальный небольшой сервис, предназанченный для тестирования, который просто выводит заголовки запроса при обращении к нему.

Создайте второй файл `docker-compose.apps.yml` для ваших приложений:

```yaml
services:
  whoami:
    image: "traefik/whoami"
    container_name: "simple-service"
    labels:
      # Включаем контейнер для Traefik
      - "traefik.enable=true"
      # Создаем Роутер с именем "whoami-router"
      - "traefik.http.routers.whoami-router.rule=Host(`whoami.localhost`)"
      # Говорим, что этот роутер использует входную точку "web"
      - "traefik.http.routers.whoami-router.entrypoints=web"
      # Создаем Сервис с именем "whoami-service"
      - "traefik.http.services.whoami-service.loadbalancer.server.port=80"
    networks:
      - web

networks:
  web:
    external: true
    name: web # Имя сети, которую создал первый compose-файл
```

### Как это работает? Логика связей

Метки Traefik имеют четкую иерархическую структуру. Давайте разберем ее на примере:

```text
traefik.enable=true
```

Это базовый выключатель. "Швейцар, обрати внимание на эту компанию (контейнер)".

```text
traefik.http.routers.whoami-router.rule=Host(whoami.localhost)
```

Мы создаем Роутер (Router) с именем `whoami-router`.

Роутер — это правило для швейцара. "Если гость пришел с приглашением, где написано Host: whoami.localhost (то есть он ищет именно этот адрес), то этот роутер сработает".

Имя роутера (`whoami-router`) вы придумываете сами. Оно нужно для связывания сущностей внутри Traefik.

```text
traefik.http.routers.whoami-router.entrypoints=web
```

Мы связываем роутер `whoami-router` с входной точкой (entrypoint) `web`.

Входная точка — это "дверь", в которую стучится гость. Мы создали дверь `web` на 80-м порту в настройках самого Traefik (файле статической конфигурации traefik.yml). Этой меткой мы говорим: "Правило `whoami-router` действует для гостей, которые пришли через дверь `web`".

```text
traefik.http.services.whoami-service.loadbalancer.server.port=80
```

Мы создаем Сервис (Service) с именем `whoami-service`.

Сервис — это и есть сама "компания". Он знает, на какой именно внутренний порт контейнера нужно направить запрос.

Но погодите! Мы нигде явно не связали роутер `whoami-router` с сервисом `whoami-service`. Почему это работает?

### Волшебство автоматических связей

Traefik старается быть умным. Если у вас есть ровно один роутер и ровно один сервис в одном контейнере, и их имена отличаются только суффиксом (`-router` и `-service`), он автоматически свяжет их невидимой нитью.

В нашем случае:

- Роутер: `whoami-router`
- Сервис: `whoami-service`

Traefik видит общую часть `whoami` и понимает, что они принадлежат друг другу. Это удобное соглашение, но явное указание всегда лучше.

## Часть 3. Явное связывание роутеров и сервисов — убираем магию

### Проблема "магии" автоматического связывания

В Traefik есть возможность автоматического связывания роутеров и сервисов, если они находятся в одном контейнере и имеют похожие имена. Например, если у вас есть роутер `my-app` и сервис `my-app` (без суффикса), Traefik может автоматически их связать.

Но представьте: швейцар должен сам догадываться о связях между правилами и компаниями. Это работает, пока всё просто, но в большом офисе с десятками компаний такая "магия" приводит к ошибкам и неразберихе.

Явное связывание — это как дать швейцару четкую инструкцию:

"Роутер `whoami` должен направлять гостей именно в сервис `whoami-service`."

Давайте расширим наш пример, чтобы показать, как это делать правильно и даже в более сложных сценариях.

**Пример**: один контейнер, два роутера

Представим, что наше приложение `whoami` должно быть доступно по двум разным доменам: `whoami.localhost` и `api.whoami.localhost`. Мы можем создать два роутера, но оба они будут указывать на один и тот же сервис.

```yaml
# docker-compose.apps.yml
services:
  whoami:
    image: "traefik/whoami"
    container_name: "simple-service"
    labels:
      - "traefik.enable=true"

      # Первый роутер для основного домена
      - "traefik.http.routers.whoami-main.rule=Host(`whoami.localhost`)"
      - "traefik.http.routers.whoami-main.entrypoints=web"
      # Задаем сервис, который будет обрабатывать этот роут
      - "traefik.http.routers.whoami-main.service=whoami-service"

      # Второй роутер для API-поддомена
      - "traefik.http.routers.whoami-api.rule=Host(`api.whoami.localhost`)"
      - "traefik.http.routers.whoami-api.entrypoints=web"
      # Задаем сервис, который будет обрабатывать этот роут
      - "traefik.http.routers.whoami-api.service=whoami-service"

      # Общий сервис для обоих роутеров
      - "traefik.http.services.whoami-service.loadbalancer.server.port=80"
    networks:
      - web

networks:
  web:
    external: true
    name: web # Имя сети, которую создал первый compose-файл
```

Теперь мы имеем:

- Два роутера: `whoami-main` и `whoami-api`
- Один сервис: `whoami-service`
- Четкие и явные связи: каждый роутер явно указал, какой сервис использовать

Преимущества явного связывания:

- Прозрачность: сразу видно, какой роутер куда ведет
- Гибкость: можно легко перенаправлять трафик с одного роутера на другой сервис
- Надежность: нет неожиданного поведения при изменении конфигурации

## Часть 4. Посерединники (Middleware) — помощники швейцара

Иногда швейцару нужны помощники, которые проверяют гостей перед тем, как пустить их в компанию. В Traefik такие помощники называются Middleware.

### Что такое Middleware?

Middleware — это промежуточные обработчики, которые могут:

- Проверять аутентификацию
- Добавлять заголовки
- Сжимать ответы
- Перенаправлять с HTTP на HTTPS
- Ограничивать количество запросов

И многое другое...

**Пример**: Базовая аутентификация (basicAuth) для админки

Допустим, у нас есть панель управления, которую нужно защитить паролем.

```yaml
# docker-compose.apps.yml
services:
  admin-panel:
    image: "some-admin-image:latest"
    container_name: "admin-service"
    labels:
      - "traefik.enable=true"
      
      # Роутер для админки
      - "traefik.http.routers.admin.rule=Host(`admin.localhost`)"
      - "traefik.http.routers.admin.entrypoints=web"
      - "traefik.http.routers.admin.service=admin-service"
      
      # Middleware для базовой аутентификации (задется как user:pass, см. ниже)
      - "traefik.http.middlewares.admin-auth.basicauth.users=admin:$$apr1$$9Cv/OMGj$$ZomWQzuQbL.3TRCS81a1g/"
      
      # Применяем middleware к роутеру
      - "traefik.http.routers.admin.middlewares=admin-auth"
      
      # Сервис (неявное связывание, см. выше)
      - "traefik.http.services.admin-service.loadbalancer.server.port=3000"
    networks:
      - web

networks:
  web:
    external: true
    name: web
```

>Как создать хеш пароля:
>
>```bash
># Установите apache2-utils если нет утилиты htpasswd
>sudo apt-get install apache2-utils
>
># Создаем хеш пароля (измените admin и password на свои)
>echo $(htpasswd -nb admin password) | sed -e s/\\$/\\$\\$/g
>
># Результат будет выглядеть так:
># admin:$$apr1$$9Cv/OMGj$$ZomWQzuQbL.3TRCS81a1g/
>
>```

#### Пример 2: Перенаправление HTTP на HTTPS

Давайте добавим в нашу конфигурацию Traefik точку входа для HTTPS и настроим перенаправление.

Сначала обновим статическую конфигурацию Traefik (`traefik.yml`):

```yaml
# traefik.yml
api:
  dashboard: true
  insecure: true  # В продакшене нужно отключить!

entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"      # Добавляем точку входа для HTTPS

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
```

Теперь добавим middleware для редиректа в наше приложение:

```yaml
# docker-compose.apps.yml
services:
  whoami:
    image: "traefik/whoami"
    container_name: "simple-service"
    labels:
      - "traefik.enable=true"
      
      # Middleware для редиректа HTTP -> HTTPS
      - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"
      
      # Роутер для HTTP (никуда не приведет, только редирект)
      - "traefik.http.routers.whoami-http.rule=Host(`whoami.localhost`)"
      - "traefik.http.routers.whoami-http.entrypoints=web"
      - "traefik.http.routers.whoami-http.middlewares=redirect-to-https"
      
      # Роутер для HTTPS (основной)
      - "traefik.http.routers.whoami-https.rule=Host(`whoami.localhost`)"
      - "traefik.http.routers.whoami-https.entrypoints=websecure"
      - "traefik.http.routers.whoami-https.service=whoami-service"
      
      # Сервис (явное связывание, см. выше)
      - "traefik.http.services.whoami-service.loadbalancer.server.port=80"
    networks:
      - web
```

Разберем, что здесь происходит.

1. Запрос приходит на HTTP (`whoami-http` роутер) (http://whoami.localhost)
2. Middleware `redirect-to-https` перенаправляет его на HTTPS
3. Перенаправленный запрос приходит на HTTPS (`whoami-https` роутер) (https://whoami.localhost)
4. Traefik направляет запрос в сервис `whoami-service`

## Часть 5. Работа с HTTPS (Let's Encrypt)

Протокол HTTPS сегодня стал обязательным стандартом для любого веб-сайта или приложения в сети Интернет. Этот протокол обеспечивает шифрование данных между пользователем и сервером, защищая информацию от перехвата и подделки. Без HTTPS современные браузеры помечают сайты как «небезопасные», что снижает доверие посетителей и негативно влияет на поисковую выдачу. Кроме того, многие веб-сервисы и API требуют обязательного использования защищённого соединения, делая HTTPS неотъемлемой частью современного интернета.

Traefik — поддерживает автоматическое получение и обновление SSL-сертификатов через Let’s Encrypt — бесплатный центр сертификации. Благодаря встроенной интеграции с протоколом ACME, Traefik самостоятельно запрашивает, устанавливает и продлевает сертификаты, избавляя администраторов от рутинных задач и гарантируя, что все соединения остаются защищёнными без дополнительных усилий. Это одна из причин, благодаря которым Traefik полюбился многим.

### Настройка Let's Encrypt

Обновим нашу статическую конфигурацию Traefik (`traefik.yml`):

```yaml
# traefik.yml
api:
  dashboard: true
  insecure: true  # В продакшене нужно отключить!

entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false

certificatesResolvers:
  letsencrypt:
    acme:
      email: your-email@example.com  # Ваш email для уведомлений
      storage: /letsencrypt/acme.json
      httpChallenge:
        entryPoint: web  # Используем HTTP-челлендж для получения сертификатов
```

Теперь обновим Docker Compose для Traefik, чтобы добавить том для хранения сертификатов:

```yaml
# docker-compose.yml (обновленный)
services:
  traefik:
    image: traefik:3.6
    container_name: traefik
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080" # В продакшене нужно отключить!
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./traefik.yml:/etc/traefik/traefik.yml
      - ./letsencrypt:/letsencrypt  # Том для хранения сертификатов
    networks:
      - traefik-web

networks:
  web:
    external: false
```

Теперь в сервисах мы можем указать, что нужно использовать HTTPS и резолвер сертификатов:

```yaml
# docker-compose.apps.yml (обновленная конфигурация whoami)
whoami:
  image: "traefik/whoami"
  container_name: "simple-service"
  labels:
    - "traefik.enable=true"
    
    # HTTP роутер с перенаправлением на HTTPS
    - "traefik.http.routers.whoami-http.rule=Host(`whoami.localhost`)"
    - "traefik.http.routers.whoami-http.entrypoints=web"
    - "traefik.http.routers.whoami-http.middlewares=redirect-to-https"
    
    # HTTPS роутер
    - "traefik.http.routers.whoami-https.rule=Host(`whoami.localhost`)"
    - "traefik.http.routers.whoami-https.entrypoints=websecure"
    - "traefik.http.routers.whoami-https.service=whoami-service"
    - "traefik.http.routers.whoami-https.tls.certresolver=letsencrypt"  # Используем Let's Encrypt!
    
    # Сервис
    - "traefik.http.services.whoami-service.loadbalancer.server.port=80"
  
  networks:
    - web
```

Разбиаемся, что здесь происходит:

1. При первом обращении к `https://whoami.localhost` Traefik автоматически (сам, без нашей помощи, без каких-либо действий с нашей стороны) получит SSL-сертификат от Let's Encrypt
2. Сертификат будет сохранен в `./letsencrypt/acme.json`
3. Traefik будет автоматически обновлять сертификат перед истечением срока действия

## Часть 6. Best Practices для продакшена

### 1. Защита Dashboard Traefik

Dashboard Traefik — это мощный инструмент, который показывает всю вашу конфигурацию, активные соединения, метрики и многое другое. В неправильных руках эта информация может быть использована для атак на вашу инфраструктуру. Поэтому никогда не оставляйте Dashboard открытым без аутентификации!

Давайте разберем примерную безопасную конфигурацию по шагам.

#### Шаг 1. Отключаем небезопасный доступ к Dashboard

В статической конфигурации Traefik (`traefik.yml`) мы отключаем прямой доступ к панели:

```yaml
# traefik.yml
api:
  dashboard: true
  insecure: false  # ← ВАЖНО: отключаем публичный доступ без шифрования

entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"

...

```

**Что изменилось**:

`insecure: false` — теперь Dashboard недоступен по `http://localhost:8080`

Весь доступ к Dashboard теперь будет проходить через обычные роутеры с аутентификацией, которые мы сейчас добавим.

#### Шаг 2. Настраиваем безопасный доступ через Docker метки

Теперь добавим в Docker Compose файл Traefik метки для создания защищенного роутера для нашей панели:

```yaml
# docker-compose.yml
services:
  traefik:
    image: traefik:3.6
    container_name: traefik
    ports:
      - "80:80"
      - "443:443" # Добавляем порт HTTPS
      # УБИРАЕМ порт 8080, так как Dashboard теперь будет доступен через HTTPS
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./traefik.yml:/etc/traefik/traefik.yml
      - ./letsencrypt:/letsencrypt
    labels:
      # Включаем Traefik для самого Traefik контейнера
      - "traefik.enable=true"
      
      # Роутер для Dashboard
      - "traefik.http.routers.traefik-dashboard.rule=Host(`traefik.yourdomain.com`)"
      - "traefik.http.routers.traefik-dashboard.entrypoints=websecure"
      - "traefik.http.routers.traefik-dashboard.tls.certresolver=letsencrypt"
      
      # Так как Dashboard от Traefik не имеет какой-либо аутентификации, то добавим
      # Middleware для базовой аутентификации (задется как user:pass, см. выше)
      - "traefik.http.middlewares.traefik-auth.basicauth.users=admin:$$apr1$$9Cv/OMGj$$ZomWQzuQbL.3TRCS81a1g/"
      
      # Применяем аутентификацию к роутеру
      - "traefik.http.routers.traefik-dashboard.middlewares=traefik-auth"
      
      # Специальный сервис для Dashboard - api@internal
      - "traefik.http.routers.traefik-dashboard.service=api@internal"
    networks:
      - web
```

`api@internal` — это специальный встроенный сервис, который представляет сам Traefik и его Dashboard. Не требует отдельного определения сервиса с портом

#### Шаг 3: Дополнительные меры безопасности (рекомендуется)

Давайте также ограничим доступ к панели с определенных IP-адресов, добавим whitelist по IP (только для внутренней сети):

```yaml
- "traefik.http.middlewares.traefik-whitelist.ipwhitelist.sourcerange=192.168.1.0/24, 10.0.0.0/8"
- "traefik.http.routers.traefik-dashboard.middlewares=traefik-auth,traefik-whitelist"
```

Все, после этого доступ к панели будет только:

- С определенных IP-адресов и подсетей (которые мы указали выше)
- После ввода логина и пароля
- Только по HTTPS

Обратите внимание, мидлвары для роутера перечисляются через запятую. Если применить присвоение каждого мидлвара роутеру отдельными строками с метками, то примениться только последний опредленный мидлвар!

### 2. Разделение сетей

В Docker сети позволяют изолировать контейнеры друг от друга. Traefik, как швейцар, может находиться в нескольких сетях и маршрутизировать трафик между ними. Сотрудники комапний в нашем офисном центре могут перемещаться между собой не привлекая для этого нашего швейцара, т.е. работать в отдельной от Traefik сети, но только внутри центра (сервера). Наружу они выйти не могут.

Зачем разделять сети?

Представьте: в офисе есть зона для посетителей (frontend) и закрытые помещения для сотрудников (backend). Вы же не хотите, чтобы случайный гость мог зайти в серверную комнату!

Базовая настройка сетей:

```yaml
# docker-compose.yml
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # Важно: запрещаем выход в интернет
```

Как распределяем сервисы:

```yaml
services:
  traefik:
    networks:
      - frontend  # Traefik общается с внешним миром

  nginx:
    networks:
      - frontend  # Принимает запросы от Traefik
      - backend   # Общается с внутренними сервисами

  database:
    networks:
      - backend   # Только внутри, невидим извне
    labels:
      - "traefik.enable=false"  # Явно отключаем!
```

Что это дает:

- Безопасность: Базы данных и внутренние сервисы недоступны из интернета
- Контроль: Четкое разделение того, что можно "показывать снаружи"
- Производительность: Меньше ненужного сетевого трафика

Продолжая наши аналогии:

- frontend — это приемная офиса (доступна всем)
- backend — это рабочие кабинеты (только для сотрудников)
- Traefik — это секретарь, который знает, кого куда пускать

Такой подход значительно усложняет жизнь злоумышленникам!

### 3. Логирование и мониторинг

#### Настройка параметров логирования

Зачем это нужно?

Логи Traefik — это как журнал посещений у швейцара. Он записывает кто, куда и когда заходил, и не пускал ли он кого-то по ошибке. Логирование помогает быстро находить проблемы: почему сервис недоступен, кто делает много запросов, какие ошибки возникают у пользователей.

Базовая настройка логирования в `traefik.yml`:

```yaml
# traefik.yml
log:
  level: INFO        # Уровень детализации: DEBUG, INFO, WARN, ERROR
  format: common     # Формат: common (читаемый) или json (для машин)
  
# Дополнительные настройки для HTTP-запросов (журнал запросов)
accessLog:
  filePath: "/var/log/traefik/access.log"
  format: common  # Или json

  # Опционально: фильтрация запросов по коду ответа
  filters:
    statusCodes:
      - "200-299"
      - "300-399"
      - "400-499"
      - "500-599"

  # Опционально: буферизация записи логов
  bufferingSize: 100
```

Уровни логирования простыми словами:

- `DEBUG` - Записывается абсолютно всё происходящее, даже мельчайшие детали (подходит для глубокого анализа проблемы)
- `INFO` - Фиксируются важные события системы, вроде действий пользователей (полезно для ежедневной эксплуатации).
- `WARN` - Отмечаются потенциально опасные ситуации, которые требуют внимания (если что-то пошло не так, но система продолжает работать).
- `ERROR` - Регистрируются критичные сбои, приводящие к ошибкам и остановке работы (самый важный уровень, показывает серьёзные неполадки).

Как смотреть логи в Docker:

```bash
# Посмотреть логи в реальном времени
docker logs -f traefik

# Посмотреть только ошибки
docker logs traefik | grep ERROR

# Посмотреть логи за последний час
docker logs --since 1h traefik
```

#### Мониторинг через Dashboard

Dashboard Traefik — это как "пульт управления" швейцара. Там видно:

- Какие роутеры активны (правила)
- Какие сервисы работают (компании)
- Сколько запросов обработано (гостей проведено)
- Какие ошибки происходят (проблемы)

>**Практический совет:**
>
> В продакшене используйте level: `INFO` и `JSON` формат — так логи будет проще анализировать специальными инструментами (ELK Stack, Grafana Loki).

#### Интеграция с внешними системами мониторинга

Traefik может отправлять метрики в системы мониторинга, такие как Prometheus. Для этого нужно настроить соответствующий экспорт в статической конфигурации.

Пример настройки для Prometheus:

```yaml
# traefik.yml
metrics:
  prometheus:
    entryPoint: web  # Указываем точку входа, на которой будет доступен /metrics
    addRoutersLabels: true  # Добавлять метки роутеров в метрики
```

Затем в Prometheus нужно добавить job для сбора метрик с Traefik.

### 4. Здоровье сервисов

**Зачем это нужно?**

Health Checks (проверки здоровья) — это как регулярный медосмотр для ваших сервисов. Traefik может автоматически прекращать направлять трафик в нездоровые контейнеры, пока они не восстановятся.

**Как это работает?**

Вы добавляете в описание сервиса инструкцию, как проверять его здоровье. Docker будет выполнять эту проверку и сообщать Traefik, доступен ли сервис.

Как это работает в Docker Compose:

```yaml
services:
  web-app:
    image: your-web-app:latest
    container_name: web-service
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.web.rule=Host(`app.localhost`)"
      - "traefik.http.routers.web.service=web-service"
      - "traefik.http.services.web-service.loadbalancer.server.port=80"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80/health"]  # Команда для проверки
      interval: 30s        # Проверять каждые 30 секунд
      timeout: 10s         # Ждать ответа не более 10 секунд
      retries: 3           # Считать сервис упавшим после 3 неудачных попыток
      start_period: 40s    # Дать сервису 40 секунд на запуск перед первой проверкой
    networks:
      - web
```

Что здесь происходит:

- Docker каждые 30 секунд выполняет команду проверки
- Если 3 проверки подряд проваливаются — контейнер помечается как unhealthy
- Traefik видит это и перестает направлять трафик в "больной" контейнер
- Когда проверки снова начинают проходить — трафик возобновляется

Примеры проверок для разных сервисов:

```yaml
# Для базы данных PostgreSQL
test: ["CMD-SHELL", "pg_isready -U postgres"]

# Для Redis
test: ["CMD", "redis-cli", "ping"]

# Для Node.js приложения
test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000/health"]
```

Что получили в итоге:

- Автоматическое восстановление после сбоев
- Пользователи не видят ошибок от неработающих контейнеров
- Traefik сам знает, куда можно направлять трафик

### 5. Ограничение ресурсов

Данный пункт вытекает из следующей возможной проблемы - один "прожорливый" контейнер может "съесть" все ресурсы сервера и "уморить голодом" остальные сервисы.

Для решения этой проблемы рекомендуется ограничивать ресурсы для каждого контейнера:

```yaml
services:
  web-app:
    image: your-web-app:latest
    container_name: web-service
    deploy:
      resources:
        limits:
          memory: 512M    # Не более 512MB RAM
          cpus: '0.5'     # Не более половины ядра CPU
        reservations:
          memory: 256M    # Гарантированные 256MB RAM
          cpus: '0.1'     # Гарантированные 10% ядра CPU
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.web.rule=Host(`app.localhost`)"
    networks:
      - web
```

Что это значит на практике:

- `Limits` — "потолок": контейнер не сможет использовать больше
- `Reservations` — "гарантия": контейнеру всегда достанется хотя бы столько

Примеры разумных ограничений:

```yaml
# Для легкого веб-сервера (nginx)
memory: 256M
cpus: '0.2'

# Для базы данных (PostgreSQL)  
memory: 1G
cpus: '1.0'

# Для кэша (Redis)
memory: 512M
cpus: '0.3'
```

Что это нам дас в конечном итоге:

- Ни один сервис не может "задушить" всю систему
- Предсказуемое поведение под нагрузкой
- Легче планировать масштабирование
- Защита от DDoS-атак и утечек памяти

Опять же, продолжая нашу тему с аналогиями, то эти заданные ограничения - это как квоты в коммунальной квартире: каждый получает свою долю, но не может отобрать у соседей!

## Заключение: Traefik 3 — ваш надежный швейцар в мире контейнеров

Реверс-прокси сервер Traefik 3 — это мощный и гибкий инструмент, который делает маршрутизацию в Docker-окружении простой и эффективной. За время чтения мы прошли путь от базовых понятий до продвинутых практик для продакшена. Давайте подведем краткие итоги.

Что мы освоили:

- Основы работы: Теперь мы понимаем, как Traefik работает как "швейцар" для наших Docker-контейнеров, автоматически обнаруживая сервисы и маршрутизируя трафик.
- Современная конфигурация: Мы научились настраивать Traefik 3 через статические YAML-файлы вместо устаревших аргументов командной строки.
- Явные связи: Мы узнали важность явного связывания роутеров и сервисов, что делает конфигурацию предсказуемой и надежной.
- Безопасность: Мы разобрали, как защитить Dashboard, настроить HTTPS с Let's Encrypt и разделить сети для изоляции сервисов.
- Надежность: Мы познакомились с health checks и ограничением ресурсов — ключевыми элементами стабильной работы в продакшене.

Traefik 3 — это не просто инструмент, а философия подхода к маршрутизации в микросервисной архитектуре. Как хороший швейцар, он работает незаметно, но эффективно, обеспечивая порядок и надежность в вашей инфраструктуре.
