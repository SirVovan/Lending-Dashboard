# База накопленных багов и исправлений

Каждый баг, найденный и исправленный в процессе генерации или доработки — фиксируется здесь.
При создании новой админки — читать этот файл и закладывать исправления заранее в шаблоны.

---

## Формат записи

**Симптом** — что видит пользователь.  
**Причина** — почему это происходит технически.  
**Решение** — что именно изменить и где.  
**Шаблон** — в каком шаблоне (`step3_index_html.md`, `step4_admin_html.md` и т.д.) нужно учесть при следующей генерации.

---

## Баги iframe / клики

### BUG-001: iframe не перехватывает клики — src до onload

**Проект:** любой  
**Симптом:** клики по iframe не регистрируются, секция не переключается  
**Причина:** `<iframe src="...">` загружается ДО навешивания обработчика mousedown  
**Решение:** убрать `src` из HTML-тега, задать динамически в `initIframe()` через `iframe.src = '...'` внутри функции до установки `onload`  
**Шаблон:** step4_admin_html.md

---

### BUG-002: canvas поверх страницы блокирует определение секции по клику

**Проект:** druzya-druzey (2026-04-28)  
**Симптом:** секции "Три шага..." (how) и "Отзывы" (reviews) не выделяются при клике в пустую область — `e.target` всегда возвращает `canvas#bgCanvas`  
**Причина:** canvas с анимацией частиц растянут на весь экран поверх контента; при клике в зону без `data-bind` элементов (`e.target = canvas`) старый fallback шёл вверх по DOM от canvas до body, не находил совпадения с `SECTION_IDS` и останавливался  
**Решение:**  
1. В `attachIframeClickHandler()` поднять вызов `elementsFromPoint` выше `if`-блока, передать результат в `handleIframeClick(target, iDoc, elems)`  
2. В `handleIframeClick` в конце fallback-блока объединить `elemsAtPoint` + ancestor walk от `target` в массив `candidateEls`, перебрать его по `id` и сопоставить с `SECTION_IDS` / nav-items  
**Шаблон:** step4_admin_html.md — функции `attachIframeClickHandler` и `handleIframeClick`

---

## Баги контента / data-bind

### BUG-003: Логотип в nav частично не привязан

**Проект:** druzya-druzey (2026-04-28)  
**Симптом:** редактируется только второе слово логотипа ("друзей"), первое ("Друзья") — хардкод  
**Причина:** шаблон генерировал `Друзья <span data-bind="nav.logo_text">друзей</span>` — привязка только на `<span>`, а не на весь элемент  
**Решение:**  
- Убрать `nav.logo_text` из content.json  
- Добавить `hero.brand_name` (полное название)  
- Привязать `data-bind="hero.brand_name"` на весь `<a class="nav-logo">` и на `<div class="footer-logo">`  
- В admin: поле в секции Hero, в секции Nav — информационная заметка "редактируется в Герое"  
**Шаблон:** step3_index_html.md, step4_admin_html.md

---

### BUG-004: Ссылка социальной сети не привязана (href)

**Проект:** druzya-druzey (2026-04-28)  
**Симптом:** MAX-ссылка в контактах хардкодирована, не редактируется через админку; в contacts был пустой `social_wa` вместо реального `social_max`  
**Причина:** система привязки не имела типа `data-bind-href` для атрибута `href`; генератор добавил WhatsApp-заглушку вместо реального мессенджера  
**Решение:**  
- Добавить `data-bind-href` как новый тип привязки  
- В `applyContent()` (index.html): обработка `[data-bind-href]` → `el.href = val`  
- В `updateIframeField()` (admin.html): `querySelectorAll('[data-bind-href="'+path+'"]').forEach(el => el.href = val)`  
- В `cancelData()` (admin.html): восстановление href по снимку  
- В content.json: `contacts.social_max` вместо `contacts.social_wa`  
**Шаблон:** step3_index_html.md, step4_admin_html.md — добавить `data-bind-href` в базовую систему привязок

---

## Баги деплоя

### BUG-005: deploy.py перезаписывает боевой пароль на placeholder

