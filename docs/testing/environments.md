# Окружения, variables и secrets

GitHub Environments `dev`, `alpha` и `main` изолируют URL и секреты контуров. Настройки находятся в `drill-cloud-test → Settings → Environments`.

## Обязательный минимум

### Variables

| Variable | `dev` | `alpha` | `main` |
|---|---|---|---|
| `E2E_BASE_URL` | `https://dev.drillcloud.ru` | `https://alpha.drillcloud.ru` | `https://beta.drillcloud.ru` |
| `E2E_API_URL` | `https://dev.drillcloud.ru/api` | `https://alpha.drillcloud.ru/api` | `https://beta.drillcloud.ru/api` |
| `E2E_AUTH_MODE` | `required` | `required` | `required` |
| `E2E_SEED_ENABLED` | `false` | `false` | `false` |
| `E2E_PUBLISH_LIVE` | `false` | `false` | `false` |
| `E2E_REQUIRE_HISTORY_DATA` | `false` | `false` | `false` |
| `E2E_REQUIRE_VIDEO_PLAYBACK` | `false` | `false` | `false` |
| `E2E_LIVE_WAIT_SECONDS` | `30` | `30` | `30` |
| `E2E_SSE_OBSERVE_SECONDS` | `8` | `8` | `8` |
| `E2E_MAX_CURRENT_REQUESTS` | `12` | `12` | `12` |

### Secrets

| Secret | Назначение |
|---|---|
| `E2E_USERNAME` | Логин основной технической учётной записи Keycloak |
| `E2E_PASSWORD` | Пароль основной технической учётной записи |

Не используйте личную или административную учётную запись Keycloak. Пароли не записываются в документацию, `.env`, repository variables или логи.

## Проверка ролей

Для полного P1 создайте отдельные пары secrets:

| Secrets | Права пользователя |
|---|---|
| `E2E_ADMIN_USERNAME`, `E2E_ADMIN_PASSWORD` | роль `drill-admin` |
| `E2E_EDGE_USERNAME`, `E2E_EDGE_PASSWORD` | только `drill-edge-<E2E_EDGE_ID>` |
| `E2E_NO_ROLE_USERNAME`, `E2E_NO_ROLE_PASSWORD` | без `drill-admin` и `drill-edge-*` |

Если одна пара отсутствует, соответствующий role-тест помечается `SKIPPED` с указанием недостающих variables.

## Подготовленные объекты и интеграции

| Variable | Назначение |
|---|---|
| `E2E_EDGE_ID` | Буровая для current/history |
| `E2E_FORBIDDEN_EDGE_ID` | Буровая, недоступная edge-пользователю |
| `E2E_VIDEO_EDGE_ID` | Буровая с камерой |
| `E2E_NO_VIDEO_EDGE_ID` | Буровая без камер |
| `E2E_INDICATOR_QUERY` | Стабильный поиск показателя |
| `E2E_HISTORY_TAG_QUERY` | Показатель с историей |
| `E2E_LIVE_TAG` | Меняющийся live-показатель |
| `E2E_VIDEO_WS_URL` | Тестовый WebSocket-видеопоток |
| `E2E_UI_COMMIT` | Коммит Frontend для заголовка отчёта |
| `E2E_CLOUD_COMMIT` | Коммит Backend для заголовка отчёта |

Если `E2E_VIDEO_EDGE_ID` не задан, тест пытается найти первую буровую с камерой. Если подходящей буровой нет, это будет ошибка подготовки стенда, а не `SKIPPED`. То же относится к буровой без камер.

Для стабильного теста обновления current задайте `E2E_LIVE_TAG` и обеспечьте публикацию меняющегося значения. Иначе тест выберет существующий тег, который может оказаться статичным.

## ReportPortal

В каждом Environment:

```text
REPORTPORTAL_ENABLED=true
REPORTPORTAL_ENDPOINT=https://reportportal.drillcloud.ru
REPORTPORTAL_PROJECT=drill_cloud
```

Secret:

```text
REPORTPORTAL_API_KEY=<ключ технического пользователя>
```

Endpoint указывается без `/api` и завершающего `/`.

## Важная особенность visual-тестов

Текущий workflow явно задаёт `E2E_VISUAL_ENABLED=true`. Поэтому repository/environment variable с таким именем не отключит visual-тесты в профилях `p2` и `all`. Для изменения поведения нужно править `.github/workflows/e2e.yml` либо не выбирать профиль, содержащий visual marker.

## Защита PROD

- `E2E_SEED_ENABLED=true` на `main` останавливает job до запуска тестов;
- `E2E_PUBLISH_LIVE=true` на `main` также запрещён;
- тесты marker `settings` на `main` исключаются;
- тестовый job не должен входить в required status checks branch protection.
