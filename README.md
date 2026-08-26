# Drill Cloud Docs

Единый источник общей документации Drill Cloud. Содержимое доступно в каталоге [docs](docs/index.md) и публикуется как сайт с помощью MkDocs Material.

## Локальный просмотр сайта

Требуется Python 3.11 или новее.

```powershell
py -m venv .venv
.\.venv\Scripts\python -m pip install -r requirements.txt
.\.venv\Scripts\python -m mkdocs serve
```

Откройте `http://127.0.0.1:8000`. Строгая проверка перед MR:

```powershell
.\.venv\Scripts\python -m mkdocs build --strict
```

Инструкция по [сабмодулям](docs/submodules.md) находятся внутри документации.
