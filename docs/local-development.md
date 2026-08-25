# Локальный контур edge + cloud + UI

Инструкция поднимает на компьютере разработчика изолированный поток данных:

```text
Node-RED → локальный Mosquitto → локальный mqtt-ingest
                                      ├─ HTTP → локальный cloud → локальная TimescaleDB
                                      └─ WebSocket video → локальный UI

локальный UI ↔ https://sso.drillcloud.ru
```

Общие dev/prod MQTT и базы данных в этом сценарии не используются.

## 1. Требования

- Git;
- Node.js 22 LTS и npm;
- Docker Desktop;
- учётная запись в realm `drillcloud`;
- роль `drill-edge-dev` или одна из административных ролей `drill-admin`, `admin`;
- для видео — FFmpeg и, при работе с файлами, MediaMTX.

В клиенте Keycloak `drillcloud-ui` должны быть разрешены:

- Valid Redirect URI: `http://localhost:5173/*`;
- Web Origin: `http://localhost:5173`.

Если эти значения отсутствуют, их добавляет администратор SSO. Локально отключать авторизацию не требуется.

## 2. Клонирование

В PowerShell:

```powershell
New-Item -ItemType Directory -Force C:\dev\drill | Out-Null
Set-Location C:\dev\drill

git clone https://github.com/Drill-Cloud/drill-cloud-backend.git cloud
git clone https://github.com/Drill-Cloud/drill-cloud-frontend.git ui
git clone https://github.com/Drill-Cloud/drill-mqtt-ingest.git mqtt-ingest
git clone https://github.com/Drill-Cloud/drill-edge-nodered-setup.git drill-edge-nodered-setup
```

