# LENDING-DASHBOARD: HTML-ЛЕНДИНГ → ADMIN-ПАНЕЛЬ

Проект превращает HTML-лендинг в редактируемый сайт с WYSIWYG admin-панелью.
Рабочий процесс состоит из двух стадий:

- **СТАДИЯ 1 — Генерация:** анализ HTML-лендинга → создание 5 файлов в `output/`
- **СТАДИЯ 2 — Доработка:** отлов багов генерации + правки и уточнения по запросу

---

## РЕЖИМ РАБОТЫ — читай ПЕРВЫМ

Найди активный проект: `projects/*/`

Если `output/` содержит **все 5 файлов** (content.json, index.html, admin.html, save.php, upload.php):
→ **СТАДИЯ 2** — НЕМЕДЛЕННО перейди к §ДОРАБОТКА. Раздел §ГЕНЕРАЦИЯ — НЕ ЧИТАЙ.

Иначе:
→ **СТАДИЯ 1** — НЕМЕДЛЕННО перейди к §ГЕНЕРАЦИЯ. Раздел §ДОРАБОТКА — НЕ ЧИТАЙ.

---

## §ГЕНЕРАЦИЯ

### Найти HTML-файл

1. В текущей директории (любое имя, кроме `index.html`, `admin.html`, `example-landing.html`)
2. Иначе — в `projects/*/` (первый HTML, не в `output/`, `_template/`, `_archive/`)
3. Если HTML не найден, но в `project.md` есть домен — скачать через `curl`, сохранить как `landing.html`

### Прочитать project.md

Название проекта → `<title>` и `.auth-logo`. URL сайта → кнопка «Открыть сайт».

### Создать файлы

Создай `output/` и `output/images/uploads/`. Генерируй СТРОГО в порядке:

1. `output/content.json`
2. `output/save.php`
3. `output/upload.php`
4. `output/index.html`
5. `output/admin.html`

### Алгоритм анализа HTML

IMPORTANT: читай `docs/ALGORITHM.md` перед ШАГом 1. Краткая суть:

- CSS-переменные из `:root`; шрифты (Google Fonts / Bunny)
- Секции: `section[id]` → `div[id]` с h1/h2 → классы → h2-разбивка → fallback `"main"`
- Элементы: `data-bind` / `data-bind-html` / `data-bind-src` / `data-builder`
- Всегда создавай секцию `"contacts"` (phone, email, address, social_*)

### Шаблоны кода

- content.json — `docs/templates/step2_content_json.md`
- index.html — `docs/templates/step3_index_html.md`
- admin.html — `docs/templates/step4_admin_html.md`
- save.php — `docs/templates/step5_save_php.md`
- upload.php — `docs/templates/step6_upload_php.md`

### Нестандартные ситуации

**Нет CSS-переменных** → fallback: `--bg:#1a1a1a; --text:#ffffff; --accent:#6c63ff; --border:rgba(255,255,255,0.15)`

**`<em>` в заголовках** → `data-bind-html` + `<textarea>` в admin.

**Tailwind/Bootstrap** → опираться на семантические теги и id, игнорировать утилитарные классы.

**PHP/Jinja** → игнорировать `{{ }}`, `<?php ?>`, `{% %}`, брать только статичный текст.

### Финальная проверка

1. В content.json нет пустых строк в title, phone, address, email
2. Каждый `data-bind` в index.html соответствует ключу в content.json
3. Каждый `data-path` в admin.html соответствует ключу в content.json
4. `ADMIN_PASSWORD` одинаков в save.php и upload.php
5. В `renderAllArrays()` — вызов для каждого массива + `resubscribeArrayInputs()` в конце
6. `saveData()` НЕ перезагружает iframe
7. В `SECTION_IDS` прописан маппинг где admin-id ≠ landing-id

IMPORTANT: в конце сообщи пользователю — "Замените `CHANGE_ME_PASSWORD` на реальный пароль в save.php и upload.php"

---

## §ДОРАБОТКА

Стадия 2 — баги генерации (неверный data-bind, сломанный SECTION_IDS, renderArray) и правки по запросу (контент, UI, новые поля).

### Активный проект

`projects/druzya-druzey/output/` — рабочая директория. Прочти `project.md` рядом: FTP, пароль, домен.

### Правило трёх файлов

IMPORTANT: любое изменение контента требует синхронизации всех трёх файлов:

| Файл | Роль |
|---|---|
| `content.json` | источник истины (значения) |
| `index.html` | атрибуты привязки (data-bind/data-builder) |
| `admin.html` | поля редактора (data-path) |

NEVER редактируй только один файл из трёх — контент разойдётся.

### Синтаксис атрибутов

| Атрибут | Применение | Поле в admin |
|---|---|---|
| `data-bind="section.key"` | скалярный текст | `<input>` |
| `data-bind-html="section.key"` | HTML с тегами | `<textarea>` |
| `data-bind-src="section.key"` | src картинки | input + кнопка загрузки |
| `data-builder="section.items"` | массив объектов | `renderArray()` |

### Добавление нового поля

1. `content.json` — добавь ключ в нужную секцию
2. `index.html` — добавь `data-bind="section.key"` на элемент
3. `admin.html` — добавь `<input data-path="section.key">` в панель секции

### SECTION_IDS

Если id секции в admin ≠ id в landing — добавь маппинг в admin.html:

```js
const SECTION_IDS = { 'admin-section-id': 'landing-section-id' };
```

### Деплой

```bash
python deploy.py   # из папки projects/druzya-druzey/
```

Инкрементальный деплой по FTP — загружает только изменённые файлы.

### Накопленные баги (обновляй после каждого исправления)

| Симптом | Причина | Решение |
|---|---|---|
| Клики в iframe не перехватываются | `<iframe src="...">` загружается до навешивания handler | Убрать `src` из тега, задать динамически в `initIframe()` до `onload` |
| Контент не меняется после Save | data-path в admin.html не совпадает с ключом в content.json | Сверить data-path и ключи content.json |
| Массив не отображается | в `renderAllArrays()` нет вызова для data-builder | Добавить вызов + `resubscribeArrayInputs()` в конце |
| Live preview не работает | `onFieldInput()` или `postMessage` сломан | Проверить эти функции в admin.html |
| Save возвращает ошибку | `ADMIN_PASSWORD` в save.php и upload.php не совпадают | Привести к одному значению |

IMPORTANT: после каждого нового бага — добавь строку в эту таблицу.

---

NEVER пересоздавай файлы с нуля, если явно не просят — делай точечные правки.
NEVER меняй структуру HTML в index.html — только атрибуты data-bind/data-builder.
