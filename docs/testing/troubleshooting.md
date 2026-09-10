# Диагностика ошибок автотестов

Начинайте с определения класса проблемы: конфигурация, доступность стенда, тестовые данные, продукт, visual baseline или инфраструктура CI.

## `E2E_BASE_URL is not configured`

Job не получил variable из GitHub Environment.

Проверьте:

1. environment с именем `dev`, `alpha` или `main` существует;
2. variable создана именно в выбранном Environment;
3. имя совпадает полностью и без пробелов;
4. workflow использует `environment: name: ${{ inputs.environment }}`.

Аналогично проверяются `E2E_API_URL`, `E2E_USERNAME` и `E2E_PASSWORD`.

## Ошибка авторизации или не найден заголовок

Если Aria snapshot уже показывает страницу приложения, а тест ждёт старый заголовок, это selector/ожидание теста, а не ошибка логина. Сравните фактический accessible name с Page Object. Если видна страница Keycloak — проверяйте учётные данные, redirect URI, роли и `E2E_AUTH_MODE`.

## Список установок изменился после refresh

Нельзя сравнивать refresh с числом карточек, полученным до завершения начальной загрузки. Тест должен дождаться стабильного результата API/UI и только затем фиксировать count. При падении сохраните network trace и проверьте `/api/edge`.

## Live-тест не видит изменение

Если `E2E_LIVE_TAG` не задан, тест может выбрать первый текущий тег, который не меняется. Задайте управляемый `e2e-live`, запустите publisher и увеличивайте `E2E_LIVE_WAIT_SECONDS` только после проверки фактического потока.

## Video-тест не нашёл буровую

Без `E2E_VIDEO_EDGE_ID`/`E2E_NO_VIDEO_EDGE_ID` тест ищет подходящий edge через API. Если на стенде нет требуемого состояния, это ошибка данных. Подготовьте `e2e-video` и `e2e-no-video` либо задайте существующие ID.

## `SKIPPED`

| Причина | Что заполнить/сделать |
|---|---|
| Нет admin/edge/no-role пары | Добавить соответствующие Environment secrets |
| SSO отключён | Уточнить контракт стенда; для защищённого стенда поставить `E2E_AUTH_MODE=required` |
| Visual disabled локально | `E2E_VISUAL_ENABLED=true` |

Не превращайте skip в pass. Если сценарий обязателен для релиза, сначала подготовьте данные/secret и повторите запуск.

## Visual diff превышает допуск

1. Скачайте `test-results/visual/<browser>`.
2. Сравните baseline, actual и diff.
3. Проверьте viewport, браузер, шрифты, загрузку данных и маски динамических областей.
4. Если UI сломан — исправьте Frontend.
5. Если изменение утверждено — обновите baseline отдельным осознанным коммитом.

Не увеличивайте tolerance только ради зелёного запуска: это снижает ценность проверки.

## `ModuleNotFoundError: yaml`

Зависимости установлены не из актуального `pyproject.toml` либо cache устарел. В проекте `pyyaml>=6,<7` является обычной dependency. Проверьте шаг `pip install -e .`; при необходимости очистите pip cache и перезапустите job.

## ReportPortal пуст, но тесты запускались

Проверьте warning в шаге pytest и четыре значения: `REPORTPORTAL_ENABLED=true`, endpoint, project и API key. Затем проверьте доступность ReportPortal с GitHub runner и права технического пользователя на проект.

## Что скачать при падении

- `reports/e2e-report.html` — сводка pytest;
- `reports/junit.xml` — машинный результат;
- Playwright trace — DOM, действия и network;
- screenshot/video — видимое состояние;
- `browser-diagnostics.txt` — console/page/request errors;
- `live-publisher.log` — если включён publisher.

После определения времени откройте [Grafana](../observability/incident-response.md) и сопоставьте тест с метриками и логами того же контура.