**Проект:** druzya-druzey (2026-04-28)  
**Симптом:** после деплоя пароль на сервере сбивается на `CHANGE_ME_PASSWORD`  
**Причина:** deploy.py сравнивает файлы по хешу. Локальный `save.php` содержит placeholder — он отличается от серверного (с боевым паролем) → файл перезаписывается  
**Решение:** всегда держать локальный `save.php` и `upload.php` синхронизированными с боевым паролем. После смены пароля — менять его в обоих файлах локально, потом деплоить  
**Шаблон:** —, организационная проблема

---

## Баги массивов

### BUG-006: Массив не отображается в preview

**Проект:** любой  
**Симптом:** секция с `data-builder` пустая в iframe  
**Причина:** в `renderAllArrays()` нет вызова для конкретного массива, или пропущен `resubscribeArrayInputs()` в конце  
**Решение:** добавить вызов `renderArray('section.items', ...)` в `renderAllArrays()` + убедиться что `resubscribeArrayInputs()` вызывается последним  
**Шаблон:** step4_admin_html.md

---

## Прочие баги

### BUG-007: Save возвращает 403

**Проект:** любой  
**Симптом:** после нажатия Save — ошибка авторизации  
**Причина:** `ADMIN_PASSWORD` в `save.php` и `upload.php` не совпадают  
**Решение:** привести к одному значению в обоих файлах  
**Шаблон:** step5_save_php.md, step6_upload_php.md

---

### BUG-008: Live preview не обновляется при вводе

**Проект:** любой  
**Симптом:** вводишь текст в поле — iframe не меняется  
**Причина:** `onFieldInput()` или `postMessage` сломан  
**Решение:** проверить цепочку: input event → `onFieldInput()` → `postMessage({type:'update', path, val})` → listener в iframe → `updateIframeField()`  
**Шаблон:** step4_admin_html.md

---

### BUG-009: Секции без data-builder не подсвечиваются рамкой при клике в пустую зону

**Проект:** druzya-druzey (2026-04-28)  
**Симптом:** секции "Три шага" (how) и "Отзывы" (reviews) не показывают золотую рамку при клике в область без `data-bind`  
**Причина:** fallback-ветка `handleIframeClick` вызывала только `switchFieldsOnly()` (переключение правой панели), но не добавляла `data-admin-focus` на элемент в iframe  
**Решение:** в fallback-ветке перед `switchFieldsOnly()` добавить `clearIframeHighlights(iDoc)` + `sectionEl.setAttribute('data-admin-focus', '')`  
**Шаблон:** step4_admin_html.md — функция `handleIframeClick`, конец fallback-блока

---

### BUG-010: Логотип сайта в сайдбаре админки не синхронизируется с полем hero.brand_name

**Проект:** druzya-druzey (2026-04-28)  
**Симптом:** при изменении названия бренда в поле "Герой → Название" логотип в сайдбаре остаётся прежним  
**Причина:** текст в `.sidebar-logo` хардкодирован, не имеет id/data-path  
**Решение:**

1. Обернуть текст логотипа в `<span id="adminSidebarLogo">`
2. В `fillForm()` инициализировать из `_data.hero.brand_name`
3. В `updateIframeField()` при `path === 'hero.brand_name'` обновлять span

**Шаблон:** step4_admin_html.md — sidebar-logo + fillForm() + updateIframeField()

---

### BUG-011: Cookie-banner перекрывает футер в preview-iframe

**Проект:** druzya-druzey (2026-04-28)  
**Симптом:** при просмотре секции "Подвал" в admin.html плашка cookie загораживает footer  
**Причина:** cookie IIFE в index.html проверяет только localStorage; внутри iframe localStorage изолирован → баннер всегда показывается  
**Решение:** добавить в начало IIFE проверку `window !== window.top` — при открытии в iframe скрывать баннер немедленно  
**Шаблон:** step3_index_html.md — IIFE cookie banner

---

### BUG-012: Активный пункт сайдбара "прыгает" сразу после клика на раздел

**Проект:** druzya-druzey (2026-04-28)  
**Симптом:** кликаешь на раздел "Отзывы" — подсветка переключается на "Афиша" или другой раздел  
**Причина:** `showSection()` начинает плавный скролл iframe; scroll-событие (debounce 80мс) успевает сработать до завершения анимации; `updateActiveNavFromScroll()` пересчитывает ближайшую секцию по промежуточной позиции скролла  
**Решение:** ввести флаг `_scrollLocked`; в `showSection()` блокировать `updateActiveNavFromScroll` на 1200мс после клика  
**Шаблон:** step4_admin_html.md — showSection(), scroll handler в attachIframeClickHandler()

