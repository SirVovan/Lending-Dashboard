# Lending-Dashboard — генератор admin-панелей для лендингов

Универсальный инструмент: кладёшь HTML любого лендинга → Claude Code генерирует полноценную admin-панель в стиле сайта.

## Структура

```
Lending-Dashboard/
├── CLAUDE.md          ← инструкция для Claude Code (не трогать)
├── README.md          ← этот файл
├── _template/         ← шаблон для нового проекта
└── projects/          ← все проекты клиентов
    └── druzya-druzey/ ← пример готового проекта
```

## Как добавить новый проект

**Шаг 1.** Скопировать папку `_template/` в `projects/имя-клиента/`

**Шаг 2.** Заменить `example-landing.html` на реальный HTML лендинга клиента

**Шаг 3.** Заполнить `project.md` — название, URL сайта, FTP-доступ, пароль admin

**Шаг 4.** Открыть папку `projects/имя-клиента/` в Claude Code и написать:
```
генерируй
```

Claude прочитает CLAUDE.md из корня, проанализирует HTML и создаст папку `output/` с 5 файлами.

## Что генерируется

| Файл | Назначение |
|------|-----------|
| `output/index.html` | Лендинг с привязкой к content.json |
| `output/admin.html` | WYSIWYG admin-панель с iframe-превью |
| `output/content.json` | Все редактируемые тексты и данные |
| `output/save.php` | Сохранение контента (PHP, авторизация) |
| `output/upload.php` | Загрузка изображений |

## После генерации

1. Заменить `CHANGE_ME_PASSWORD` на реальный пароль в `save.php` и `upload.php`
2. Загрузить содержимое `output/` на хостинг клиента (данные FTP в `project.md`)
3. Отметить статус в `project.md`
