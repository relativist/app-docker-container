# Agent Instructions

Этот каталог содержит общий gateway-контур для нескольких frontend-приложений.

## Назначение

- Внешний трафик идет только через один общий `nginx`.
- Приложения публикуются по path prefix, а не по разным внешним портам.
- Текущая схема:
  - `/` -> стартовая HTML-страница со списком приложений
  - `/pc-man/` -> приложение `pc-man`

## Текущая структура

- [docker-compose.yml](/home/rest/intabia/app-docker-container/docker-compose.yml) содержит общий `gateway` и внутренние frontend-сервисы.
- [nginx/conf.d/default.conf](/home/rest/intabia/app-docker-container/nginx/conf.d/default.conf) содержит маршрутизацию по path prefix.
- [nginx/html/index.html](/home/rest/intabia/app-docker-container/nginx/html/index.html) содержит стартовую страницу со ссылками на приложения.
- `pc-man` подключен из соседнего каталога `../pc-man`.

## Как добавлять новое приложение

1. Добавить новый сервис в `docker-compose.yml`.
2. Указать для него свой `build.context`, обычно в виде `../<app-name>`.
3. Передать в build args свой `APP_BASE_PATH`, например `/crm/`.
4. Не публиковать отдельный внешний `ports` для приложения.
5. Добавить в `nginx/conf.d/default.conf`:
   - `location = /<app-name> { return 301 /<app-name>/; }`
   - `location /<app-name>/ { proxy_pass http://<service-name>/; ... }`
6. Добавить ссылку на приложение в `nginx/html/index.html`.

## Ограничения

- Не ломать существующую base-path подготовку приложений.
- Снаружи должен публиковаться только один порт у `gateway`.
- Для внутренних приложений использовать только внутреннюю docker-сеть.
- Не делать коммиты, пока явно не попросят.
- Не использовать сеть без запроса.
- Все изменения держать в рамках текущего workspace.

## Проверка после изменений

- Проверять итоговую конфигурацию командой `docker compose config`.
- Если менялась маршрутизация, перепроверять соответствие между:
  - `APP_BASE_PATH`
  - `location /<prefix>/`
  - ссылкой на главной странице gateway
