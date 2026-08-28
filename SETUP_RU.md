# 🚀 Scraperr — быстрый старт (RU)

Scraperr — self-hosted веб-скрейпер без кода: XPath-извлечение, очередь задач,
спайдеринг домена, скачивание медиа, экспорт в markdown/csv.

> Это поддерживаемый форк заброшенного (archived 10.2025) проекта
> `jaypyles/Scraperr`. Обновлены зависимости, починены Docker-сборки и баги.

## Состав

| Контейнер | Порт | Что это |
|-----------|------|---------|
| `scraperr` (frontend) | `3000` | Next.js UI + прокси на API |
| `scraperr_api` | `8000` | FastAPI + воркер скрейпинга |

База по умолчанию — **SQLite** (`data/database.db`), переопределяется через
`DATABASE_URL`.

## Вариант 1 — готовые образы (быстро)

```bash
docker compose -f docker-compose.hub.yml up -d
```

- UI: http://localhost:3000
- API docs: http://localhost:8000/docs

## Вариант 2 — локальная сборка (рекомендуется для форка)

```bash
docker compose up -d
```

> ⚠️ Первая сборка API долгая (устанавливает Playwright + Camoufox-браузеры,
> несколько ГБ). Фронтенд собирается быстрее (~2–3 мин).

## Почему фронтенд раньше «не видел» API

`NEXT_PUBLIC_API_URL` и `SERVER_URL` в этом проекте используются **на
серверной стороне** Next.js (в прокси-роутах `src/pages/api/*` и
`getServerSideProps`). Next.js не вшивает их в серверный бандл — они читаются
из runtime-окружения. Поэтому достаточно:

1. задать их в `environment` контейнера фронтенда:
   `NEXT_PUBLIC_API_URL=http://scraperr_api:8000`;
2. запустить оба контейнера в одной Docker-сети (имя `scraperr_api`
   резолвится по DNS).

## Переменные окружения

| Переменная | По умолчанию | Назначение |
|------------|--------------|------------|
| `NEXT_PUBLIC_API_URL` | `http://scraperr_api:8000` | URL API для серверного прокси |
| `SERVER_URL` | `http://scraperr_api:8000` | URL API в server-side props |
| `DATABASE_URL` | `sqlite+aiosqlite:///data/database.db` | строка подключения SQLAlchemy |
| `OPENAI_KEY` | _(пусто)_ | включает AI-ассистента |
| `DEFAULT_USER_EMAIL` / `DEFAULT_USER_PASSWORD` | _(пусто)_ | предзаполненный админ |

## Проверка работоспособности

```bash
# страница открывается
curl -I http://localhost:3000/

# фронтенд видит API (прокси-роут)
curl -X POST http://localhost:3000/api/check -H 'Content-Type: application/json' -d '{}'
# → {"ai_enabled":false,"registration":true,"recordings_enabled":false}

# регистрация (клиент шлёт обёртку {data:{...}})
curl -X POST http://localhost:3000/api/signup \
  -H 'Content-Type: application/json' \
  -d '{"data":{"email":"a@b.c","password":"pass","full_name":"Name"}}'

# логин → access_token
curl -X POST http://localhost:3000/api/token \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'username=a@b.c&password=pass'
```

## Починенные баги

- **#100** — `schedule_cron_job` падал с TypeError (передавал SQLAlchemy-модель
  вместо dict в `insert_job_from_cron_job`). Исправлено в
  `api/backend/job/job_router.py`.
- **Frontend/API связка** — обновлён `docker-compose.yml` (локальная сборка +
  корректные env) и `docker/frontend/Dockerfile` (Node 22 + CMD `next start`).
