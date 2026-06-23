# Nginx для Angular SSR в FastPanel

## Для чего нужен раздел

Эта страница нужна технической команде, когда iEXExchanger установлен со связкой:

```text
frontend-домен example.com -> Angular SSR
backend/admin-домен app.example.com -> Laravel backend
browser API -> example.com/backend-api/*
```

Обычная Laravel-настройка Nginx через `public/index.php` подходит для backend/admin-домена. Для основного клиентского frontend-домена нужна отдельная схема: Nginx проксирует Angular SSR на локальный Node-процесс и отдельно проксирует API в backend.

## Базовая схема

| Компонент | Пример | Что делает |
| --- | --- | --- |
| Frontend-домен | `https://example.com` | Клиентский сайт обменника |
| Backend/admin-домен | `https://app.example.com` | Laravel, админка, API, платежные callback |
| Angular SSR | `127.0.0.1:4000` | Серверный рендер клиента |
| Browser API | `/backend-api/*` | Same-origin путь из браузера в backend |
| Backend upstream | `https://app.example.com` или origin IP с `Host: app.example.com` | Куда Nginx передает API |
| Socket/Reverb | обычно `127.0.0.1:8080` или backend-домен | Realtime-соединения |

## Почему `/backend-api`, а не прямой backend-домен

Браузер должен обращаться к API через основной домен:

```text
https://example.com/backend-api/client-api/v1/...
```

А Nginx уже внутри сервера передает запрос в backend:

```text
https://app.example.com/client-api/v1/...
```

Так frontend работает same-origin и не упирается в CORS. Backend при этом остается на своем домене, со своим PHP-FPM, cookies, middleware, callback и логикой Laravel.

Старый подход через `/apis`, `alias` на backend `public` и `fastcgi_pass` внутри frontend-домена лучше не использовать как основную схему. Он смешивает frontend и backend в одном server block и повышает риск ошибок с `SCRIPT_FILENAME`, fallback routes, cookies, hidden files и FastPanel include-ами. `/apis` можно оставить только временно как compatibility alias для старой frontend-сборки.

## Что должно быть в Angular SSR `.env`

Для production FastPanel/PM2 схемы root `.env` frontend-проекта должен быть заполнен примерно так:

```bash
NODE_ENV=production
PORT=4000
HOST=127.0.0.1
PM2=true
SUPPORTED_LANGUAGES=ru,en,pl,uk,zh,ka
DEFAULT_LANGUAGE=ru
ALLOWED_HOSTS=example.com,www.example.com
API_URL=https://app.example.com
CSS_VERSION=1
CUSTOM_THEME=custom-theme
CUSTOM_THEME_URL=/static/theme/custom-theme.css
CLIENT_CUSTOM_SCRIPTS_ENABLED=false
CLIENT_CUSTOM_SSR_ENABLED=false
CLIENT_CUSTOM_DIR=/var/www/fastpanel_user/data/www/example.com/public/ng-custom
```

Важно:

| Поле | Как заполнять |
| --- | --- |
| `PORT` | Должен совпадать с Nginx upstream, обычно `4000` |
| `HOST` | Для production на том же сервере обычно `127.0.0.1` |
| `ALLOWED_HOSTS` | Только реальные frontend-домены клиента, не `*` |
| `API_URL` | Backend-origin для SSR, например `https://app.example.com` |
| `CLIENT_CUSTOM_DIR` | Внешняя папка `/ng-custom/`, которую Nginx отдает через `alias` |

Не ставьте `API_URL=/backend-api` в root `.env` для PM2/FastPanel SSR. `/backend-api` - это путь браузера через Nginx, а SSR-серверу нужен backend-origin.

## Что добавить вне `server`

Вверху Nginx-конфига, до `server { ... }`, обычно нужны:

| Блок | Для чего нужен |
| --- | --- |
| `map $http_accept_language $accept_language` | Выбор языка по браузеру |
| `map $cookie_lang $selected_lang` | Выбор языка по cookie |
| `map $http_upgrade $connection_upgrade` | WebSocket upgrade |
| `upstream frontend_ssr` | Локальный Angular SSR, обычно `127.0.0.1:4000` |
| `upstream backend_https` | Backend-домен или origin IP backend-домена |

Если у клиента включены только `ru,en`, языки в `map` и regex должны совпадать с реальными языками проекта.

## Что должно быть внутри `server` frontend-домена

Внутри server block FastPanel оставьте SSL, `root`, `index`, `set $root_path`, `disable_symlinks` и другие служебные строки панели. После этого добавляются рабочие правила frontend-схемы.

| Блок | Зачем нужен |
| --- | --- |
| `add_header Vary "Accept-Language, Cookie" always` | Корректное кеширование языковых страниц |
| `client_max_body_size 64m` | Чтобы загрузки документов/файлов не падали с 413 |
| `set $backend_domain app.example.com` | Backend Host/SNI для proxy |
| `location = /` | Редирект на язык по умолчанию или выбранный язык |
| static `location` для JS/CSS/assets | Должен стоять выше locale SSR location |
| locale SSR `location` | Проксирует `/ru/...`, `/en/...` и другие языки в Angular SSR |
| `location ^~ /ng-custom/` | Отдает клиентские вставки без долгого кеша |
| `location ^~ /backend-api/` | Проксирует browser API в backend |
| aliases `/images`, `/storage`, `/static`, `/dist`, `/exports` | Отдают публичные backend-файлы с frontend-домена |
| `location ^~ /app/` и `/apps/` | Нужны, если socket/Reverb идет через основной домен |

## Главное правило по static location

Static location должен стоять выше языкового SSR location.

Иначе файлы вроде `/ru/main-*.js` и `/ru/styles-*.css` могут получить языковые cookies/headers и плохо кешироваться через Cloudflare или браузер.

