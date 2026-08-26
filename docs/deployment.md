# Публикация MkDocs

MkDocs преобразует Markdown из `docs` в статический сайт. База данных и серверное приложение для него не нужны.

## Локальный запуск

Windows PowerShell:

```powershell
py -m venv .venv
.\.venv\Scripts\python -m pip install -r requirements.txt
.\.venv\Scripts\python -m mkdocs serve
```

Linux/macOS:

```bash
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements.txt
.venv/bin/python -m mkdocs serve
```

Сайт доступен на `http://127.0.0.1:8000` и автоматически обновляется при изменении Markdown.

## Проверка сборки

```bash
python -m mkdocs build --strict
```

Результат создаётся в `site`. `--strict` завершает сборку с ошибкой при предупреждениях, например при некорректной ссылке или отсутствующей странице навигации.

## GitHub Pages

В репозитории используется workflow `.github/workflows/docs.yml`. Он проверяет документацию в Pull Request, а после push в `main` собирает и публикует сайт.

Однократная настройка на GitHub:

1. Откройте `drill-cloud-docs → Settings → Pages`.
2. В `Build and deployment → Source` выберите `GitHub Actions`.
3. Объедините изменения MkDocs в `main` либо вручную запустите workflow `Docs` во вкладке `Actions`.
4. Дождитесь успешного job `deploy`.

Стандартный адрес проекта:

```text
https://drill-cloud.github.io/drill-cloud-docs/
```

Pages публикует сайт отдельно от исходной ветки: в `main` остаются Markdown и конфигурация, а не сгенерированные HTML-файлы.

## Собственный домен

Для адреса вида `docs.drillcloud.ru`:

1. Добавьте домен в `Settings → Pages → Custom domain`.
2. Создайте DNS-запись `CNAME`, указывающую `docs.drillcloud.ru` на `drill-cloud.github.io`.
3. Включите `Enforce HTTPS` после выпуска сертификата.
4. Замените `site_url` в `mkdocs.yml` на `https://docs.drillcloud.ru/`.

Если документация внутренняя, сначала проверьте политику доступа организации: опубликованный GitHub Pages может быть доступен из интернета даже для закрытого исходного репозитория.

## Развёртывание через Nginx Proxy Manager

Если GitHub Pages не подходит, соберите `site` командой `mkdocs build --strict` и отдавайте каталог любым статическим web-сервером. Например, контейнером Nginx:

```yaml
services:
  docs:
    image: nginx:alpine
    volumes:
      - ./site:/usr/share/nginx/html:ro
    expose:
      - "80"
    networks:
      - proxy

networks:
  proxy:
    external: true
```

В Nginx Proxy Manager укажите имя контейнера `docs` и порт `80`. Для автоматического production-развёртывания сборку следует выполнять в CI, а не вручную на сервере.
