---
draft: true
title: '3. Nginx как обратный прокси'
description: "Nginx как обратный прокси: управление upstream и тонкости proxy_pass"
author: 'DigitalChip'
tags: ["nginx","https","proxy"]
categories: ["nginx","devops"]
---

# Nginx как обратный прокси: управление upstream и тонкости proxy_pass

В предыдущих статьях мы научились маршрутизировать запросы внутри Nginx. Но в современном мире Nginx редко работает один — чаще он становится интеллектуальным посредником, направляющим трафик к различным бэкендам. Сегодня мы разберём, как превратить Nginx в полноценный обратный прокси, способный работать с микросервисами, Node.js-приложениями, Python-бэкендами и любыми другими сервисами.

## Директива proxy_pass: больше, чем просто перенаправление

На первый взгляд, директива `proxy_pass` кажется простой: взял входящий запрос, переслал на другой сервер. Но нюансы начинаются с самого синтаксиса:

```nginx
location /api/ {
    # Вариант 1: без слэша в конце
    proxy_pass http://backend;
    
    # Вариант 2: со слэшем в конце  
    proxy_pass http://backend/;
}
```

Разница фундаментальна. В первом случае Nginx передаст бэкенду полный URI, включая префикс `/api/`. Запрос `/api/users` уйдёт на `http://backend/api/users`. Во втором случае — URI будет заменён! Запрос `/api/users` превратится в `/users`.

Почему это важно? Представьте, что ваш бэкенд ожидает пути без префикса `/api/`. Правильный выбор синтаксиса определяет, будет ли приложение работать.

Более сложный сценарий — динамическое проксирование через переменные:

```nginx
location ~ ^/service/(?<service_name>\w+) {
    proxy_pass http://$service_name.internal;
}
```

Здесь запрос `/service/auth` автоматически направляется на `http://auth.internal`. Это основа для микросервисной архитектуры, где Nginx становится единой точкой входа.

## Группы бэкендов: контекст upstream

Одиночный бэкенд — это точка отказа. Реальная сила Nginx раскрывается при работе с группами серверов:

```nginx
upstream app_backend {
    server 127.0.0.1:8000 weight=3;
    server 10.0.1.1:8000 max_fails=3 fail_timeout=30s;
    server 10.0.1.2:8000 backup;
}
```

По умолчанию Nginx использует алгоритм балансировки `round-robin`, но это не всегда оптимально. Рассмотрим стратегии балансировки:

`least_conn` — отправляет запрос на сервер с наименьшим количеством активных соединений

`ip_hash` — сохраняет сессию клиента на одном сервере (устаревший подход, лучше использовать sticky-сессии на уровне приложения)

`hash` — гибкое распределение по любому ключу (например, `hash $remote_addr consistent;`)

Параметры `max_fails` и `fail_timeout` реализуют пассивные `health checks`. Если сервер трижды подряд не отвечает (по умолчанию — таймаут соединения), он временно исключается из ротации на 30 секунд. Сервер с меткой `backup` используется только когда все основные недоступны.

## Заголовки: контекст — это всё

Когда Nginx проксирует запрос, он модифицирует заголовки. По умолчанию он передаёт не всё, что нужно современным приложениям. Рассмотрим критически важные заголовки:

```nginx
location / {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Host $host;
}
```

`Host` — без него многие приложения не смогут определить, для какого домена предназначен запрос

`X-Real-IP` — единственный IP клиента (если Nginx стоит за другим прокси, используйте `$proxy_protocol_addr`)

`X-Forwarded-For` — цепочка прокси, через которые прошёл запрос

`X-Forwarded-Proto` — оригинальная схема (http/https)

***Важное предупреждение***: `proxy_set_header` полностью перезаписывает заголовок. Если вам нужно добавить значение, а не заменить, придется использовать map или переменные:

```nginx
map $http_x_custom_header $new_x_custom_header {
    default "$http_x_custom_header, nginx";
    ''      "nginx";
}
```

`proxy_set_header X-Custom-Header $new_x_custom_header;`

## Управление соединениями: таймауты и буферы

Проксирование — это не просто "переслал и забыл". Nginx управляет двумя соединениями: клиент→Nginx и Nginx→бэкенд.

### Таймауты

```nginx
location / {
    proxy_connect_timeout 5s;    # Таймаут на установку соединения с бэкендом
    proxy_send_timeout 10s;      # Таймаут на отправку запроса бэкенду
    proxy_read_timeout 30s;      # Таймаут на чтение ответа от бэкенда
}
```

`proxy_read_timeout` особенно важен для `long-polling` или Server-Sent Events. Если ваше приложение использует подобные технологии, увеличьте это значение.

### Буферизация

По умолчанию Nginx буферизует ответы бэкенда. Это эффективно, но не всегда нужно:

