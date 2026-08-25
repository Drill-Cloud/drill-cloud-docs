# Проверка Drill Cloud

## Быстрый ручной smoke

Перед проверкой должны работать TimescaleDB, cloud, Mosquitto, mqtt-ingest, Node-RED и UI.

1. `GET http://localhost:3101/api/health` возвращает успешный ответ.
2. `http://localhost:5173` перенаправляет в Keycloak и после входа возвращает в локальный UI.
3. Пользователь видит только установки, разрешённые его ролями.
4. Установка `dev` содержит ожидаемые показатели и их постоянные цвета.
5. Значения меняются без перезагрузки страницы.
6. В истории появляется линия выбранного показателя.
7. При включённом видео открывается поток соответствующей камеры.
8. В Console браузера нет повторяющихся ошибок авторизации, API, SSE или WebSocket.

Проверка сборки сервисов перед MR:

```powershell
Set-Location C:\dev\drill\cloud
npm run lint
npm test
npm run build

Set-Location C:\dev\drill\ui
npm run lint
npm run build

Set-Location C:\dev\drill\mqtt-ingest\app
npm run build
```

## Автоматизированный smoke

Полный набор хранится в [drill-cloud-test](https://github.com/Drill-Cloud/drill-cloud-test). Он использует Python, pytest и Playwright, проверяет SSO, роли, список установок, current/SSE, историю, видео и настройки пользователя.

### Подготовка

Требуются Python 3.11+ и Git Bash:

```bash
git clone https://github.com/Drill-Cloud/drill-cloud-test.git
cd drill-cloud-test
bash scripts/bootstrap.sh
cp .env.example .env
```

Минимальный `.env` для локального UI:

```dotenv
E2E_BASE_URL=http://localhost:5173
E2E_API_URL=http://localhost:5173/api
E2E_AUTH_MODE=required
E2E_USERNAME=<test-user>
E2E_PASSWORD=<test-password>
E2E_EDGE_ID=dev
```

Учётные данные тестового пользователя не коммитятся.

### Запуск

```bash
# Обязательный P0
bash scripts/run-smoke.sh

# P0 с видимым браузером
bash scripts/run-smoke.sh --headed

# Полный P0 + P1 + P2
bash scripts/run-smoke.sh --priority all
```

Отчёт создаётся в `reports/smoke-report.html`. При падении проверяйте `test-results`: там находятся Playwright trace, screenshot, video и браузерная диагностика.

### Управляемые E2E-данные

Тестовый проект умеет создавать и удалять только безопасные данные с префиксом `e2e-`:

```dotenv
E2E_DATABASE_URL=postgresql://drill:local-dev-password@localhost:5432/cloud-local
E2E_INGEST_API_KEY=local-dev-ingest-key
E2E_VIDEO_WS_URL=ws://localhost:9090/dev-camera-5
```

```bash
bash scripts/seed-test-data.sh
bash scripts/publish-live-data.sh --duration 300
bash scripts/seed-test-data.sh cleanup
```

Не задавайте seed-скрипту ID реальных установок. Скрипт специально откажется работать с ID без префикса `e2e-`.

Для полной проверки ролей нужны три пользователя:

- администратор с ролью `drill-admin`;
- edge-пользователь с ролью вида `drill-edge-e2e-main`;
- пользователь без ролей Drill Cloud.

Полная матрица сценариев и все дополнительные переменные находятся в README и `docs/` репозитория `drill-cloud-test`; здесь сохранён только общий сценарий запуска системы.
