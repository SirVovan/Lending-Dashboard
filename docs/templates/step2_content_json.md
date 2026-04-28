## §ContentJSON — Структура content.json

```json
{
  "_meta": {
    "generated": "2025-01-15T10:30:00",
    "source": "имя_исходного_файла.html",
    "sections": ["nav", "hero", "stats", "services", "contacts", "footer"]
  },
  "_seo": {
    "page_title": "...",
    "description": "..."
  },
  "nav": {
    "logo_text": "Название",
    "cta_text": "Записаться"
  },
  "hero": {
    "pill_text": "Адрес · Город",
    "title": "Главный заголовок",
    "tagline": "Текст с <strong>акцентом</strong>",
    "cta_text": "Записаться",
    "btn_secondary": "Узнать больше"
  },
  "stats": {
    "items": [
      { "num": "1 900", "label": "довольных гостей" }
    ]
  },
  "services": {
    "eyebrow": "Что мы делаем",
    "title": "Мероприятия на любой <em>вкус</em>",
    "items": [
      { "num": "01", "name": "Свадьбы", "desc": "Описание..." }
    ]
  },
  "faq": {
    "title": "Часто задаваемые вопросы",
    "items": [
      { "question": "Вопрос?", "answer": "Ответ." }
    ]
  },
  "contacts": {
    "address": "ул. Гагарина, 2А, Артёмовский",
    "phone": "+7 909 007 69 33",
    "email": "email@example.ru",
    "social_vk": "https://vk.com/...",
    "social_tg": "https://t.me/...",
    "social_wa": ""
  },
  "footer": {
    "legal_html": "© 2025 Название. ИП Фамилия."
  }
}
```

**Правила заполнения:**
- Все значения берутся из реального текста лендинга (не придумывать)
- HTML-поля сохраняются с тегами: `"Текст с <strong>акцентом</strong>"`
- Поля изображений: текущий путь из `src` атрибута
- Пустые массивы не создавать — только реально существующие элементы
- После заполнения убедиться: нет пустых строк `""` в значимых полях (title, phone, address)
