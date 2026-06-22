# CLAUDE.md

Этот файл содержит инструкции для Claude Code (claude.ai/code) при работе с кодом в этом репозитории.

## Что это

Одностраничный чат-интерфейс без зависимостей (клиент «imsoft chat» / 🌙), который общается с **локальным сервером LMStudio** через его HTTP API, совместимый с OpenAI. Всё приложение состоит из трёх статических файлов — `index.html`, `style.css`, `app.js` — **без шага сборки, без менеджера пакетов, без фреймворков и без тестов**. Текст интерфейса на русском языке.

## Запуск / разработка

Ничего собирать или устанавливать не нужно. Откройте приложение как статические файлы:

```sh
python3 -m http.server 8080   # затем откройте http://localhost:8080
```

**Важный нюанс для локальной разработки:** `API_BASE` в [app.js](app.js) по умолчанию равен `""` (относительный путь). В продакшене приложение обслуживается за Apache по адресу `chat.imsoft.pro`, который проксирует `/v1/...` в LMStudio. При пустом `API_BASE` у обычного статического сервера нет эндпоинта `/v1`, и каждый запрос завершается ошибкой. Для локальной разработки укажите в `API_BASE` напрямую адрес сервера LMStudio (например, `http://192.168.1.57:1234`) и убедитесь, что сервер LMStudio запущен с загруженной моделью.

Используемые эндпоинты LMStudio: `GET /v1/models` (список моделей + опрос доступности) и `POST /v1/chat/completions` (потоковый и непотоковый режимы).

## Архитектура

`app.js` — это единый IIFE в строгом режиме (strict mode). Ключевые архитектурные решения, охватывающие весь файл:

- **Состояние + сохранение.** Всё состояние приложения хранится в одном объекте `state` (`chats`, `activeId`, `settings`) и зеркалируется в `localStorage` под ключами `imsoft.*` (см. карту `LS`). `save()` записывает все три сразу; пополевого сохранения нет. Тема хранится отдельно под `imsoft.theme`. Перезагрузка страницы полностью восстанавливает чаты, активный диалог и настройки.

- **Собственный рендерер markdown (`renderMarkdown`).** Намеренно написан вручную и без зависимостей ради безопасности: сначала экранируется весь HTML, затем заново вводится фиксированный белый список строчных/блочных элементов. Блоки кода в ограждениях (fenced) извлекаются в плейсхолдеры *до* инлайн-разбора, чтобы их содержимое никогда не переинтерпретировалось, а затем вставляются обратно (и снова экранируются) в конце. При изменении рендеринга сохраняйте этот порядок «сначала экранирование» — вывод ассистента является недоверенным текстом модели, вставляемым через `innerHTML`.

- **Потоковая передача (streaming).** Когда `stream` включён, ответный `ReadableStream` читается вручную, буферизуется, разбивается по переводам строк и парсится построчно как SSE-события `data:`; ошибки разбора частичного JSON намеренно игнорируются (чанк может разорваться посередине токена). Единственный `AbortController` (`controller`) обслуживает кнопку Stop; при прерывании добавляется локализованная метка «остановлено», а не выводится ошибка.

- **Модель рендеринга.** Виртуального DOM нет. `renderHistory()` / `renderMessages()` перестраивают свои секции из `state`; во время генерации текущее сообщение ассистента изменяется напрямую через `updateBody()` для каждой дельты. Системные сообщения хранятся в чате, но отфильтровываются из видимой ленты.

- **Статус сервера.** `loadModels()` запускается при инициализации и по `setInterval` каждые 30 секунд, заполняя `<select>` моделей и управляя точкой/текстом статуса соединения.

При редактировании придерживайтесь существующих соглашений: ванильные DOM API через хелперы `$`/`el`, русскоязычные строки интерфейса и темизация CSS через кастомные свойства, определённые в `:root` / `:root.light` в [style.css](style.css) (тёмная тема по умолчанию; класс `light` на `<html>` переключает темы).

---

# CLAUDE.md (English)

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page, dependency-free chat UI (the "imsoft chat" / 🌙 client) that talks to a **local LMStudio server** over its OpenAI-compatible HTTP API. The entire app is three static files — `index.html`, `style.css`, `app.js` — with **no build step, no package manager, no frameworks, and no tests**. The UI text is in Russian.

## Running / developing

There is nothing to build or install. Open the app as static files:

```sh
python3 -m http.server 8080   # then open http://localhost:8080
```

**Critical gotcha for local dev:** `API_BASE` in [app.js](app.js) defaults to `""` (relative path). In production the app is served behind Apache at `chat.imsoft.pro`, which proxies `/v1/...` to LMStudio. With an empty `API_BASE`, a plain static server has no `/v1` endpoint and every request fails. For local development, point `API_BASE` directly at the LMStudio server (e.g. `http://192.168.1.57:1234`) and ensure LMStudio's server is running with a model loaded.

LMStudio endpoints used: `GET /v1/models` (model list + connectivity polling) and `POST /v1/chat/completions` (streaming and non-streaming).

## Architecture

`app.js` is a single IIFE in strict mode. Key design points that span the file:

- **State + persistence.** All app state lives in one `state` object (`chats`, `activeId`, `settings`) and is mirrored to `localStorage` under the `imsoft.*` keys (see the `LS` map). `save()` writes all three at once; there is no per-field persistence. Theme is stored separately under `imsoft.theme`. Reloading the page fully restores chats, the active conversation, and settings.

- **Custom markdown renderer (`renderMarkdown`).** Deliberately hand-rolled and dependency-free for safety: it escapes all HTML first, then re-introduces a fixed whitelist of inline/block elements. Fenced code blocks are extracted to placeholders *before* inline parsing so their contents are never reinterpreted, then re-inserted (and re-escaped) at the end. When changing rendering, preserve this escape-first ordering — assistant output is untrusted model text injected via `innerHTML`.

- **Streaming.** When `stream` is on, the response `ReadableStream` is read manually, buffered, split on newlines, and parsed line-by-line as SSE `data:` events; partial-JSON parse failures are intentionally swallowed (a chunk may split mid-token). A single `AbortController` (`controller`) backs the Stop button; aborts append a localized "stopped" marker rather than surfacing an error.

- **Rendering model.** No virtual DOM. `renderHistory()` / `renderMessages()` rebuild their sections from `state`; during generation the in-progress assistant message is mutated directly via `updateBody()` for each delta. System messages are stored in the chat but filtered out of the visible transcript.

- **Server status.** `loadModels()` runs on init and on a 30s `setInterval`, populating the model `<select>` and driving the connection-status dot/text.

When editing, match the existing conventions: vanilla DOM APIs via the `$`/`el` helpers, Russian user-facing strings, and CSS theming through the custom properties defined in `:root` / `:root.light` in [style.css](style.css) (dark is default; the `light` class on `<html>` switches themes).
