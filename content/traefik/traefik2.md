---
date: '2025-12-02T20:58:00+03:00'
draft: false
title: '2. Middleware и безопасность'
description: "Middleware и безопасность — защищаем ваши приложения"
author: 'DigitalChip'
tags: ["traefik","https","proxy","docker","middlewares"]
categories: ["traefik","devops"]
---

# Middleware и безопасность — защищаем ваши приложения

В этой статье продолжим разбирать работу реверс-прокси сервера Traefik 3 и перейдем к теме Middleware.

Вспомним нашу аналогию: Traefik — это [профессиональный швейцар для вашего Docker-окружения](/traefik/traefik1#введение-для-будущих-инженеров). Он встречает всех входящих "гостей" (запросы) и знает, в какой "офис" (контейнер) их проводить. Основам работы этого швейцара мы научились в [первой статье](/traefik/traefik1).

Но что, если к некоторым гостям нужен особый подход? У одних — нужно проверить пропуск, другим — ограничить скорость входа, третьим — выдать защитный шлем. Вот именно этим и занимаются Middleware - помощники нашего швейцара.

Middleware — это дополнительные модули, которые обрабатывают запрос по пути от клиента к вашему приложению. Они не знают о бизнес-логике вашего сервиса, их задача — выполнить чётко поставленную техническую задачу: проверить, добавить, изменить или ограничить.

В этой статье мы научим нашего швейцара работать с этими помощниками, чтобы сделать ваши приложения не только функциональными, но и безопасными.

## Часть 1. Middleware для безопасности

### 1.1 Базовая аутентификация ("Проверка пропуска")

Представьте, что у вас есть какой-то конфиденциальный проект. Вы не пускаете туда всех подряд, а просите гостей показать пропуск (логин и пароль) у входа. Middleware `basicAuth` делает то же самое.

#### Создание "пропуска"

Сначала сгенерируем пару логин-пароль. В терминале выполните:

```bash
echo $(htpasswd -nbB myuser mypassword) | sed -e s/\\$/\\$\\$/g
```

Вы получите строку вида `myuser:$$2y$$05$$9b67V9yuHMbBhrF7H8fqYuq0n6aXl.WH3lGqGrLWj6ikY7c4s44La`. Это зашифрованный пароль. Сохраните его.

> **Примечание**
> Если на вашей системе нет утилиты htpasswd, то нужно установить apache2-utils
>
>```bash
>sudo apt-get install apache2-utils
>```

#### Начинаем проверку

Вот готовый пример `docker-compose.yml` с включением мидлвара `basicAuth`:

```yaml
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

  whoami:
    image: traefik/whoami
    labels:
      - "traefik.enable=true"
      # Создаем роутер для сервиса whoami на entrypoint `web`
      - "traefik.http.routers.whoami.rule=Host(`whoami.localhost`)"
      - "traefik.http.routers.whoami.entrypoints=web"

      # Создаем Middleware "проверка пропуска"
      - "traefik.http.middlewares.my-auth.basicauth.users=myuser:$$2y$$05$$9b67V9yuHMbBhrF7H8fqYuq0n6aXl.WH3lGqGrLWj6ikY7c4s44La"

      # Связываем этот Middleware с нашим роутером
      - "traefik.http.routers.whoami.middlewares=my-auth"
    networks:
      - web

networks:
  web:
    name: web
```

Теперь по адресу `http://whoami.localhost` вас встретит окно авторизации. Без правильного логина и пароля приложение недоступно.

![BasicAuth](/img/traefik/basicAuth.png)

B заметье, что проверкой пропусков (авторизацией) занимается именно помощник швейцара (мидлвар), а не наше приложение.

### 1.2 IP Whitelist ("Список доверенных гостей")

Это ваш личный список "VIP-персон". Вход разрешён только с IP-адресов, внесённых в этот список. Всех остальных швейцар будет вежливо, но твёрдо разворачивать.

Пример для локальной сети и конкретных IP:

```yaml
# Фрагмент docker-compose для сервиса
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.my-secure-service.rule=Host(`admin.localhost`)"
  - "traefik.http.routers.my-secure-service.entrypoints=web"

  # Создаем Middleware "белый список"
  # Разрешаем только локальные адреса (192.168.1.0/24) и один конкретный внешний IP
  - "traefik.http.middlewares.my-whitelist.ipallowlist.sourcerange=192.168.1.0/24, 203.0.113.45"

  # Применяем middleware к роутеру
  - "traefik.http.routers.my-secure-service.middlewares=my-whitelist"
```

Если ваш IP не из указанный диапазонов, то получаем, сответственно:

![IP Whitelist](/img/traefik/whitelist.png)

> [!WARNING] **Предупреждение о безопасности**
> Не используйте этот метод как единственную защиту для действительно критичных систем, так как IP-адрес можно подделать (спуфить).

Внимательно отнеситесь к вышесказанному!

### 1.3 Rate Limiting ("Ограничение количества гостей")

Чтобы один какой-нидудь чересчур настойчивый гость не создавал толкучку и не мешал остальным, этот помощник швейцара считает, сколько запросов пришло с одного адреса. Если их больше допустимого — просит подождать.

Пример для защиты API:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.my-api.rule=Host(`api.localhost`) && PathPrefix(`/api/`)"
  - "traefik.http.routers.my-api.entrypoints=web"

  # Создаем Middleware "ограничитель по rate"
  # 100 запросов в среднем за 5 минут, "всплески" до 20 запросов за 10 секунд 
  - "traefik.http.middlewares.my-ratelimit.ratelimit.average=100"
  - "traefik.http.middlewares.my-ratelimit.ratelimit.period=5m"
  - "traefik.http.middlewares.my-ratelimit.ratelimit.burst=20"

  - "traefik.http.routers.my-api.middlewares=my-ratelimit"
```
Эти ограничения - для примера, это очень маленькие ограничения. В реальности - они намного больше.

### 1.4 Security Headers ("Дополнительные заголовки безопасности")

Когда наш швейцар провожает гостя к нужному офису, он даёт ему набор важных инструкций для браузера: "используй только защищённое соединение", "не открывайся в чужом фрейме", "не меняй тип содержимого". В мире веб-приложений эти инструкции называются Security Headers.

Полные определения middleware лучше выносить в отдельный YAML-файл, а в labels только ссылаться на него. Поэтому для сложных настроек безопасности в Traefik 3+ мы используем динамическую YAML-конфигурацию. Это как дать швейцару подробную инструкцию перед началом рабочего дня.

#### Структура проекта

Создайте в вашем проекте следующую структуру файлов:

```text
traefik-project/
├── docker-compose.yml
├── traefik.yml          # Статическая конфигурация Traefik
└── dynamic/
    └── security-headers.yml   # Динамическая конфигурация middleware
```

Теперь отредактируйте соответсвующие файлы.

#### 1. Файл traefik.yml (основная конфигурация)

```yaml
#traefik.yml
api:
  dashboard: true
  debug: true

entryPoints:
  web:
    address: ":80"

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
    network: web
  file:
    directory: "/etc/traefik/dynamic"
    watch: true

log:
  level: INFO

```

#### 2. Файл dynamic/security-headers.yml (наши "помощники")

Здесь мы определяем весь набор необходимых нам заголовков. Ниже приведен список заголовков для примера. Весь смысл сего действа, показать, что мы заранее определяем весь набор нужных нам заголовков в миделваре в отдельном файле, а потом в описании сервисов в `docker-compose.yml` в разделе `labels` мы просто применяем этот мидлвар.

```yaml
#docker-compose.yml
http:
  middlewares:
    security-headers:
      headers:
        sslRedirect: true           # Автоматическое перенаправление на HTTPS
        stsSeconds: 31536000        # HSTS: год использовать только HTTPS
        stsIncludeSubdomains: true  # Правило для всех поддоменов
        stsPreload: true            # Разрешить включение в предзагрузку браузеров
        forceSTSHeader: true        # Принудительно добавлять HSTS заголовок
        customRequestHeaders:
          X-Frame-Options: "SAMEORIGIN"        # Запретить встраивание в чужой <iframe>
          X-Content-Type-Options: "nosniff"    # Запретить браузеру угадывать тип контента
        customResponseHeaders:
          X-Permitted-Cross-Domain-Policies: "none"  # Блокировать кросс-доменные политики
          Referrer-Policy: "strict-origin-when-cross-origin" # Контроль передачи referrer

    strict-csp:
      headers:
        contentSecurityPolicy: "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'"
```

#### 3. Обновлённый docker-compose.yml

Здесь мы не описываем нужные нам заголовки в `labels` сервиса, а применяем заранее подготовленный и динамически подключенный набор.

```yaml
services:
  traefik:
    image: traefik:latest
    command:
      - "--configfile=/etc/traefik/traefik.yml"
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./traefik.yml:/etc/traefik/traefik.yml:ro
      - ./dynamic:/etc/traefik/dynamic:ro
    networks:
      - web

  secure-app:
    image: traefik/whoami
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.secure-app.rule=Host(`secure.localhost`)"
      - "traefik.http.routers.secure-app.entrypoints=web"
      # Подключаем middleware из файловой конфигурации
      - "traefik.http.routers.secure-app.middlewares=security-headers@file"
    networks:
      - web

networks:
  web:
    name: web
```

Суффикс `@file` при подключении мидлвара можно не ипользовать, если нет конфликта имен. Т.е., если вы определили мидлвар `security-headers` в секции `labels` какого-либо сервиса, и также применяется динамическое подключение мидлвара `security-headers` из приведенного выше файла, то тогда нужно указывать, какой именно мидлвар мы подключаем.

Что делают эти заголовки:

- `HSTS (Strict-Transport-Security):` - Приказывает браузеру всегда использовать HTTPS
- `X-Frame-Options:` - Защищает от атак "clickjacking" через iframe
- `X-Content-Type-Options:` - Предотвращает MIME-sniffing
- `CSP (Content Security Policy):` - Контролирует, откуда можно загружать ресурсы

#### Список доступных параметров headers

Traefik 3 поддерживает конкретный набор полей в `http.middlewares.<name>.headers` для безопасности — полный список из официальной документации.

Список всех доступных параметров headers:

- `stsSeconds:` - число секунд для HSTS (например, 31536000) — только в YAML!
- `stsIncludeSubdomains:` true/false — поддомены в HSTS
- `stsPreload:` - true/false — предзагрузка HSTS в браузеры
- `frameDeny:` -  true/false — запрет всех iframe ( вместо него лучше используйте customFrameOptionsValue)
- `customFrameOptionsValue:` - "SAMEORIGIN" | "DENY" — настройка X-Frame-Options
- `contentTypeNosniff:` - true/false — X-Content-Type-Options: nosniff
- `browserXssFilter:` - true/false — X-XSS-Protection
- `forceSTSHeader:` - true/false — Strict-Transport-Security
- `sslRedirect:` -  true/false — редирект на HTTPS (устарело, используйте middleware RedirectScheme)
- `referrerPolicy:` - "no-referrer" | "no-referrer-when-downgrade" и др. — Referrer-Policy

Дополнительно: `allowedHosts` (список доменов), `hostsProxyHeaders` (X-Forwarded-*), `customRequestHeaders/customResponseHeaders` для произвольных заголовков.​

Подробнее можно ознакомиться в официально документации Traefik.

#### Проверка работы

После запуска выполните в терминале:

```bash
curl -I http://secure.localhost
```

Вы увидите все настроенные заголовки безопасности в ответе сервера.

![Headers](/img/traefik/headers.png)

Такой подход позволяет централизованно управлять настройками безопасности и легко применять их к нескольким сервисам одновременно.

## Часть 2. Middleware для оптимизации

### 2.1 Compress ("Упаковка посылок")

Швейцар перед отправкой упаковывает большие и тяжёлые посылки (например, HTML, CSS, JS) в компактные архивы. Это ускоряет их доставку и экономит трафик.

Настройка сжатия:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.my-webapp.rule=Host(`webapp.localhost`)"
  - "traefik.http.routers.my-webapp.entrypoints=web"

  # Включаем Middleware сжатия
  - "traefik.http.middlewares.my-compress.compress=true"

  - "traefik.http.routers.my-webapp.middlewares=my-compress"
```

### 2.2 Retry ("Повторная попытка")

Если при попытке отвести гостя в офис, тот временно закрыт (сервис упал), швейцар не станет сразу говорить "не могу", а попробует провести его через 5 секунд. И так несколько раз.

Настройка повторных попыток:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.my-unstable-service.rule=Host(`service.localhost`)"
  - "traefik.http.routers.my-unstable-service.entrypoints=web"

  # Настраиваем Middleware "повторитель"
  # 5 попыток с интервалом в 500 миллисекунд
  - "traefik.http.middlewares.my-retry.retry.attempts=5"
  - "traefik.http.middlewares.my-retry.retry.initialinterval=500ms"

  - "traefik.http.routers.my-unstable-service.middlewares=my-retry"
```

### 2.3 Circuit Breaker ("Автоматический выключатель")

Если офис постоянно закрывается, швейцар перестаёт пытаться туда водить гостей, чтобы не тратить время и дать сервису "прийти в себя". Через некоторое время он осторожно попробует снова.

Настройка автоматического выключателя:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.my-fragile-service.rule=Host(`fragile.localhost`)"
  - "traefik.http.routers.my-fragile-service.entrypoints=web"

  # Настраиваем Middleware "автоматический выключатель"
  # Срабатывает, если > 50% запросов за 10 секунд завершаются ошибкой
  - "traefik.http.middlewares.my-circuitbreaker.circuitbreaker.expression=ResponseCodeRatio(500, 600) > 0.50"

  - "traefik.http.routers.my-fragile-service.middlewares=my-circuitbreaker"
```

## Часть 3. Практический пример — защищаем WordPress

Давайте соберём полученные знания вместе и защитим какой-нибудь сервис, к примеру - типичный блог на WordPress.

```yaml
services:
  traefik:
    image: traefik:latest
    container_name: traefik
    ports:
      # "Открываем дверь" для входящего HTTP-трафика
      - "80:80"
    volumes:
      # Даем доступ к Docker-сокету, чтобы Traefik мог "видеть" другие контейнеры
      - /var/run/docker.sock:/var/run/docker.sock
      # Монтируем файл статической конфигурации
      - ./traefik.yml:/etc/traefik/traefik.yml
    networks:
      - web

  wordpress:
    image: wordpress:latest
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: exampleuser
      WORDPRESS_DB_PASSWORD: examplepass
      WORDPRESS_DB_NAME: exampledb
    labels:
      - "traefik.enable=true"

      # Роутер для основного сайта
      - "traefik.http.routers.wp-site.rule=Host(`wp.localhost`)"
      - "traefik.http.routers.wp-site.entrypoints=web"

      # Роутер для панели администратора (админку защищаем сильнее)
      - "traefik.http.routers.wp-admin.rule=Host(`wp.localhost`) && PathPrefix(`/wp-admin`)"
      - "traefik.http.routers.wp-admin.entrypoints=web"

      # Middleware: Сжатие (для всего сайта)
      - "traefik.http.middlewares.compress.compress=true"

      # Middleware: Базовые заголовки безопасности (для всего сайта)
      # Здесь заголовки безопасности прописаны для примера! Этот мидлвар лучше выносить в файл 
      # динамичской конфигурации! Смотри пункт 1.4 или официальную документацию.
      - "traefik.http.middlewares.security-headers.headers.forceSTSHeader=true"
      - "traefik.http.middlewares.security-headers.headers.stsseconds=31536000"

      # Middleware: Ограничение запросов к админке (100/мин)
      - "traefik.http.middlewares.admin-ratelimit.ratelimit.average=100"
      - "traefik.http.middlewares.admin-ratelimit.ratelimit.period=1m"

      # Middleware: Базовая аутентификация для админки (создайте свой пароль!)
      - "traefik.http.middlewares.admin-auth.basicauth.users=admin:$$2y$$05$$YOUR_GENERATED_PASSWORD_HASH"

      # Цепочка Middleware для основного сайта: Сжатие -> Заголовки
      - "traefik.http.routers.wp-site.middlewares=compress, security-headers"

      # Цепочка Middleware для админки: Аутентификация -> Ограничение запросов -> Сжатие -> Заголовки
      - "traefik.http.routers.wp-admin.middlewares=admin-auth, admin-ratelimit, compress, security-headers"

    networks:
      - web

  db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: exampledb
      MYSQL_USER: exampleuser
      MYSQL_PASSWORD: examplepass
      MYSQL_RANDOM_ROOT_PASSWORD: '1'
    networks:
      - web

networks:
  web:
    name: web
```

Итак, что мы сделали в этом конфиге:

- Для всего сайта: включили сжатие и безопасные заголовки.
- Для админки (/wp-admin): добавили двойную защиту — базовую аутентификацию и ограничение по количеству запросов. Теперь даже если злоумышленник узнает пароль от WordPress, ему будет очень сложно провести атаку подбора.

Тестирование и отладка.

Инструменты для проверки: Браузер Developer Tools (вкладка Network) -  посмотрите на заголовки ответа (Response Headers). Вы должны увидеть Strict-Transport-Security, X-Frame-Options и другие, которые мы настроили.

Команда `curl` - идеальна для проверки.

`curl -I http://wp.localhost` — проверить заголовки.

`curl http://admin.localhost` — убедиться, что запрашивает пароль (вернет 401 Unauthorized).

`curl -u myuser:mypassword http://admin.localhost` — доступ с аутентификацией.

Чеклист безопасности нашего блога:

- Базовая аутентификация включена для критичных зон.
- Настроены безопасные заголовки (HSTS, X-Frame-Options).
- Включено сжатие для текстового контента.
- Настроены ограничения по частоте запросов для API/админки.

## Часть 4. Обзор middleware в Traefik — полный список помощников

Теперь, когда мы разобрались с основными middleware, давайте посмотрим на весь арсенал "помощников швейцара", доступный в Traefik 3. Это поможет нам понять, какие инструменты есть в нашем распоряжении для решения различных задач.

### Категории middleware

#### Middleware для безопасности

- `basicAuth` — базовая аутентификация (логин/пароль)
- `digestAuth` — Digest-аутентификация
- `forwardAuth` — внешняя аутентификация через другой сервис
- `ipAllowList` — белый список IP-адресов
- `ipWhiteList` — устаревший аналог ipAllowList
- `rateLimit` — ограничение частоты запросов
- `headers` — управление HTTP-заголовками безопасности
- `redirectScheme` — перенаправление между схемами (HTTP/HTTPS)
- `sslRedirect` — автоматическое перенаправление на HTTPS

#### Middleware для оптимизации

- `compress` — сжатие ответов (gzip, brotli)
- `retry` — повторная отправка запросов при ошибках
- `circuitBreake`r — автоматический выключатель при проблемах
- `buffering` — буферизация больших запросов/ответов
- `inFlightReq` — ограничение количества одновременных запросов

#### Middleware для модификации запросов

- `addPrefix` — добавление префикса к URL
- `stripPrefix` — удаление префикса из URL
- `stripPrefixRegex` — удаление префикса по регулярному выражению
- `replacePath` — замена пути запроса
- `replacePathRegex` — замена пути по регулярному выражению

#### Middleware для управления трафиком

- `chain` — объединение нескольких middleware в цепочку
- `errorPage` — кастомные страницы ошибок
- `passTLSClientCert` — передача клиентских TLS-сертификатов
- `plugins` — пользовательские middleware через плагины

### Примеры конфигурации для разных middleware

#### Digest Auth (в `dynamic/middleware.yml`)

```yaml
http:
  middlewares:
    digest-auth:
      digestAuth:
        users:
          - "user1:traefik:password1"
          - "user2:traefik:password2"
```

#### Forward Auth (внешняя аутентификация)

```yaml
http:
  middlewares:
    external-auth:
      forwardAuth:
        address: "https://auth-service.example.com/validate"
        trustForwardHeader: true
        authResponseHeaders:
          - "X-User-Id"
          - "X-User-Role"
```

#### Strip Prefix (удаление префикса URL)

```yaml
http:
  middlewares:
    api-strip:
      stripPrefix:
        prefixes:
          - "/api/v1"
    # Запрос /api/v1/users → будет отправлен в сервис как /users
```

### Replace Path (замена пути)

```yaml
http:
  middlewares:
    legacy-redirect:
      replacePath:
        path: "/new-path"
```

#### Error Pages (кастомные страницы ошибок)

```yaml
http:
  middlewares:
    custom-errors:
      errors:
        status:
          - "400-599"
        service: error-service
        query: "/{status}.html"
```

Как выбрать подходящий middleware? Вот общие рекомендации, хотя это дело - творческое....

- Для защиты API → rateLimit + ipAllowList + headers
- Для веб-приложений → compress + headers + basicAuth (для админки)
- Для микросервисов → circuitBreaker + retry + chain
- Для миграции старых систем → replacePath + stripPrefix

**Важные замечания:**

- Порядок имеет значение: Middleware выполняются в том порядке, в котором указаны в цепочке
- Производительность: Каждый middleware добавляет небольшую задержку
- Тестирование: Всегда проверяйте middleware в staging-окружении перед продакшеном

Теперь у вас есть полная картина того, какие "помощники" доступны вашему швейцару Traefik!

***Примечание:*** Полный актуальный список middleware всегда можно найти в [официальной документации Traefik](https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/overview/).

## Заключение

Поздравляем! Теперь наш швейцар Traefik обзавелся целым штатом умных помощников — Middleware. Вы научились защищать приложения от самых распространённых угроз и оптимизировать их работу.

Middleware — это мощные и гибкие инструменты для обработки запросов. Их можно комбинировать в цепочки для комплексной защиты.

Безопасность должна настраиваться с самого начала разработки.

В следующей статье мы превратим нашего швейцара в самого осведомлённого сотрудника: научим его собирать и показывать метрики и логи о том, что происходит в вашей системе. Вы будете видеть всё: кто заходит, куда направляется и нет ли подозрительной активности.

Если вы хотите потренироваться в настойке Middleware, то могу предложить выполнить вот такое задание:

1. Разверните пример с WordPress по инструкции выше.

2. Создайте свой Middleware, который добавляет кастомный заголовок X-Student-Name: YourName к вашему сервису.

3. Убедитесь с помощью curl -I, что заголовок действительно возвращается.

Удачи в изучении Traefik!
