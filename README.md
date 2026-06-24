# app-docker-container

Общий входной контейнер для frontend-приложений, которые открываются по префиксам путей через один `nginx`, а не по отдельным внешним портам.

Сейчас подключено:

- `/pc-man/` -> сервис `pc-man`
- `/chess-quest/` -> сервис `chess-quest` + внутренний Postgres `chess-quest-postgres`

## Запуск

```bash
docker compose up --build
```

По умолчанию `gateway` публикует `80:80`. Если порт `80` на хосте уже занят, можно запустить на другом внешнем порту:

```bash
GATEWAY_PORT=8080 docker compose up --build
```

Тогда приложения будут доступны по:

- `http://host:8080/pc-man/`
- `http://host:8080/chess-quest/`

## chess-quest и Postgres

`chess-quest` собирается из соседнего каталога `../chess-quest` с base path `/chess-quest` и не публикует отдельный внешний порт. Nginx проксирует путь `/chess-quest/` без обрезания префикса, потому что Next.js собран с `NEXT_PUBLIC_BASE_PATH=/chess-quest`.

Для `chess-quest` поднимается отдельный внутренний Postgres:

- сервис: `chess-quest-postgres`
- volume: `chess-quest-postgres-data`
- database: `${CHESS_QUEST_POSTGRES_DB:-chess_quest}`
- user: `${CHESS_QUEST_POSTGRES_USER:-postgres}`
- password: `${CHESS_QUEST_POSTGRES_PASSWORD:-postgres}`

При старте контейнер `chess-quest` выполняет `prisma migrate deploy`, затем `npm run db:seed`. Seed идемпотентный: встроенные карты и карточки создаются или обновляются через upsert.

## Как добавить следующее приложение

1. Добавить новый сервис в [docker-compose.yml](/home/rest/intabia/app-docker-container/docker-compose.yml) по аналогии с `pc-man`.
2. Указать для него свой `build.context` и свой `APP_BASE_PATH`, например `/crm/`.
3. Не публиковать внешний `ports`; достаточно `expose: - "80"` или даже можно обойтись без него.
4. Добавить в [nginx/conf.d/default.conf](/home/rest/intabia/app-docker-container/nginx/conf.d/default.conf) два правила:
   - `location = /crm { return 301 /crm/; }`
   - `location /crm/ { proxy_pass http://crm/; ... }`
5. Убедиться, что само приложение собрано с тем же base path и его router использует тот же префикс.

## Шаблон для нового сервиса

```yaml
  crm:
    build:
      context: ../crm
      dockerfile: Dockerfile
      args:
        APP_BASE_PATH: /crm/
    restart: unless-stopped
    expose:
      - "80"
    networks:
      - frontend-gateway
```

## Шаблон для нового location

```nginx
location = /crm {
  return 301 /crm/;
}

location /crm/ {
  proxy_pass http://crm/;
  proxy_http_version 1.1;
  proxy_set_header Host $host;
  proxy_set_header X-Real-IP $remote_addr;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Proto $scheme;
  proxy_set_header X-Forwarded-Prefix /crm;
  proxy_redirect off;
}
```