Проект потоков [drill-edge-nodered-edge5](https://github.com/Drill-Cloud/drill-edge-nodered-edge5) подключается позже через интерфейс Node-RED и сохраняется внутри `drill-edge-nodered-setup\projects`.

Не вставляйте personal access token в URL репозитория. Используйте Git Credential Manager или SSH.

## 3. TimescaleDB

Создайте отдельную локальную БД:

```powershell
docker run --name drill-timescaledb-local -e POSTGRES_DB=cloud-local -e POSTGRES_USER=drill -e POSTGRES_PASSWORD=local-dev-password -p 5432:5432 -v drill-timescaledb-local:/var/lib/postgresql/data -d timescale/timescaledb:2.28.0-pg17
```

При повторном запуске существующего контейнера:

```powershell
docker start drill-timescaledb-local
```

## 4. Backend cloud

Создайте `C:\dev\drill\cloud\.env`:

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

Установите зависимости и запустите backend:

```powershell
Set-Location C:\dev\drill\cloud
npm install
npm run start:dev
```

При старте backend сам создаёт `schema_migrations` и применяет все отсутствующие миграции, включая TimescaleDB, камеры, nullable current, настройки UI и постоянный цвет тега.

Проверка:

```powershell
Invoke-RestMethod http://localhost:3101/api/health
```

## 5. Минимальные справочники

После успешного старта backend выполните:

```powershell
docker exec -it drill-timescaledb-local psql -U drill -d cloud-local
```

В `psql`:

```sql
INSERT INTO edge (id, name)
VALUES ('dev', 'Локальная установка')
ON CONFLICT (id) DO UPDATE SET name = EXCLUDED.name;

INSERT INTO tag
  (id, name, min, max, comment, unit_of_measurement, precision, tag_group, color)
VALUES
  ('edge5-v3-wk',   'Вес на крюке',       0,   50,  'Эмулятор', 'кг',    1, 'Бурение', '#FACC15'),
  ('edge5-v3-hk',   'Высота крюка',       0,    2,  'Эмулятор', 'м',     2, 'Бурение', '#649DFF'),
  ('edge5-v3-vw',   'Скорость ветра',     0,    3,  'Эмулятор', 'м/с',   2, 'Среда',   '#6EE7B7'),
  ('edge5-v3-pdk',  'Концентрация газа',  0,    1,  'Эмулятор', 'ПДК',   2, 'Среда',   '#FB7185'),
  ('edge5-v3-h2s',  'Сероводород',        0,    1,  'Эмулятор', 'ПДК',   2, 'Среда',   '#A78BFA'),
  ('edge5-v3-nrot', 'Обороты ротора',     0,  150,  'Эмулятор', 'об/мин',0, 'Бурение', '#F97316'),
  ('edge5-v3-vsp',  'Скорость подачи',    0,    1,  'Эмулятор', 'м/с',   2, 'Бурение', '#22D3EE')
ON CONFLICT (id) DO UPDATE SET
  name = EXCLUDED.name,
  min = EXCLUDED.min,
  max = EXCLUDED.max,
  comment = EXCLUDED.comment,
  unit_of_measurement = EXCLUDED.unit_of_measurement,
  precision = EXCLUDED.precision,
  tag_group = EXCLUDED.tag_group,
  color = EXCLUDED.color;

INSERT INTO camera (edge, name, protocol, source)
VALUES
  ('dev', 'Камера 5', 'ws', 'localhost:9090/dev-camera-5'),
  ('dev', 'Камера 6', 'ws', 'localhost:9090/dev-camera-6'),
  ('dev', 'Камера 7', 'ws', 'localhost:9090/dev-camera-7')
ON CONFLICT (edge, protocol, source) DO UPDATE SET name = EXCLUDED.name;
```

Выйдите командой `\q`.

Цвет хранится в `tag.color` в формате `#RRGGBB`. Он одинаков для линии графика и легенды и не вычисляется UI динамически.

## 6. Локальный MQTT broker

В репозитории `mqtt-ingest` уже есть Mosquitto. Запустите только broker:

```powershell
Set-Location C:\dev\drill\mqtt-ingest
docker compose up -d mosquitto
docker compose ps
```

Broker должен быть доступен по адресу `mqtt://localhost:1883`.

## 7. mqtt-ingest

Создайте `C:\dev\drill\mqtt-ingest\app\.env`:

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

Запуск:

```powershell
Set-Location C:\dev\drill\mqtt-ingest\app
npm install
npm run dev
```

В логе должно появиться подключение к `mqtt://localhost:1883`.

## 8. Node-RED

В `C:\dev\drill\drill-edge-nodered-setup\.env` укажите:

```dotenv
EDGE_RUNTIME_MODE=dev
DEV_MQTT_BROKER=localhost
DEV_CAMERA_URL1=rtsp://127.0.0.1:8554/video1
DEV_CAMERA_URL2=rtsp://127.0.0.1:8554/video2
DEV_CAMERA_URL3=
```

Запустите runtime:

```powershell
Set-Location C:\dev\drill\drill-edge-nodered-setup
npm install
npm start
```

Откройте `http://localhost:1880`. В меню Node-RED выберите `Projects → New → Clone repository` и укажите `https://github.com/Drill-Cloud/drill-edge-nodered-edge5.git`. При запросе credential secret создайте отдельное локальное значение и не сохраняйте его в документации.

Если зависимости проекта ещё не установлены, остановите Node-RED и выполните:

```powershell
Set-Location C:\dev\drill\drill-edge-nodered-setup\projects\drill-edge-nodered-edge5
npm install
```

После этого снова запустите Node-RED из каталога `drill-edge-nodered-setup`.

Убедитесь, что активен режим `dev`, и нажмите `Deploy`. Эмулятор Modbus публикует объект в `data/dev/modbus/v3`; ingest сохраняет его с `edge=dev`.

Если Node-RED сообщает об отсутствующих узлах, откройте `Projects → Project settings → Dependencies` и установите зависимости проекта.

## 9. Frontend

Создайте `C:\dev\drill\ui\.env`:

```dotenv
DEV_API_URL=http://localhost:3101
VITE_KEYCLOAK_URL=https://sso.drillcloud.ru
VITE_KEYCLOAK_REALM=drillcloud
VITE_KEYCLOAK_CLIENT_ID=drillcloud-ui
```

Запуск:

```powershell
Set-Location C:\dev\drill\ui
npm install
npm run dev
```

Откройте `http://localhost:5173`, войдите через серверный SSO и выберите `Локальная установка`.

## 10. Проверка потока

### Без Node-RED

Эта команда проверяет цепочку HTTP ingest → cloud → DB:

```powershell
$body = @{
  edge = 'dev'
  tag = 'edge5-v3-wk'
  value = 25.4
  timestamp = (Get-Date).ToUniversalTime().ToString('o')
} | ConvertTo-Json

Invoke-RestMethod `
  -Method Post `
  -Uri http://localhost:3101/api/ingest `
  -Headers @{ 'x-api-key' = 'local-dev-ingest-key' } `
  -ContentType 'application/json' `
  -Body $body
```

### Полная цепочка

1. В Node-RED узел MQTT показывает подключение к `localhost:1883`.
2. В логе mqtt-ingest появляются `dev.modbus.v3.received` и `dev.modbus.v3.posted`.
3. В UI у локальной установки отображаются семь показателей.
4. Значения обновляются, а история появляется на графике.

Если ingest получает `value=null`, backend обновляет `current.value`, но не добавляет такую точку в `history`. Нулевое значение `0` остаётся обычным числом.

Проверка непосредственно в БД:

```sql
SELECT edge, tag, value, "updatedAt"
FROM current
WHERE edge = 'dev'
ORDER BY tag;

SELECT edge, tag, value, "timestamp"
FROM history
WHERE edge = 'dev'
ORDER BY "timestamp" DESC
LIMIT 20;
```

## 11. Видео

Для проверки реальных видеофайлов запустите MediaMTX и отправьте один или несколько файлов в RTSP:

```powershell
ffmpeg -re -stream_loop -1 -i C:\video\video1.mp4 -an -c:v libx264 -preset veryfast -tune zerolatency -f rtsp rtsp://127.0.0.1:8554/video1
```

Node-RED читает `DEV_CAMERA_URL1`, публикует MPEG-TS чанки в `data/dev/video/v2/camera-5`, а ingest транслирует их в `ws://localhost:9090/dev-camera-5`.

Для второй и третьей камер повторите команду с другими файлами и адресами `video2`, `video3`, затем заполните `DEV_CAMERA_URL2`, `DEV_CAMERA_URL3`.

## 12. Необязательный MQTT monitor

Монитор нужен для наблюдения, но не участвует в доставке данных:

```powershell
Set-Location C:\dev\drill
git clone https://github.com/Drill-Cloud/drill-mqtt-ingest-monitor.git mqtt-monitor
Set-Location mqtt-monitor
Copy-Item .env.example .env
```

Для локального broker измените `MQTT_URL=mqtt://localhost:1883`, оставьте `TELEGRAM_ENABLED=false`, затем:

```powershell
npm install
npm run dev
```

Интерфейс будет доступен на `http://localhost:5174`.

## 13. Остановка

Остановите процессы npm сочетанием `Ctrl+C`, затем контейнеры:

```powershell
Set-Location C:\dev\drill\mqtt-ingest
docker compose stop mosquitto
docker stop drill-timescaledb-local
```

Команды сохраняют данные. Удаляйте контейнер и volume только если локальные данные точно больше не нужны.

## 14. Типовые проблемы

### UI возвращает 401/403

- `401` — token отсутствует, истёк либо не прошёл проверку issuer/client.
- `403` — пользователь авторизован, но не имеет роли `drill-edge-dev` или административной роли.
- Проверьте redirect URI и Web Origin клиента Keycloak.

### Vite proxy отвечает ECONNREFUSED

Backend не запущен либо `DEV_API_URL` содержит неверный порт. Значение должно быть `http://localhost:3101` без `/api`.

### Backend не стартует

- Проверьте `docker ps` и доступность порта `5432`.
- Используйте образ TimescaleDB, а не обычный PostgreSQL.
- Посмотрите ошибку конкретной миграции в логе backend.
- Не применяйте миграции выборочно вручную.

### MQTT-сообщения есть, а cloud пуст

- Node-RED: `DEV_MQTT_BROKER=localhost`.
- mqtt-ingest: `MQTT_URL=mqtt://localhost:1883`.
- Для текущего проекта edge5 ожидается `data/dev/modbus/v3`.
- `CLOUD_INGEST_API_KEY` ingest должен совпадать с `INGEST_API_KEY` backend.
- В БД должны существовать `edge=dev` и соответствующие строки `tag`.

### Камера есть в БД, но видео нет

- Проверьте поступление чанков в `data/dev/video/v2/camera-5`.
- Проверьте лог mqtt-ingest и доступность `ws://localhost:9090/dev-camera-5`.
- Камера в таблице должна иметь `protocol='ws'`, непустое `name` и правильный `source`.
- FFmpeg должен быть доступен из процесса Node-RED.

### Порты заняты

Проверьте процессы на портах `1880`, `1883`, `3101`, `5173`, `5432`, `8080`, `9090`, а для монитора — `3205`, `5174`.