---

### BUG-013: data-builder секции не позволяют кликнуть на отдельное поле

**Проект:** druzya-druzey (2026-04-29)  
**Симптом:** В секциях с `data-builder` (Услуги, Зачем выбирать нас, Афиша) клик по конкретному тексту в карточке выделяет весь блок золотой рамкой, но не переходит к конкретному input-полю в правой панели  
**Причина:** Builder-функции (`build_services_items`, `build_whyus_items`, `build_events_items`) в index.html генерировали HTML-строки **без** `data-bind`/`data-bind-src` атрибутов. При загрузке страницы `applyContent()` пересоздавала все карточки через эти функции — любые `data-bind` из оригинального HTML уничтожались. В итоге `elementsFromPoint` не находил ни одного `data-bind` в точке клика → `handleIframeClick` брал только `data-builder`-контейнер (весь блок), игнорируя конкретное поле.  
**Решение:**

1. Добавить параметр `idx` во все builder-функции (`function(item, idx) { idx = idx !== undefined ? idx : 0; ... }`)
2. Включить `data-bind="section.items[idx].key"` и `data-bind-src="section.items[idx].img_url"` в HTML-строки внутри каждой функции
3. В `applyContent()` (index.html) изменить `forEach(function(item)` на `forEach(function(item, idx)` и передавать `idx` в `buildFn(item, idx)`
4. В `updateIframeField()` (admin.html) аналогично передавать `idx` в forEach

**Шаблон:** step3_index_html.md — builder-функции ОБЯЗАНЫ принимать `idx` и включать `data-bind`/`data-bind-src` с форматом `section.items[idx].key`; `applyContent` и `updateIframeField` ОБЯЗАНЫ передавать `idx` в forEach

---

### BUG-014: Правая панель не прокручивает к нужному полю при клике в центре

**Проект:** druzya-druzey (2026-04-29)  
**Симптом:** кликаешь на конкретный элемент в центральном iframe — правая панель не прокручивается, соответствующее поле остаётся скрытым за нижней границей  
**Причина:** три проблемы одновременно:

1. `scrollPanelToTop()` скроллила только если элемент **не виден полностью** (`if (!elFullyVisible || !elFitsInPanel)`). Если поле частично попадало в зону — скролла не было.
2. `highlightField()` передавала в `scrollPanelToTop` целый `.field-group` (а не конкретный field). Группа начиналась с group-title, а нужное поле могло быть 3-м или 4-м внутри неё — всё равно за экраном.
3. `switchFieldsOnly()` не сбрасывал `panel.scrollTop = 0` при переключении секции. Если предыдущая секция была прокручена вниз — новая открывалась с тем же смещением.

**Решение:**

1. В `scrollPanelToTop()` убрать условие — всегда вызывать `panel.scrollTo({ top: newScroll, behavior: 'smooth' })`
2. В `highlightField()` найти предшествующий `.field-label` и скроллить к нему (чтобы лейбл был виден): `var label = field.previousElementSibling; var scrollTarget = (label && label.classList.contains('field-label')) ? label : field;`
3. В `switchFieldsOnly()` добавить `if (panel) panel.scrollTop = 0` перед активацией новой секции  
**Шаблон:** step4_admin_html.md — функции `scrollPanelToTop`, `highlightField`, `switchFieldsOnly`

---

### BUG-015: Клик по img с data-bind-src внутри data-builder не фокусирует поле

**Проект:** druzya-druzey (2026-04-29)  
**Симптом:** клик по фотографии в карточке (services, whyus) открывает раздел целиком, но правая панель не прокручивается к конкретному dz-zone и не выделяет его  
**Причина:** в `handleIframeClick` `data-builder`-ветка перехватывала клик раньше, чем ветка `data-bind-src`. Путь возвращался как `services.items` вместо `services.items[0].img_url` → `focusFieldByPath` не мог найти нужный скрытый input  
**Решение:**

