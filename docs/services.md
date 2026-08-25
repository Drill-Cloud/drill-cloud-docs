# Сервисы и переменные окружения

Документ фиксирует границы сервисов и минимальную конфигурацию локального контура. Полные списки переменных остаются в `.env.example` соответствующих репозиториев.

## Компоненты и порты

| Компонент | Назначение | Локальный адрес |
|---|---|---|
| Node-RED | эмуляция и публикация edge-данных | `http://localhost:1880` |
| Mosquitto | локальная MQTT-шина | `mqtt://localhost:1883` |
| mqtt-ingest | преобразование MQTT в HTTP и видео WebSocket | HTTP `8080`, WS `9090` |
| cloud | API, current/history и миграции БД | `http://localhost:3101/api` |
| UI | пользовательский интерфейс | `http://localhost:5173` |
| TimescaleDB | локальные данные cloud | `localhost:5432` |
| MQTT monitor | необязательный read-only монитор | UI `5174`, API `3205` |
| Keycloak | серверный SSO | `https://sso.drillcloud.ru` |

## cloud

```dotenv
PORT=3101
DATABASE_URL=postgres://drill:local-dev-password@localhost:5432/cloud-local
INGEST_API_KEY=local-dev-ingest-key
CURRENT_EVENTS_POLL_MS=1000
KEYCLOAK_ISSUER_URL=https://sso.drillcloud.ru/realms/drillcloud
KEYCLOAK_CLIENT_ID=drillcloud-ui
KEYCLOAK_EDGE_ROLE_PREFIX=drill-edge-
KEYCLOAK_ADMIN_ROLES=drill-admin,admin
```

Backend при старте последовательно применяет ещё не выполненные SQL-файлы из каталога `migrations`. Ручной запуск одной выбранной миграции не нужен и может оставить схему в промежуточном состоянии.

`KEYCLOAK_AUTH_DISABLED` больше не поддерживается. Все защищённые GET-запросы требуют действующий Keycloak token. `GET /api/health` остаётся публичным, а `POST /api/ingest` использует `x-api-key`, если задан `INGEST_API_KEY`.

## frontend

```dotenv
DEV_API_URL=http://localhost:3101
VITE_KEYCLOAK_URL=https://sso.drillcloud.ru
VITE_KEYCLOAK_REALM=drillcloud
VITE_KEYCLOAK_CLIENT_ID=drillcloud-ui
```

`DEV_API_URL` указывается без `/api`: Vite проксирует браузерные запросы `/api/*` на backend. Переменные `VITE_*` попадают в браузерный bundle и не должны содержать секреты.

## mqtt-ingest

```dotenv
MQTT_URL=mqtt://localhost:1883
HTTP_PORT=8080
WS_PORT=9090
CLOUD_INGEST_URL=http://localhost:3101/api/ingest
DEMO_CLOUD_INGEST_URL=http://localhost:3101/api/ingest
CLOUD_INGEST_API_KEY=local-dev-ingest-key
ARCHIVE_DIR=./video-archive
ARCHIVE_BUCKET_SECONDS=3600
ARCHIVE_UPLOAD_SCAN_SECONDS=300
YANDEX_DISK_TOKEN=
YANDEX_DISK_REMOTE_ROOT=
```

`CLOUD_INGEST_API_KEY` должен совпадать с `INGEST_API_KEY` backend. `ARCHIVE_DIR` оставляет временный видеоархив внутри проекта, а Yandex Disk для локальной проверки не требуется.

Актуальные локальные топики проекта edge5:

- `data/dev/modbus/v3`;
- `data/dev/video/v2/camera-5`;
- `data/dev/video/v2/camera-6`;
- `data/dev/video/v2/camera-7`.

Обработчики `mqtt-ingest` также поддерживают demo-топики `data/demo/modbus/v1`, `data/demo/plc/v1`, `data/demo/video/v1` и `data/demo/video/v2/+`. Топик и обработчик должны соответствовать друг другу: неизвестный топик ingest только залогирует и не отправит в cloud.

## Node-RED edge5

Переменные читаются из `.env` в корне `drill-edge-nodered-setup`:

```dotenv
EDGE_RUNTIME_MODE=dev
DEV_MQTT_BROKER=localhost
DEV_CAMERA_URL1=rtsp://127.0.0.1:8554/video1
DEV_CAMERA_URL2=rtsp://127.0.0.1:8554/video2
DEV_CAMERA_URL3=
```

В режиме `dev` реальные Modbus и камеры выключены. В режиме `prod` включаются реальные источники и production-топики; этот режим нельзя использовать для обычной локальной разработки.

## MQTT monitor

Монитор необязателен для прохождения данных. Он подключается к broker как отдельный подписчик с `QoS 0`, подписывается на `#`, ничего не публикует и не забирает сообщения у ingest.

Минимальная локальная конфигурация:

```dotenv
HTTP_PORT=3205
VITE_PORT=5174
VITE_API_URL=http://localhost:3205
MQTT_URL=mqtt://localhost:1883
IMPORTANT_TOPICS=data/dev/modbus/v3,data/dev/video/v2/+
IMPORTANT_CAMERAS=camera-5,camera-6,camera-7
TOPIC_SILENCE_MS=45000
TZ=Europe/Moscow
TELEGRAM_ENABLED=false
```

Telegram и SOCKS5 относятся к production-развёртыванию монитора и описаны в README его репозитория.