```nginx
location /api/stream {
    proxy_buffering off;  # Отключаем буферизацию для стриминга
    proxy_pass http://backend;
}


location / {
    proxy_buffering on;
    proxy_buffer_size 4k;      # Первый буфер для заголовков
    proxy_buffers 8 16k;       # 8 буферов по 16k для тела ответа
    proxy_busy_buffers_size 24k; # Размер буферов, которые можно отправить клиенту
}
```

### Keep-alive соединения

По умолчанию Nginx использует HTTP/1.0 для соединений с бэкендом. Это неэффективно:

```nginx
upstream backend {
    server 127.0.0.1:8000;
    keepalive 32;  # Количество keep-alive соединений в пуле
}

location / {
    proxy_http_version 1.1;           # Переходим на HTTP/1.1
    proxy_set_header Connection "";   # Отключаем close, разрешаем keep-alive
    proxy_pass http://backend;
}
```
Такая конфигурация ускоряет обработку запросов в 2-5 раз за счет повторного использования соединений.

## Обработка ошибок и ретраи

Бэкенды падают, сеть глючит, базы данных тормозят. Прокси должен быть устойчив к этим проблемам:

```nginx
upstream backend {
    server backend1.example.com;
    server backend2.example.com;
}

location / {
    proxy_next_upstream error timeout http_500 http_502 http_503 http_504;
    proxy_next_upstream_timeout 0;
    proxy_next_upstream_tries 3;
    
    proxy_intercept_errors on;
    error_page 500 502 503 504 /50x.html;
    
    proxy_pass http://backend;
}
```

`proxy_next_upstream` определяет, в каких случаях пробовать следующий сервер. Обратите внимание: `http_404` в этом списке обычно не нужен — "Not Found" это валидный ответ приложения, а не ошибка сервера.

`proxy_intercept_errors` позволяет Nginx обрабатывать ошибки бэкенда с помощью директивы error_page. Без неё код 500 от бэкенда уйдёт прямиком клиенту.

## Полная конфигурация: собираем всё вместе

Рассмотрим типичный сценарий: веб-приложение на Node.js с отдельной раздачей статики и SSL-терминацией на Nginx:

```nginx
# Определяем группу бэкендов
upstream node_app {
    least_conn;               # Балансировка по наименьшим соединениям
    server 127.0.0.1:3000 weight=2;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002 backup;
    keepalive 16;
}

server {
    listen 443 ssl http2;
    server_name app.example.com;
    
    ssl_certificate /etc/ssl/app.example.com.crt;
    ssl_certificate_key /etc/ssl/app.example.com.key;
    
    # Статика обслуживается напрямую Nginx
    location /static/ {
        root /var/www/app;
        expires 1y;
        add_header Cache-Control "public, immutable";
        
        # Безопасность: не передаём статику в приложение
        try_files $uri =404;
    }
    
    location /uploads/ {
        root /var/www/app;
        
        # Защита от горячих ссылок (опционально)
        valid_referers none blocked app.example.com;
        if ($invalid_referer) {
            return 403;
        }
    }
    
    # Все динамические запросы — в приложение
    location / {
        # Базовые настройки прокси
        proxy_pass http://node_app;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        
        # Критически важные заголовки
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        
        # Таймауты
        proxy_connect_timeout 3s;
        proxy_send_timeout 10s;
        proxy_read_timeout 30s;
        
        # Буферизация (оптимизирована для API и HTML)
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 16k;
        proxy_busy_buffers_size 32k;
        proxy_max_temp_file_size 0;
        
        # Ретраи и обработка ошибок
        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_next_upstream_tries 2;
        proxy_intercept_errors on;
        
        # Защита от медленных клиентов
        proxy_ignore_client_abort off;
    }
    
    # Кастомизированная страница ошибок
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
        internal;
    }
}
```

Обратите внимание на разделение статики и динамики. Статические файлы обслуживаются напрямую Nginx — это в разы эффективнее, чем проксировать их через Node.js. SSL-терминация на Nginx разгружает приложение от криптографических операций.

## Заключение: Nginx как интеллектуальный диспетчер

В современной архитектуре Nginx редко бывает конечным пунктом запроса. Чаще он выступает как интеллектуальный диспетчер: распределяет нагрузку, обеспечивает отказоустойчивость, термирует SSL, кэширует ответы и защищает уязвимые бэкенды.

Настройка проксирования — это баланс между производительностью и надёжностью. Слишком агрессивные таймауты увеличат количество ошибок, слишком мягкие — риск зависаний. Отсутствие правильных заголовков сломает логику приложения, а их избыток — создаст уязвимости.

В следующей статье мы поговорим о кэшировании в Nginx — от статических файлов до динамических ответов API. Вы узнаете, как снизить нагрузку на бэкенды в десятки раз и ускорить доставку контента пользователям.

Помните: хорошо настроенный прокси невидим для пользователей. Они просто получают быстрый и стабильный сервис, не задумываясь о том, сколько серверов работает на заднем плане и как Nginx управляет этой сложностью.