1. В `handleIframeClick`: ветка `data-bind`/`data-bind-html`/`data-bind-src` стоит первой (без изменений). После нахождения пути — дополнительно проверить есть ли `data-builder`-предок и пересчитать путь с правильным индексом: `arrPath + '[' + idx + '].' + path.split('.').pop()`
2. В `focusFieldByPath`: добавить ветку для `input[type=hidden]` — найти `.dz-zone` через `.closest()` и вызвать `highlightDzZone(dz)` вместо `highlightField()`
3. Добавить функцию `highlightDzZone(dz)` — выделяет зону загрузки золотой рамкой, скроллит к ней  
**Шаблон:** step4_admin_html.md — функции `handleIframeClick`, `focusFieldByPath`, добавить `highlightDzZone`

---

### BUG-016: Поля загрузки изображений (img_url) в admin.html — нет upload UI для нетехнических пользователей

**Проект:** druzya-druzey (2026-04-29)  
**Симптом:** поля `img_url` в массивах services.items и whyus.items отображались как обычный `<input type="text">` с сырым путём к файлу. Пользователь не мог понять, что туда вводить  
**Причина:** шаблон `step4_admin_html.md` генерировал `img_url` как текстовое поле без UI загрузки  
**Решение:**

1. Добавить флаг `image:true` в схему `buildItem()` для полей `img_url`
2. В `buildItem()` при `f.image` — рендерить `.dz-zone` вместо `<input type="text">`: зона drag&drop с превью, скрытым file input и `input[type=hidden][data-path=...]`
3. После `div.innerHTML = html` — инициализировать зону через `initDzZone(zone, dpPath, currentVal)`
4. `initDzZone(zone, dpPath, currentVal)` — привязывает click/dragover/drop/change, при успехе загрузки обновляет превью, hidden input, `_data` и iframe через `updateIframeField`
5. `dzUpload(file, onSuccess)` — POST на `upload.php` с `file` + `_auth`, при `res.ok` вызывает `onSuccess(res.path)`
6. Добавить CSS `.dz-zone`, `.dz-empty`, `.dz-filled`, `.dz-filled-overlay`  
**Шаблон:** step4_admin_html.md — CSS + `buildItem()` + `initDzZone()` + `dzUpload()` добавить в шаблон; `img_url` поля ВСЕГДА генерировать с `image:true`

---

### BUG-017: `focusFieldByPath` не вызывал `setActiveItem` — фиолетовый фокус не обновлялся при клике в iframe

**Проект:** druzya-druzey (2026-04-29)  
**Симптом:** Клик по полю или карточке в центральном iframe прокручивает правую панель к нужному input и выделяет его золотом (`highlightField`), но `.array-item--active` рамка карточки в правой панели и фиолетовый outline в iframe (`data-builder-highlight`) остаются на предыдущей карточке — не обновляются.  
**Причина:** `focusFieldByPath` вызывается из `handleIframeClick` при клике в iframe. Она вызывает `highlightField`, но НЕ вызывает `setActiveItem`/`clearActiveItem`. Эти функции вызывались только из `onFieldFocus` (нативный `focus`-event при прямом клике по `<input>` в правой панели). Клик в iframe нативного focus-события на правой панели не генерирует.  
**Решение:** В начале `focusFieldByPath` (до `requestAnimationFrame`) добавить:

```js
var _fma = path.match(/^([\w.]+)\[(\d+)\]/);
if (_fma) setActiveItem(_fma[1], parseInt(_fma[2]));
else clearActiveItem();
```

Дополнительно: в RAF-ветке, когда path содержит `[idx]` но конкретный input не найден, скроллить к `.array-item` с этим индексом, а не к обёртке всего массива.  
**Шаблон:** step4_admin_html.md — функция `focusFieldByPath`

---

### BUG-018: `handleIframeClick` теряет индекс карточки из-за `elementsFromPoint` override

**Проект:** druzya-druzey (2026-04-29)  
**Симптом:** Клик по пустому месту внутри карточки (паддинг `.event-top`, `.event-card`, пространство рядом с полем) выделяет весь `data-builder` блок (золотая рамка на весь `.events-grid`), а не конкретную карточку.  
**Причина:** В mousedown-обработчике `target` перезаписывается на первый элемент с `data-builder`/`data-bind` из `elementsFromPoint`. При клике в паддинг карточки первым matching-элементом оказывается `.events-grid` (сам контейнер). В `handleIframeClick` поиск прямого потомка начинался с `bTarget = target = .events-grid = node` → первый шаг `while (bTarget.parentNode !== node)` уходил вверх мимо контейнера → `bIdx = -1` → весь блок подсвечивался.  
**Решение:** Перебирать весь массив `elemsAtPoint` (все элементы в точке клика, от переднего к заднему) и для каждого искать прямого предка `node`:

