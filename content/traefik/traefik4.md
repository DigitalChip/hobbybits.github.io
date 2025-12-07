---
date: '2025-12-02T20:56:46+03:00'
draft: false
title: '4. Мониторинг и метрики'
description: "Traefik 3: Мониторинг, метрики и производительность — видим всё, контролируем всё"
author: 'DigitalChip'
tags: ["traefik","https","proxy","docker","promrtheus","grafana"]
categories: ["traefik","devops"]
---

# Traefik 3: Мониторинг, метрики и производительность — видим всё, контролируем всё

Итак, наш швейцар Traefik трудится не покладая рук, пропуская гостей (запросы) в офисный центр (ваше приложение). Он эффективен, вежлив, но... мы не знаем, что он делает. Сколько гостей пришло? Кто задерживается у него надолго? Не стоит ли ли в очереди? Без мониторинга мы слепы. Сегодня мы установим для нашего швейцара полноценную систему наблюдения, с дашбордами, метриками и алертами.

В предыдущих статьях мы наняли швейцара ([Traefik](/traefik/traefik1)) и дали ему помощников ([Middleware](/traefik/traefik2)) для сложных задач. Теперь пришло время наблюдать за его работой. Мониторинг — это и есть наша система наблюдения, а метрики — это статистика его работы: количество гостей, время обработки и т.д.

Почему это важно? Без этого мы узнаём о проблемах только тогда, когда наши пользователи начинают жаловаться, что "всё упало". Мы же хотим быть проактивными и предвидеть проблемы ещё до их возникновения!

## Часть 1. Настройка мониторинга в Traefik

### 1.1 Метрики Prometheus — "Статистика работы швейцара"

Prometheus выполняет роль нашего основного инструмента для сбора статистики. Он работает непрерывно, отслеживая ключевые показатели работы Traefik. Среди собираемых данных: общее количество запросов, количество ошибок, а также время задержки при обработке. Эти метрики помогают анализировать производительность системы и оперативно реагировать на возможные проблемы.

Давайте заставим Traefik отдавать эти метрики. Для этого дополним статическую конфигурацию в `traefik.yml`.

```yaml
#traefik.yml

...

# Точка входа для метрик Prometheus
metrics:
  prometheus:
    entryPoint: metrics # Создадим специальный "вход" для статистики
    addRoutersLabels: true # Подписывать метрики именами наших роутеров
    addServicesLabels: true # И именами сервисов

# Определяем точки входа (двери для гостей)
entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: ":443"
  metrics:
    address: ":8082" # Специальная дверь, куда будет заходить Prometheus за статистикой

...

```

### 1.2 Access Logs — "Журнал учета всех гостей"

Это детальный журнал, куда швейцар записывает каждого гостя: кто пришёл, когда, куда направился и сколько времени это заняло.

Добавим в динамическую конфигурацию (dynamic.yml):

```yaml
#dynamic.yml

http:
  middlewares:
    # Здесь будут наши промежуточные ПО (следующая статья)
    
  routers:
    # Здесь будут наши роутеры

  services:
    # Здесь будут наши сервисы

# Включаем детальные логи для всего Traefik
accessLog: 
  filePath: "/var/log/traefik/access.log" # Куда писать логи
  format: json # В формате JSON
  bufferingSize: 100 # Буферизация для производительности
```

### 1.3 Dashboard Traefik — "Пульт управления швейцара"

Дашборд — это монитор, на котором мы в реальном времени видим, что происходит. Но будьте осторожны! Оставлять его открытым — как раздать ключи от пульта управления всем подряд.

Настройка безопасного доступа к Дашборду через динамическую конфигурацию:

Добавим в dynamic.yml:

```yaml
http:
  routers:
    # Создаем роутер специально для дашборда
    traefik-dashboard:
      rule: "Host(`traefik.mydomain.com`)" # Доступ по домену
      entryPoints:
        - websecure
      service: api@internal # Волшебный сервис - сам Traefik
      middlewares:
        - auth-middleware # Защищаем аутентификацией!
      tls: {} # Обязательно с HTTPS!

  middlewares:
    # Middleware базовой аутентификации
    auth-middleware:
      basicAuth:
        users:
          - "admin:$2y$10$92SHDxdbdEZKSDzT/N..e.OwQaX.8LxBBFkqP8sQlMQMbA32RoLb6" # пароль: mypassword
```

