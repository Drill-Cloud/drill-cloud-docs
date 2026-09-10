# Управляемые тестовые данные

P0 может работать с существующим стендом, но live/history/video и роли становятся воспроизводимыми только на заранее известных объектах.

## Что создаёт seed

- `e2e-main` — current и история;
- `e2e-no-video` — буровая без камер;
- `e2e-video` — камера при заданном `E2E_VIDEO_WS_URL`;
- теги `e2e-depth`, `e2e-pressure`, `e2e-live`;
- почасовая история за месяц и пятиминутная история за сутки.

Seed изменяет только ID с префиксом `e2e-` и отказывается трогать остальные установки.

## Локальная подготовка

В локальном `.env` задайте доступ к выбранному non-prod стенду:

```dotenv
E2E_DATABASE_URL=postgresql://user:password@host:5432/app
E2E_INGEST_API_KEY=
E2E_VIDEO_WS_URL=wss://example.test/video/e2e
```

Создать или обновить данные:

```bash
bash scripts/seed-test-data.sh
```

Публиковать `e2e-live` пять минут:

```bash
bash scripts/publish-live-data.sh --duration 300
```

Удалить только E2E-объекты:

```bash
bash scripts/seed-test-data.sh cleanup
```

## GitHub Actions

Для автоматического seed/live publisher в DEV или ALPHA нужны:

| Тип | Имя | Назначение |
|---|---|---|
| Secret | `E2E_DATABASE_URL` | Строка подключения к БД контура |
| Secret | `E2E_INGEST_API_KEY` | Ключ backend ingest, если защита включена |
| Variable | `E2E_SEED_ENABLED=true` | Запустить seed перед тестами |
| Variable | `E2E_PUBLISH_LIVE=true` | Публиковать live-значение во время тестов |

GitHub-hosted runner должен видеть PostgreSQL. Не открывайте БД в интернет ради тестов: используйте self-hosted runner либо подготовьте данные из внутренней сети и оставьте seed выключенным.

## PROD

На `main` seed и live publisher запрещены в самом workflow. Production проверяется только на существующих данных и без изменения пользовательских настроек.

## Учётные записи ролей

Нужны три отдельные учётные записи Keycloak: admin, edge-only и no-role. Имена и пароли хранятся только в GitHub Environment secrets. После каждого settings-теста исходные настройки восстанавливаются в `finally`.