```js
var bIdx = -1;
var bSearch = (elemsAtPoint && elemsAtPoint.length) ? elemsAtPoint : [target];
for (var bi = 0; bi < bSearch.length && bIdx < 0; bi++) {
  var bwalk = bSearch[bi];
  while (bwalk && bwalk !== node && bwalk.parentNode !== node) bwalk = bwalk.parentNode;
  if (bwalk && bwalk !== node && bwalk.parentNode === node) bIdx = bChildren.indexOf(bwalk);
}
```

Карточка находится корректно, даже если клик попал на паддинг. Только при клике строго на зазор между карточками (фон самого grid) `bIdx` остаётся `-1` — это ожидаемое поведение.  
**Шаблон:** step4_admin_html.md — функция `handleIframeClick`, ветка `data-builder`

---

### BUG-019: deploy.py перезаписывает content.json, затирая правки из admin-панели

**Проект:** druzya-druzey (2026-04-29)  
**Симптом:** изменения, внесённые через admin-панель и сохранённые через save.php, пропадают после следующего деплоя — на сервере снова появляется исходный контент.  
**Причина:** deploy.py загружал `content.json` из локальной папки `output/` в общем списке `FILES`. Локальный файл не знает о правках на сервере → перезаписывал их.  
**Решение:** убрать `content.json` из `FILES`. Добавить отдельную функцию `upload_content_if_missing()`, которая через `ftp.size()` проверяет наличие файла на сервере — загружает только если его там нет (первичный деплой на чистый сервер).  
**Шаблон:** при генерации deploy.py для любого проекта — НИКОГДА не включать `content.json` в общий список загружаемых файлов. Всегда использовать условную загрузку.

---

### BUG-020: Порядок полей в карточке массива не совпадает с превью

**Проект:** druzya-druzey (2026-04-30)  
**Симптом:** В секции «Услуги» правая панель показывает поля в порядке: Номер → Название → Описание → Фото. В превью карточка выглядит иначе: Фото стоит первым. Вносит путаницу при редактировании.  
**Причина:** Порядок ключей в схеме `renderAllArrays()` не совпадал с визуальным порядком элементов в карточке landing-а.  
**Решение:** В `renderAllArrays()` переставить `img_url` первым в схеме `services.items`. Аналогично исправить порядок в кнопке `addItem`.  
**Шаблон:** step4_admin_html.md — порядок ключей в схемах `renderArray` ДОЛЖЕН совпадать с визуальным порядком элементов в карточке на лендинге.

---

### BUG-021: Стрелки перемещения карточек (▲▼) визуально неразличимы

**Проект:** druzya-druzey (2026-04-30)  
**Симптом:** Кнопки ↑↓ перемещения элементов массива почти невидимы — мелкие, бесцветные, сливаются с фоном.  
**Причина:** `.btn-move` имел `color: var(--muted)` и `background: none` — нет визуального веса.  
**Решение:** Сделать `.btn-move` заметными: фиолетовый фон `rgba(112,64,200,0.25)`, граница `rgba(180,140,255,0.35)`, цвет `var(--purple-lt)`, размер `.95rem`. При hover — золотой акцент. Символы заменить на ▲▼.  
**Шаблон:** step4_admin_html.md — CSS `.btn-move` обновить по умолчанию.

---

### BUG-022: Правая панель редактирования визуально блёклая

**Проект:** druzya-druzey (2026-04-30)  
**Симптом:** Правая панель выглядит однородно и тускло — нет чёткого разделения блоков, поля почти не видны на фоне.  
**Причина:** `.field-group` имел слабый фон и границу без акцента. `.array-item` — тёмный без структуры. Заголовки мелкие.  
**Решение:**  
1. `.field-group` — `border-left: 3px solid var(--purple-dim)` + hover светлее + заголовок с градиентной линией  
2. `.array-item` — отдельный тёмный header (`rgba(112,64,200,0.18)`) + `.array-item-body` обёртка для полей  
3. `.field-input` — фон `rgba(255,255,255,0.07)` вместо 0.05, граница ярче  
4. `.btn-add` — пунктирная граница с фиолетовым акцентом  
5. `border-left` у `#fieldsPanel` — `2px solid rgba(180,140,255,0.35)`  
**Шаблон:** step4_admin_html.md — CSS правой панели обновить целиком.