## Часть 2: Визуализация в Grafana

### 2.1 Запуск стека мониторинга

Теперь нам нужен красивый интерфейс для нашей статистики. Запустим Prometheus (сбор статистики) и Grafana (красивые графики) через Docker Compose.

```yaml
#docker-compose.monitoring.yml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    networks:
      - traefik-net
    labels:
      - "traefik.enable=true"
      # Роутер для Prometheus UI
      - "traefik.http.routers.prometheus.rule=Host(`prometheus.mydomain.com`)"
      - "traefik.http.routers.prometheus.entrypoints=websecure"
      - "traefik.http.routers.prometheus.tls=true"

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=mygrafanaPassword123
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources.yml:/etc/grafana/provisioning/datasources/datasources.yml
    networks:
      - traefik-net
    labels:
      - "traefik.enable=true"
      # Роутер для Grafana
      - "traefik.http.routers.grafana.rule=Host(`grafana.mydomain.com`)"
      - "traefik.http.routers.grafana.entrypoints=websecure"
      - "traefik.http.routers.grafana.tls=true"

volumes:
  prometheus_data:
  grafana_data:

networks:
  traefik-net:
    external: true # Используем внешнюю сеть, к которой подключен Traefik
```

Конфигурация Prometheus (`prometheus.yml`), который будет собирать метрики с Traefik:

```yaml
#prometheus.yml
global:
  scrape_interval: 15s # Как часто собирать метрики

scrape_configs:
  - job_name: 'traefik'
    static_configs:
      - targets: ['traefik:8082'] # Traefik в той же сети Docker
    metrics_path: '/metrics' # Путь к метрикам Traefik
```

### 2.2 Дашборд для Traefik — "Приборная панель"

После настройки можно импортировать готовый дашборд в Grafana. Вот ключевые метрики, которые должны быть на вашей "приборной панели":

### 2.3 Ключевые метрики для отслеживания

traefik_service_requests_total — "Сколько гостей пришло"

Простыми словами: Счётчик всех HTTP-запросов. Показывает общую нагрузку.

Что смотреть: Резкие падения (всё упало?) или резкие взлёты (DDoS?).

traefik_service_request_duration_seconds — "Сколько времени занимает провести гостя"

Простыми словами: Время обработки запроса. Главный показатель производительности ваших приложений.

Что смотреть: Если время растёт — ваше приложение не справляется с нагрузкой.

traefik_service_open_connections — "Сколько гостей сейчас в офисе"

Простыми словами: Текущее количество открытых соединений.

Что смотреть: Резкий рост может указывать на утечку соединений или атаку.

traefik_config_reloads_total — "Сколько раз менялись инструкции швейцару"

Простыми словами: Количество перезагрузок конфигурации Traefik.

Что смотреть: Частые перезагрузки могут говорить о нестабильности конфига.

## Часть 3: Оптимизация производительности

### 3.1 Балансировка нагрузки — "Распределение гостей"

Представьте, что у вас не один офис, а три. И вы хотите 70% гостей отправлять в старый проверенный офис, 20% — в новый с крутыми фичами, а 10% — в экспериментальный.

Пример weighted load balancing в dynamic.yml:

```yaml
http:
  services:
    my-app:
      weighted:
        services:
          - name: my-app-v1
            weight: 7 # 70% трафика
          - name: my-app-v2 
            weight: 2 # 20% трафика
          - name: my-app-canary
            weight: 1 # 10% трафика

    my-app-v1:
      loadBalancer:
        servers:
          - url: "http://app-v1:80"
    my-app-v2:
      loadBalancer:
        servers:
          - url: "http://app-v2:80"
    my-app-canary:
      loadBalancer:
        servers:
          - url: "http://app-canary:80"
```

### 3.2 Кэширование — "Запомнить частые запросы"

Зачем бегать за одним и тем же документом каждый раз, если можно его один раз запомнить и отдавать копию?

