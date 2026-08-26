# Документация как Git submodule

Backend, frontend и mqtt-ingest подключают этот репозиторий в `docs/shared`. Родительский репозиторий хранит не копию файлов, а точный commit документации. Это делает сборку воспроизводимой, но обновление ссылки нужно коммитить отдельно.

## Новое клонирование сервиса

Клонировать сервис вместе с документацией:

```bash
git clone --recurse-submodules <service-repository-url>
```

Если сервис уже склонирован:

```bash
npm run docs:init
```

В `mqtt-ingest` npm-команды выполняются из каталога `app`.

## Обновление документации

После публикации изменений в `drill-cloud-docs/main`:

```bash
npm run docs:update
npm run docs:status
git status
```

`docs:update` переводит `docs/shared` на последний commit ветки `main`. После проверки новый указатель нужно сохранить в сервисе:

```bash
git add docs/shared
git commit -m "docs: update shared documentation"
```

Без этого коммита коллеги продолжат получать прежнюю версию документации — это нормальное свойство submodule.

## Изменение общей документации

Не редактируйте `docs/shared` как обычную папку сервиса: submodule обычно находится в detached HEAD.

Правильный порядок:

1. Изменить документацию в отдельном клоне `drill-cloud-docs`.
2. Создать и объединить Pull Request в `main`.
3. В каждом сервисе выполнить `npm run docs:update`.
4. Закоммитить изменившийся указатель `docs/shared`.

Документация одного сервиса остаётся в его собственном `docs`, но вне `docs/shared`.

## Эквивалентные команды Git

Если npm недоступен:

```bash
git submodule update --init --recursive docs/shared
git submodule update --remote --recursive docs/shared
git submodule status docs/shared
```

В `.gitmodules` задана отслеживаемая ветка `main`. Команда обновления не меняет файлы родительского сервиса, кроме git-ссылки на submodule.