---

### BUG-023: Клик на «Навигация»/«Подвал» в сайдбаре не скроллит iframe

**Проект:** druzya-druzey (2026-04-30)  
**Симптом:** Клик на «Навигация» — iframe не прокручивается вверх. Клик на «Подвал» — не прокручивается вниз. Остальные секции скроллятся нормально.  
**Причина:** `SECTION_IDS` содержал только `{ 'nav': 'mainNav' }`. Для `footer` маппинга не было. При `scrollIframeTo('footer')` — `getElementById('footer')` возвращал null, fallback не был предусмотрен. Для `nav` — `scrollIntoView` на `#mainNav` не давал видимого эффекта т.к. элемент уже вверху страницы, скролл не происходит.  
**Решение:**  
1. Для `nav` — явно обнулять `doc.documentElement.scrollTop = 0` и `doc.body.scrollTop = 0` вместо `scrollIntoView`  
2. Для `footer` — fallback `doc.querySelector('footer')` если `getElementById` вернул null  
3. Подсветка — использовать `data-admin-focus` (тот же CSS что при клике мышью), не inline style  
**Шаблон:** step4_admin_html.md — `SECTION_IDS` добавить `footer`, `scrollIframeTo` добавить специальную обработку nav/footer.

---

### BUG-024: При переключении секции в сайдбаре не видно куда скроллится сайт

**Проект:** druzya-druzey (2026-04-30)  
**Симптом:** Кликаешь на раздел слева — сайт в центре ездит, но непонятно, к какой секции он перешёл.  
**Причина:** `scrollIframeTo` только скроллил, но не подсвечивал целевую секцию. Подсветка `data-admin-focus` работала только при клике мышью в iframe (через `handleIframeClick`).  
**Решение:** В `scrollIframeTo` после скролла добавить `el.setAttribute('data-admin-focus', '')` — тот же CSS-атрибут что при ручном клике. Подсветка остаётся (не снимается автоматически) — снимается при следующем переключении через `clearIframeHighlights`.  
**Шаблон:** step4_admin_html.md — `scrollIframeTo` добавить `data-admin-focus` после скролла.

---

### BUG-025: Вкладка «Главная» — пустой центр и бесполезная кнопка

**Проект:** druzya-druzey (2026-04-30)  
**Симптом:** При открытии admin.html центральная область на вкладке «Главная» — чёрный экран с одной золотой кнопкой «Открыть сайт». Непонятно что делать, нет навигации к разделам.  
**Причина:** `#previewHome` содержал только приветственный текст и одну ссылку — заглушка из шаблона.  
**Решение:** Заменить центральную область на дашборд с карточками-ярлыками всех разделов (сетка 3×3): клик на карточку = переход в раздел (аналог сайдбара). Разделы сгруппированы: «Контент» и «Настройки». Кнопка «Открыть сайт ↗» перенесена в правый верхний угол как вспомогательная. Название бренда из `hero.brand_name` синхронизируется в заголовок дашборда.  
**Шаблон:** step4_admin_html.md — `#previewHome` заменить на дашборд с `.dash-grid` и `.dash-card` по умолчанию.

---

### BUG-026: Изменения в админке не видны без ручного сброса кэша

**Проект:** любой  
**Симптом:** после сохранения в админке посетители видят старую версию сайта — изменения появляются только после Ctrl+Shift+R  
**Причина:** браузер кэширует `content.json`. Так как URL файла не менялся, браузер отдавал кэш даже при наличии новой версии на сервере  
**Решение:**  
1. `save.php` — после успешного `rename()` писать `file_put_contents(__DIR__ . '/version.txt', time())`  
2. `index.html` — вместо прямого `fetch('./content.json')` сначала брать `version.txt` с `{ cache: 'no-store' }` (крошечный файл, 10 байт), затем загружать `content.json?v=<timestamp>`. Браузер кэширует URL с параметром, при смене параметра берёт свежую версию. Если `version.txt` недоступен — graceful fallback на `content.json` без параметра  
**Шаблон:** step3_index_html.md — функция `loadContent` + двухшаговый fetch; step5_save_php.md — `file_put_contents version.txt` после rename

---

*Обновлять после каждого исправления. Перечитывать перед генерацией новой админки.*