Пример Middleware для кэширования в dynamic.yml:

```yaml
http:
  middlewares:
    my-cache:
      plugin:
        traefik-plugin-http-cache: # Это пример плагина, который нужно установить
          default_storage_ttl: 300 # Хранить в кэше 5 минут
          storage:
            - name: memory
```

### 3.3 Оптимизация TLS/SSL — "Ускорение проверки документов"

HTTPS — это безопасно, но может быть медленно. Давайте ускорим процесс "проверки документов".

Оптимизация в traefik.yml:

```yaml
entryPoints:
  websecure:
    address: ":443"
    http2:
      maxConcurrentStreams: 250 # Настройка для HTTP/2
    http3: {} # Включаем современный HTTP/3
    # Настройки для TLS
    transport:
      lifeCycle:
        requestAcceptGraceTimeout: 0
        graceTimeOut: 10
      respondingTimeouts:
        readTimeout: 5
        writeTimeout: 5
        idleTimeout: 30
```

## Часть 4: Траблшутинг и анализ проблем

### 4.1 Анализ логов

Ваш швейцар ведёт детальный журнал. Давайте его почитаем с помощью утилиты jq.

Примеры команд:

```bash
# Показать 10 самых частых IP-адресов
cat access.log | jq -r '.ClientHost' | sort | uniq -c | sort -nr | head -10

# Найти все запросы с ошибками 500
cat access.log | jq 'select(.OriginStatus == 500)'

# Посчитать количество запросов по роутерам
cat access.log | jq -r '.RouterName' | sort | uniq -c | sort -nr
```

### 4.2 Использование метрик для диагностики

Проблема: Всё медленно.

Смотрим: traefik_service_request_duration_seconds. Если растёт — проблема в вашем приложении, а не в Traefik.

Проблема: Пользователи не могут подключиться.

Смотрим: traefik_service_open_connections. Если достигло лимита — возможно, приложение не закрывает соединения.

Проблема: Traefik "глючит".

Смотрим: traefik_config_reloads_failure_total. Если есть неудачи — ищите ошибку в динамической конфигурации.

Практический пример: Полная система мониторинга

Финальный docker-compose.yml для тестового стенда:

```yaml
services:
  traefik:
    image: traefik:v3.0
    container_name: traefik
    ports:
      - "80:80"
      - "443:443"
      - "8082:8082" # Метрики
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik.yml:/etc/traefik/traefik.yml:ro
      - ./dynamic.yml:/etc/traefik/dynamic.yml:ro
      - ./logs:/var/log/traefik
    networks:
      - traefik-net

  whoami:
    image: traefik/whoami
    container_name: whoami
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.mydomain.com`)"
      - "traefik.http.routers.whoami.entrypoints=websecure"
      - "traefik.http.routers.whoami.tls=true"
    networks:
      - traefik-net

  prometheus:
    image: prom/prometheus:latest
    # ... конфиг как выше
    networks:
      - traefik-net

  grafana:
    image: grafana/grafana:latest
    # ... конфиг как выше
    networks:
      - traefik-net

networks:
  traefik-net:
```

## Заключение

Поздравляю! Теперь у вашего швейцара есть не только глаза и уши, но и целый центр управления полётами. Вы можете видеть всю статистику, анализировать проблемы и оптимизировать производительность.

Что дальше? В следующей статье мы перейдём к продвинутым сценариям: Canary-деплойменты, зеркалирование трафика, кастомные плагины и подготовка к высоким нагрузкам в продакшене.

Практическое задание для студентов:
Разверните стенд по финальному docker-compose.yml.

Настройте доступ к Дашборду Traefik по собственному домену с паролем.

Подключите Prometheus к метрикам Traefik и импортируйте в Grafana любой готовый дашборд для Traefik.

Сымитируйте проблему: запустите команду while true; do curl https://whoami.mydomain.com; done и понаблюдайте, как меняются графики в реальном времени.

Найдите в логах все запросы от того IP, с которого запускали нагрузочный тест.

Теперь вы не слепые разработчики, а полноценные DevOps-инженеры, видящие всю картину! Удачи в экспериментах