Ожидаемо для hashed JS/CSS:

```text
Cache-Control: public, max-age=31536000, immutable
нет Set-Cookie
нет Vary: Cookie
```

## `/ng-custom/`

Для production не используйте:

```nginx
proxy_pass http://localhost:4200;
```

Это только Angular dev server.

Production-папка должна быть внешней, например:

```text
/var/www/fastpanel_user/data/www/example.com/public/ng-custom
```

Nginx должен отдавать ее через `alias`, а SSR должен читать ее через `CLIENT_CUSTOM_DIR`. Для `/ng-custom/` нужен `Cache-Control: no-store`, чтобы клиентские вставки, verification meta, пиксели или сторонние скрипты обновлялись без пересборки Angular.

## `/backend-api/`

Canonical API location:

```text
location = /backend-api -> redirect to /backend-api/
location ^~ /backend-api/ -> proxy to backend upstream
```

В proxy важно сохранить:

| Header/настройка | Зачем |
| --- | --- |
| `proxy_ssl_server_name on` | Корректный SNI для HTTPS backend |
| `proxy_ssl_name $backend_domain` | Backend получает правильное имя |
| `proxy_set_header Host $backend_domain` | Laravel видит backend-домен |
| `X-Forwarded-Host $host` | Backend знает публичный frontend host |
| `X-Forwarded-Proto $scheme` | Корректные HTTPS-ссылки |
| `proxy_cookie_domain` | Cookie backend корректно работают на frontend-домене |

Если `/backend-api/client-api/v1/ping` возвращает HTML от Angular, значит API location не сработал и запрос ушел в SSR fallback.

Если backend отвечает редиректом на `/ru/client-api/...`, часто причина в неправильном `Host` для backend proxy.

## Публичный API `/api/v3`

Если новый публичный Exchanger API должен быть доступен same-origin с frontend-домена, его тоже можно проксировать через Nginx:

```text
https://example.com/api/v3/*
```

Он должен уходить в backend upstream с корректным backend Host/SNI, как и `/backend-api`.

## Socket/Reverb

Если backend `/start` отдает socket host основного домена и порт `443`, на frontend-домене нужны proxy-блоки:

```text
/app/*
/apps/*
```

Они должны передавать `Upgrade` и `Connection` headers и иметь длинные `proxy_read_timeout` / `proxy_send_timeout`, обычно до `3600s`.

Если socket остается на backend-домене или отдельном порту, эти блоки на frontend-домене могут быть не нужны, но backend-домен и firewall должны быть настроены корректно.

## Проверка после настройки

Минимальные проверки:

```bash
nginx -t
curl -I https://example.com/backend-api/client-api/v1/ping
curl -I https://example.com/dist/assets/icons/locale/ru.svg
curl -I https://example.com/ru/
curl -I -H 'Host: example.com' http://127.0.0.1:4000/ru/
```

Что должно быть:

| Проверка | Ожидаемый результат |
| --- | --- |
| `nginx -t` | Конфиг валиден |
| `/backend-api/...` | Backend JSON или backend-статус, не Angular HTML |
| `/dist/assets/...` | Файл открывается с frontend-домена |
| `/ru/` | Страница SSR открывается |
| `127.0.0.1:4000/ru/` с Host | SSR отвечает локально |
| Browser DevTools | Нет запросов на `https://app.example.com/client-api/...` |

## Частые ошибки

| Ошибка | Что происходит | Как исправить |
| --- | --- | --- |
| `API_URL=/backend-api` в SSR `.env` | SSR не знает backend-origin | Указать `https://app.example.com` |
| `ALLOWED_HOSTS=*` | Angular предупреждает о риске host validation | Указать реальные frontend-домены |
| `/ng-custom/` проксируется на `localhost:4200` | Production зависит от dev server | Использовать `alias` на runtime-папку |
| Старый `/apis` остался основным API | Смешиваются frontend и backend правила | Перейти на `/backend-api`, `/apis` оставить временно |
| `/backend-api` отдает Angular HTML | Nginx location не совпал или стоит ниже fallback | Поднять API location выше fallback и проверить `proxy_pass` |
| Backend редиректит на `/ru/client-api/...` | Backend получил неверный Host | Проверить `proxy_set_header Host` и `proxy_ssl_name` |
| Static JS/CSS получает cookie | Static location ниже языкового SSR location | Переставить static location выше |
| 413 при загрузке | Малый `client_max_body_size` | Увеличить, обычно до `64m` или по требованиям проекта |

## Что отвечать в AI-чате

| Вопрос | Ответ |
| --- | --- |
| "Как настроить Nginx для frontend?" | "Frontend-домен проксирует Angular SSR на `127.0.0.1:4000`, а API должен идти через `/backend-api` в backend-домен. Backend Laravel остается на отдельном домене с PHP-FPM 8.4." |
| "Почему API дает CORS?" | "Браузер должен ходить на `https://frontend-domain/backend-api/...`, а не напрямую на backend-домен. Проверьте `API_URL`, frontend build и Nginx proxy." |
| "Почему `/backend-api` открывает сайт, а не API?" | "Nginx не попал в API location. Проверьте порядок location: `/backend-api/` должен быть выше SSR fallback." |
| "Что писать в `API_URL`?" | "Для SSR `.env` пишите backend-origin, например `https://app.example.com`. `/backend-api` нужен браузеру через Nginx, а не SSR-серверу." |
| "Можно оставить `/apis`?" | "Только временно для старой frontend-сборки. Новая каноническая схема - `/backend-api`." |

## Связанные разделы

FastPanel, Nginx и PHP 8.4, CRON, очереди и Horizon, Защита FastPanel и сервера, После покупки и запуск, Настройки системы, Контент и дизайн, Ошибки и диагностика.
