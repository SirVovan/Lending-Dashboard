## §AdminWYSIWYG — Admin-панель (WYSIWYG с iframe)

### Концепция layout

```
┌──────────────┬──────────────────────────┬────────────────┐
│ SIDEBAR      │ IFRAME: index.html       │ FIELDS PANEL   │
│ 200px        │ (масштабирован по ширине)│ 360px          │
│ • Главная    │  [реальный лендинг]      │ Поля секции    │
│ • Hero       │  клик → подсветка +      │ без аккордеона │
│ • ★ Events   │  фокус в правой панели   │                │
│ • ...        │                          │ [Отмена][Save] │
└──────────────┴──────────────────────────┴────────────────┘
```

- Центр — `<iframe src="index.html">` шириной 1280px, масштабируется через `transform: scale()`
- Правая панель — поля редактирования без аккордеона, в `fields-section`
- Клик по iframe → подсвечивает элемент (gold outline) + скроллит правую панель к полю
- Live preview — любое изменение поля немедленно отражается в iframe
- Save — POST на save.php, НЕ перезагружает iframe
- Отмена — откат к `_dataSnapshot` на момент последнего сохранения

### CSS

```html
<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Управление сайтом</title>

  <!-- ВСТАВИТЬ ТЕ ЖЕ <link> шрифтов что в лендинге -->

  <style>
    :root {
      /* [скопировать все --переменные из :root лендинга] */

      --sidebar-w:  200px;
      --fields-w:   360px;
      --input-bg:   rgba(255,255,255,0.05);
      --input-brd:  rgba(180,140,255,0.30);
      --input-foc:  rgba(112,64,200,0.50);
      --success:    #4caf7d;
      --danger:     #e05555;
    }

    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
    html, body { height: 100%; overflow: hidden; }
    body {
      background: var(--bg);
      color: var(--white, var(--text));
      font-family: /* основной шрифт */;
      font-size: 14px;
    }

    /* ── AUTH ────────────────────────────────────────────── */
    #authOverlay {
      position: fixed; inset: 0; z-index: 9999;
      background: var(--bg);
      display: flex; align-items: center; justify-content: center;
    }
    .auth-box {
      background: var(--bg2, rgba(255,255,255,.06));
      border: 1px solid var(--border);
      border-radius: 8px; padding: 48px 40px; width: 360px; text-align: center;
    }
    .auth-logo { font-family: /* заголовочный шрифт */; font-size: 1.5rem; color: var(--gold, var(--accent)); margin-bottom: 4px; }
    .auth-hint { font-size: .78rem; color: var(--muted); margin-bottom: 28px; }
    .auth-error { font-size: .78rem; color: var(--danger); margin-bottom: 12px; display: none; }

    /* ── LAYOUT ──────────────────────────────────────────── */
    #adminApp { display: none; height: 100vh; }
    #adminApp.visible {
      display: grid;
      grid-template-columns: var(--sidebar-w) 1fr var(--fields-w);
      grid-template-rows: 100vh;
    }

    /* ── SIDEBAR ─────────────────────────────────────────── */
    .sidebar {
      grid-column: 1; height: 100vh;
      background: var(--bg3, rgba(0,0,0,.3));
      border-right: 1px solid var(--border);
      overflow-y: auto;
      display: flex; flex-direction: column;
    }
    .sidebar-logo {
      padding: 18px 16px 14px;
      font-family: /* заголовочный шрифт */;
      font-size: .95rem; color: var(--gold, var(--accent));
      border-bottom: 1px solid var(--border); flex-shrink: 0;
    }
    .sidebar-logo small {
      display: block; font-family: /* основной шрифт */;
      font-size: .65rem; color: var(--muted); font-weight: 400; margin-top: 3px;
    }
    .sidebar-nav { flex: 1; padding: 6px 0; }
    .nav-item {
      display: block; width: 100%; padding: 9px 16px;
      background: none; border: none; border-left: 2px solid transparent;
      color: var(--muted); font-family: /* основной шрифт */;
      font-size: .78rem; font-weight: 500;
      text-align: left; cursor: pointer;
      transition: background .15s, color .15s, border-color .15s;
      white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
    }
    .nav-item:hover { background: var(--glass); color: var(--white, var(--text)); }
    .nav-item.active {
      background: var(--glass2, rgba(255,255,255,.10));
      color: var(--white, var(--text));
      border-left-color: var(--gold, var(--accent));
    }
    .nav-item.highlight { color: var(--gold-lt, var(--accent)); }
    .nav-item.highlight.active { border-left-color: var(--gold-lt, var(--accent)); }
    .nav-divider { height: 1px; background: var(--border); margin: 5px 14px; }
    .sidebar-footer { padding: 10px 12px; border-top: 1px solid var(--border); flex-shrink: 0; }

    /* ── PREVIEW AREA ────────────────────────────────────── */
    .preview-area { grid-column: 2; position: relative; background: #000; overflow: hidden; }
    .preview-area iframe {
      position: absolute; top: 0; left: 0;
      width: 1280px; height: 100vh;
      border: none; transform-origin: top left;
      pointer-events: auto;
    }
    .preview-home { display: none; height: 100%; align-items: center; justify-content: center; flex-direction: column; gap: 20px; color: var(--muted); text-align: center; padding: 40px; }
    .preview-home.visible { display: flex; }

    /* ── FIELDS PANEL ────────────────────────────────────── */
    #fieldsPanel {
      grid-column: 3; height: 100vh;
      overflow-y: auto;
      background: var(--bg3, rgba(0,0,0,.3));
      border-left: 1px solid var(--border);
      padding: 20px 18px 90px;
    }
    .fields-section { display: none; }
    .fields-section.active { display: block; }
    .fields-title { font-family: /* заголовочный шрифт */; font-size: 1.1rem; color: var(--white, var(--text)); margin-bottom: 4px; }
    .fields-hint { font-size: .74rem; color: var(--muted); margin-bottom: 18px; line-height: 1.5; }

    .field-group {
      background: var(--glass, rgba(255,255,255,.04));
      border: 1px solid var(--border); border-radius: 4px;
      padding: 14px 14px 6px; margin-bottom: 10px;
    }
    .field-group-title {
      font-size: .68rem; font-weight: 700; letter-spacing: .08em; text-transform: uppercase;
      color: var(--gold-lt, var(--accent)); margin-bottom: 12px; padding-bottom: 8px;
      border-bottom: 1px solid var(--border);
    }
    .field-label { font-size: .68rem; font-weight: 700; letter-spacing: .06em; text-transform: uppercase; color: var(--muted); margin-bottom: 4px; }
    .field-hint { font-size: .7rem; color: var(--muted); margin-bottom: 6px; line-height: 1.4; }
    .field-input {
      width: 100%; padding: 8px 11px;
      background: var(--input-bg); border: 1px solid var(--input-brd); border-radius: 3px;
      color: var(--white, var(--text)); font-family: /* основной шрифт */; font-size: .82rem; line-height: 1.5;
      transition: border-color .2s, box-shadow .2s; margin-bottom: 10px;
    }
    .field-input:focus { outline: none; border-color: var(--accent-lt, var(--accent)); box-shadow: 0 0 0 3px var(--input-foc); }
    textarea.field-input { min-height: 120px; resize: vertical; font-size: .76rem; }

    /* ── ARRAYS ──────────────────────────────────────────── */
    .array-wrap { margin-bottom: 6px; }
    .array-item { background: rgba(0,0,0,.25); border: 1px solid var(--border); border-radius: 3px; padding: 10px 12px; margin-bottom: 7px; }
    .array-item-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 8px; }
    .array-item-num { font-size: .66rem; font-weight: 700; color: var(--muted); letter-spacing: .06em; text-transform: uppercase; }
    .btn-del { background: none; border: none; color: var(--danger); cursor: pointer; font-size: .74rem; padding: 2px 6px; border-radius: 2px; transition: background .15s; }
    .btn-del:hover { background: rgba(224,85,85,.12); }
    .btn-add { display: inline-flex; align-items: center; gap: 5px; background: var(--glass); border: 1px solid var(--border); color: var(--muted); padding: 6px 12px; border-radius: 3px; cursor: pointer; font-size: .76rem; font-family: /* основной шрифт */; transition: border-color .2s, color .2s; margin-top: 2px; }
    .btn-add:hover { border-color: var(--accent); color: var(--white, var(--text)); }

    /* ── IMAGE UPLOAD ────────────────────────────────────── */
    .img-preview { max-width: 100%; max-height: 110px; border-radius: 3px; border: 1px solid var(--border); display: block; margin-bottom: 7px; object-fit: cover; }
    .img-preview[src=""], .img-preview:not([src]) { display: none; }
    .btn-upload { display: inline-flex; align-items: center; gap: 5px; background: var(--glass); border: 1px solid var(--border); color: var(--gold, var(--accent)); padding: 6px 12px; border-radius: 3px; cursor: pointer; font-size: .76rem; font-family: /* основной шрифт */; transition: opacity .2s; margin-bottom: 10px; }
    .btn-upload:hover { opacity: .8; }

    /* ── SAVE BAR ────────────────────────────────────────── */
    .save-bar {
      position: fixed; bottom: 0; right: 0; width: var(--fields-w);
      padding: 11px 18px;
      background: rgba(0,0,0,.9); border-top: 1px solid var(--border);
      backdrop-filter: blur(12px);
      display: flex; align-items: center; justify-content: flex-end; gap: 12px;
      z-index: 100;
    }
    .save-status { font-size: .78rem; color: var(--success); opacity: 0; transition: opacity .3s; }
    .save-status.show { opacity: 1; }

    /* ── BUTTONS ─────────────────────────────────────────── */
    .btn-primary {
      background: var(--gold, var(--accent)); color: var(--bg, #000);
      border: none; padding: 10px 28px; font-family: /* основной шрифт */;
      font-size: .8rem; font-weight: 700; letter-spacing: .08em; text-transform: uppercase;
      border-radius: 2px; cursor: pointer; transition: opacity .2s;
    }
    .btn-primary:hover { opacity: .85; }
    .btn-primary:disabled { opacity: .5; cursor: default; }
    .btn-secondary { background: transparent; border: 1px solid var(--border); color: var(--muted); padding: 8px 16px; font-family: /* основной шрифт */; font-size: .76rem; border-radius: 2px; cursor: pointer; transition: border-color .2s, color .2s; }
    .btn-secondary:hover { border-color: var(--accent); color: var(--white, var(--text)); }

    @media (max-width: 1100px) {
      #adminApp.visible { grid-template-columns: var(--sidebar-w) 0 var(--fields-w); }
      .preview-area { display: none; }
    }
  </style>
</head>
<body>
```

### HTML-структура

```html
<div id="authOverlay">
  <div class="auth-box">
    <div class="auth-logo">[Название из content.json]</div>
    <div class="auth-hint">Введите пароль для входа в панель управления</div>
    <div class="auth-error" id="authError">Неверный пароль</div>
    <input type="password" class="field-input" id="authInput"
           placeholder="Пароль" style="margin-bottom:12px"
           onkeydown="if(event.key==='Enter')doAuth()">
    <button class="btn-primary" onclick="doAuth()" style="width:100%">Войти</button>
  </div>
</div>

<div id="adminApp">
  <aside class="sidebar">
    <div class="sidebar-logo">
      [Название]
      <small>Панель управления</small>
    </div>
    <nav class="sidebar-nav">
      <button class="nav-item active" data-section="home" onclick="showSection('home')">Главная</button>
      <div class="nav-divider"></div>
      <!-- По кнопке для каждой секции content.json (кроме _meta, _seo) -->
      <!-- events/мероприятия → добавить класс highlight -->
      <button class="nav-item" data-section="СЕКЦИЯ" onclick="showSection('СЕКЦИЯ')">Название</button>
    </nav>
    <div class="sidebar-footer">
      <button class="btn-secondary" style="width:100%" onclick="logout()">Выйти</button>
    </div>
  </aside>

  <div class="preview-area" id="previewArea">
    <div class="preview-home" id="previewHome">
      <div style="font-family:/* заголовочный шрифт */; font-size:1.8rem; color:var(--white,var(--text)); margin-bottom:8px;">Привет!</div>
      <div style="color:var(--muted); font-size:.9rem; margin-bottom:24px;">Выберите раздел в меню слева,<br>чтобы начать редактирование.</div>
      <a href="index.html" target="_blank" style="background:var(--accent);color:var(--white,#fff);text-decoration:none;padding:13px 36px;border-radius:3px;font-size:.88rem;font-weight:600;">
        Открыть сайт →
      </a>
    </div>
    <iframe id="sitePreview" style="display:none"></iframe>
  </div>

  <div class="fields-panel" id="fieldsPanel">
    <div class="fields-section active" id="fields-home"></div>

    <!-- Для каждой секции content.json -->
    <div class="fields-section" id="fields-СЕКЦИЯ">
      <div class="fields-title">Название секции</div>
      <div class="fields-hint">Кликните на элемент в центре чтобы быстро перейти к полю</div>

      <!-- Скалярные поля -->
      <div class="field-group">
        <div class="field-group-title">Основные тексты</div>
        <div class="field-label">Заголовок</div>
        <input type="text" class="field-input" data-path="СЕКЦИЯ.title">
        <div class="field-label">Подзаголовок</div>
        <textarea class="field-input" data-path="СЕКЦИЯ.subtitle"></textarea>
      </div>

      <!-- Массив объектов -->
      <div class="field-group">
        <div class="field-group-title">Список элементов</div>
        <div class="array-wrap" id="arr-СЕКЦИЯ-items"></div>
        <button class="btn-add" onclick="addItem('СЕКЦИЯ.items','arr-СЕКЦИЯ-items',SCHEMA)">+ Добавить</button>
      </div>

      <!-- Изображение -->
      <div class="field-group">
        <div class="field-group-title">Изображение</div>
        <img class="img-preview" id="prev-СЕКЦИЯ-photo" src="">
        <input type="text" class="field-input" data-path="СЕКЦИЯ.photo_url" placeholder="Путь к файлу">
        <input type="file" id="file-СЕКЦИЯ-photo" accept="image/*" style="display:none"
               onchange="uploadImage(this,'СЕКЦИЯ.photo_url','prev-СЕКЦИЯ-photo')">
        <button class="btn-upload" onclick="document.getElementById('file-СЕКЦИЯ-photo').click()">
          📎 Загрузить фото
        </button>
      </div>
    </div>
  </div>

  <div class="save-bar">
    <span class="save-status" id="saveStatus"></span>
    <button class="btn-secondary" onclick="cancelData()">Отмена</button>
    <button class="btn-primary" onclick="saveData()">Сохранить</button>
  </div>
</div>
```

### JavaScript

```javascript
<script>
var _pwd = '';
var _data = {};
var _dataSnapshot = null;
var _currentSection = 'home';

/* ── AUTH ──────────────────────────────────────────────── */
function doAuth() {
  var p = document.getElementById('authInput').value.trim();
  if (!p) return;
  var btn = document.querySelector('#authOverlay .btn-primary');
  btn.disabled = true; btn.textContent = 'Проверяю...';
  fetch('save.php', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ _action: 'auth', _auth: p })
  })
  .then(function(r) { return r.json(); })
  .then(function(res) {
    if (res.ok) {
      _pwd = p;
      sessionStorage.setItem('_ap', p);
      document.getElementById('authOverlay').style.display = 'none';
      document.getElementById('adminApp').classList.add('visible');
      showSection('home');
      loadContent();
    } else {
      document.getElementById('authError').style.display = 'block';
      btn.disabled = false; btn.textContent = 'Войти';
    }
  })
  .catch(function() { btn.disabled = false; btn.textContent = 'Войти'; });
}

function logout() { sessionStorage.removeItem('_ap'); location.reload(); }

/* ── DATA ────────────────────────────────────────────────── */
function loadContent() {
  fetch('content.json?t=' + Date.now())
    .then(function(r) { return r.json(); })
    .then(function(d) { _data = d; fillForm(); })
    .catch(function() { alert('Не удалось загрузить content.json'); });
}

function fillForm() {
  _dataSnapshot = JSON.parse(JSON.stringify(_data));
  document.querySelectorAll('[data-path]').forEach(function(el) {
    var val = getPath(_data, el.getAttribute('data-path'));
    if (val !== undefined && val !== null) el.value = String(val);
  });
  renderAllArrays();
  document.querySelectorAll('[data-path]').forEach(function(el) {
    el.removeEventListener('input', onFieldInput);
    el.addEventListener('input', onFieldInput);
  });
}

/* ── LIVE PREVIEW ──────────────────────────────────────── */
function onFieldInput() {
  var path = this.getAttribute('data-path');
  var val  = this.value;
  setPath(_data, path, val);
  updateIframeField(path, val);
}

function updateIframeField(path, val) {
  var iframe = document.getElementById('sitePreview');
  if (!iframe || iframe.style.display === 'none') return;
  try {
    var iDoc = iframe.contentDocument || iframe.contentWindow.document;
    var iWin = iframe.contentWindow;
    iDoc.querySelectorAll('[data-bind="' + path + '"]').forEach(function(el) { el.textContent = val; });
    iDoc.querySelectorAll('[data-bind-html="' + path + '"]').forEach(function(el) { el.innerHTML = val; });
    iDoc.querySelectorAll('[data-bind-src="' + path + '"]').forEach(function(el) { el.src = val; });
    var arrMatch = path.match(/^([^[]+)\[/);
    if (arrMatch) {
      var arrPath = arrMatch[1];
      var container = iDoc.querySelector('[data-builder="' + arrPath + '"]');
      if (container) {
        var buildFn = iWin['build_' + arrPath.replace(/\./g, '_')];
        var items = getPath(_data, arrPath);
        if (typeof buildFn === 'function' && Array.isArray(items)) {
          container.innerHTML = '';
          items.forEach(function(item) { container.appendChild(buildFn(item)); });
          if (iWin._revealObserver) {
            iDoc.querySelectorAll('.reveal:not(.visible)').forEach(function(el) { iWin._revealObserver.observe(el); });
          }
        }
      }
    }
  } catch(e) {}
}

/* ── ARRAYS ────────────────────────────────────────────── */
// renderAllArrays() — генерируется по массивам в content.json:
// function renderAllArrays() {
//   renderArray('services.items', 'arr-services-items',
//     [{key:'num',label:'Номер'},{key:'name',label:'Название'},{key:'desc',label:'Описание',long:true}]);
//   renderStringArray('hero.tags', 'arr-hero-tags');
//   resubscribeArrayInputs();
// }

function renderArray(path, cid, schema) {
  var items = getPath(_data, path);
  if (!Array.isArray(items)) return;
  var container = document.getElementById(cid);
  if (!container) return;
  container.innerHTML = '';
  items.forEach(function(item, idx) { container.appendChild(buildItem(item, idx, path, schema)); });
}

function buildItem(item, idx, path, schema) {
  var div = document.createElement('div');
  div.className = 'array-item';
  var html = '<div class="array-item-header">' +
    '<span class="array-item-num">Элемент ' + (idx+1) + '</span>' +
    '<button class="btn-del" onclick="deleteItem(\'' + path + '\',' + idx + ',\'' +
    'arr-' + path.replace(/\./g,'-') + '\')">✕</button></div>';
  schema.forEach(function(f) {
    html += '<div class="field-label">' + f.label + '</div>';
    if (f.long) {
      html += '<textarea class="field-input" data-path="' + path + '[' + idx + '].' + f.key + '">' +
        escHtml(item[f.key] || '') + '</textarea>';
    } else {
      html += '<input type="text" class="field-input" data-path="' + path + '[' + idx + '].' + f.key + '" value="">';
    }
  });
  div.innerHTML = html;
  schema.forEach(function(f) {
    if (!f.long) {
      var inp = div.querySelector('[data-path="' + path + '[' + idx + '].' + f.key + '"]');
      if (inp) inp.value = item[f.key] || '';
    }
  });
  return div;
}

function renderStringArray(path, cid) {
  var arr = getPath(_data, path);
  if (!Array.isArray(arr)) return;
  var container = document.getElementById(cid);
  if (!container) return;
  container.innerHTML = '';
  arr.forEach(function(val, idx) {
    var div = document.createElement('div');
    div.className = 'array-item';
    div.innerHTML = '<div class="array-item-header"><span class="array-item-num">Пункт ' + (idx+1) + '</span>' +
      '<button class="btn-del" onclick="deleteStringItem(\'' + path.replace(/'/g,"\\'") + '\',' + idx + ',\'' + cid + '\')">✕</button></div>';
    var inp = document.createElement('input');
    inp.type = 'text'; inp.className = 'field-input';
    inp.setAttribute('data-path', path + '[' + idx + ']');
    inp.value = val;
    div.appendChild(inp);
    container.appendChild(div);
  });
}

function addItem(path, cid, schema) {
  var arr = getPath(_data, path) || [];
  var template = {}; schema.forEach(function(f) { template[f.key] = ''; });
  arr.push(template); setPath(_data, path, arr);
  renderAllArrays(); resubscribeArrayInputs();
}

function addStringItem(path, cid) {
  var arr = getPath(_data, path) || [];
  arr.push(''); setPath(_data, path, arr);
  renderStringArray(path, cid); resubscribeArrayInputs();
}

function deleteItem(path, idx, cid) {
  if (!confirm('Удалить элемент ' + (idx+1) + '?')) return;
  var arr = getPath(_data, path);
  if (Array.isArray(arr)) arr.splice(idx, 1);
  renderAllArrays(); resubscribeArrayInputs();
}

function deleteStringItem(path, idx, cid) {
  if (!confirm('Удалить пункт ' + (idx+1) + '?')) return;
  var arr = getPath(_data, path);
  if (Array.isArray(arr)) arr.splice(idx, 1);
  renderStringArray(path, cid); resubscribeArrayInputs();
}

function resubscribeArrayInputs() {
  document.querySelectorAll('[data-path]').forEach(function(el) {
    el.removeEventListener('input', onFieldInput);
    el.addEventListener('input', onFieldInput);
  });
}

/* ── SAVE ──────────────────────────────────────────────── */
function collectData() {
  document.querySelectorAll('[data-path]').forEach(function(el) {
    var path = el.getAttribute('data-path');
    if (path.indexOf('[') === -1) setPath(_data, path, el.value);
  });
  document.querySelectorAll('.array-item [data-path]').forEach(function(el) {
    setPath(_data, el.getAttribute('data-path'), el.value);
  });
}

function saveData() {
  collectData();
  var btn = document.querySelector('.save-bar .btn-primary');
  btn.disabled = true; btn.textContent = 'Сохраняю...';
  var payload = JSON.parse(JSON.stringify(_data));
  payload._action = 'save'; payload._auth = _pwd;
  fetch('save.php', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(payload) })
    .then(function(r) { return r.json(); })
    .then(function(res) {
      btn.disabled = false; btn.textContent = 'Сохранить';
      if (res.ok) {
        _dataSnapshot = JSON.parse(JSON.stringify(_data));
        var s = document.getElementById('saveStatus');
        s.textContent = 'Сохранено ✓'; s.classList.add('show');
        setTimeout(function() { s.classList.remove('show'); }, 2500);
      } else { alert('Ошибка: ' + (res.error || 'неизвестная')); }
    })
    .catch(function() { btn.disabled = false; btn.textContent = 'Сохранить'; alert('Ошибка соединения'); });
}

function cancelData() {
  if (!_dataSnapshot) return;
  if (!confirm('Отменить все несохранённые изменения?')) return;
  _data = JSON.parse(JSON.stringify(_dataSnapshot));
  fillForm();
  var iframe = document.getElementById('sitePreview');
  if (_iframeReady && iframe) {
    try {
      var iDoc = iframe.contentDocument || iframe.contentWindow.document;
      iDoc.querySelectorAll('[data-bind]').forEach(function(el) {
        var val = getPath(_data, el.getAttribute('data-bind'));
        if (val !== undefined) el.textContent = String(val);
      });
      iDoc.querySelectorAll('[data-bind-html]').forEach(function(el) {
        var val = getPath(_data, el.getAttribute('data-bind-html'));
        if (val !== undefined) el.innerHTML = val;
      });
    } catch(e) {}
  }
  var s = document.getElementById('saveStatus');
  s.textContent = 'Изменения отменены'; s.classList.add('show');
  setTimeout(function() { s.classList.remove('show'); }, 2000);
}

/* ── UPLOAD ────────────────────────────────────────────── */
function uploadImage(input, path, previewId) {
  var file = input.files[0]; if (!file) return;
  var form = new FormData(); form.append('file', file); form.append('_auth', _pwd);
  fetch('upload.php', { method: 'POST', body: form })
    .then(function(r) { return r.json(); })
    .then(function(res) {
      if (res.ok) {
        setPath(_data, path, res.path);
        var inp = document.querySelector('[data-path="' + path + '"]');
        if (inp) inp.value = res.path;
        var prev = document.getElementById(previewId);
        if (prev) prev.src = res.path;
      } else { alert('Ошибка загрузки: ' + (res.error || '')); }
    });
}

/* ── IFRAME ────────────────────────────────────────────── */
var SECTION_IDS = { /* 'admin_section': 'landing_section_id' */ };
var _iframeReady = false;

function initIframe() {
  var iframe = document.getElementById('sitePreview');
  iframe.style.display = 'block';
  document.getElementById('previewHome').classList.remove('visible');
  scaleIframe();
  if (_iframeReady) { scrollIframeTo(_currentSection); return; }
  if (!iframe.getAttribute('src')) { iframe.src = 'index.html'; }
  iframe.onload = function() {
    _iframeReady = true;
    scrollIframeTo(_currentSection);
    attachIframeClickHandler();
  };
}

function scaleIframe() {
  var area = document.getElementById('previewArea');
  var iframe = document.getElementById('sitePreview');
  var scale = area.offsetWidth / 1280;
  iframe.style.transform = 'scale(' + scale + ')';
  iframe.style.height = (100 / scale) + 'vh';
}

function scrollIframeTo(sectionId) {
  var landingId = (SECTION_IDS[sectionId] || sectionId);
  var iframe = document.getElementById('sitePreview');
  try {
    var el = (iframe.contentDocument || iframe.contentWindow.document).getElementById(landingId);
    if (el) el.scrollIntoView({ behavior: 'smooth', block: 'start' });
  } catch(e) {}
}

function attachIframeClickHandler() {
  var iframe = document.getElementById('sitePreview');
  try {
    var iDoc = iframe.contentDocument || iframe.contentWindow.document;
    if (!iDoc.getElementById('admin-highlight-style')) {
      var style = iDoc.createElement('style');
      style.id = 'admin-highlight-style';
      style.textContent =
        '[data-admin-focus]{outline:2px solid #c9a227!important;outline-offset:3px;background:rgba(201,162,39,0.08)!important;border-radius:3px;}' +
        '[data-admin-builder-active]{outline:2px solid #c9a227!important;outline-offset:3px;background:rgba(201,162,39,0.05)!important;border-radius:4px;}' +
        '[data-admin-builder-active]>*{cursor:pointer!important;outline:1px dashed rgba(201,162,39,0.5)!important;outline-offset:2px;}' +
        '[data-bind],[data-bind-html],[data-bind-src],[data-builder]{cursor:pointer!important;}';
      iDoc.head.appendChild(style);
    }
    iDoc.addEventListener('mousedown', function(e) {
      e.preventDefault(); e.stopPropagation(); e.stopImmediatePropagation();
      handleIframeClick(e.target, iDoc);
    }, true);
    iDoc.addEventListener('click', function(e) {
      e.preventDefault(); e.stopPropagation(); e.stopImmediatePropagation();
    }, true);
    var _scrollTimer = null;
    iframe.contentWindow.addEventListener('scroll', function() {
      clearTimeout(_scrollTimer);
      _scrollTimer = setTimeout(function() { updateActiveNavFromScroll(iDoc, iframe.contentWindow); }, 80);
    }, { passive: true });
  } catch(e) {}
}

var _activeBuilderEl = null;

function handleIframeClick(target, iDoc) {
  var node = target;
  while (node && node !== iDoc.body) {
    if (node.hasAttribute && (node.hasAttribute('data-bind') || node.hasAttribute('data-bind-html') || node.hasAttribute('data-bind-src'))) {
      var path = node.getAttribute('data-bind') || node.getAttribute('data-bind-html') || node.getAttribute('data-bind-src');
      clearIframeHighlights(iDoc);
      node.setAttribute('data-admin-focus', '');
      focusFieldByPath(path);
      _activeBuilderEl = null;
      return;
    }
    if (node.hasAttribute && node.hasAttribute('data-builder')) {
      var builderPath = node.getAttribute('data-builder');
      if (_activeBuilderEl === node) {
        var children = Array.prototype.slice.call(node.children);
        var clickedChild = target;
        while (clickedChild && clickedChild.parentNode !== node) clickedChild = clickedChild.parentNode;
        var idx = children.indexOf(clickedChild);
        if (idx >= 0) focusFieldByPath(builderPath + '[' + idx + ']');
      } else {
        clearIframeHighlights(iDoc);
        node.setAttribute('data-admin-builder-active', '');
        _activeBuilderEl = node;
        focusFieldByPath(builderPath);
      }
      return;
    }
    node = node.parentNode;
  }
}

function clearIframeHighlights(iDoc) {
  iDoc.querySelectorAll('[data-admin-focus]').forEach(function(el) { el.removeAttribute('data-admin-focus'); });
  iDoc.querySelectorAll('[data-admin-builder-active]').forEach(function(el) { el.removeAttribute('data-admin-builder-active'); });
}

function focusFieldByPath(path) {
  var sectionId = path.split('.')[0];
  if (sectionId !== _currentSection) switchFieldsOnly(sectionId);
  requestAnimationFrame(function() {
    requestAnimationFrame(function() {
      var fieldsEl = document.getElementById('fields-' + sectionId);
      if (!fieldsEl) return;
      var field = fieldsEl.querySelector('[data-path="' + path + '"]');
      if (field) { highlightField(field); return; }
      var arrId = 'arr-' + path.replace(/\./g, '-');
      var arrWrap = document.getElementById(arrId);
      if (arrWrap) { scrollPanelToTop(arrWrap.closest('.field-group') || arrWrap); }
    });
  });
}

function highlightField(field) {
  var prev = document.querySelector('.field-input.field-focused');
  if (prev) { prev.classList.remove('field-focused'); prev.style.boxShadow = ''; prev.style.borderColor = ''; }
  var group = field.closest ? field.closest('.field-group') : null;
  scrollPanelToTop(group || field);
  field.classList.add('field-focused');
  field.style.transition = 'box-shadow .25s, border-color .25s';
  field.style.boxShadow = '0 0 0 3px rgba(201,162,39,0.5)';
  field.style.borderColor = '#c9a227';
  setTimeout(function() {
    field.style.boxShadow = ''; field.style.borderColor = '';
    field.classList.remove('field-focused');
  }, 2500);
}

function scrollPanelToTop(el) {
  var panel = document.getElementById('fieldsPanel');
  if (!panel || !el) return;
  var panelRect = panel.getBoundingClientRect();
  var elRect = el.getBoundingClientRect();
  var newScroll = panel.scrollTop + (elRect.top - panelRect.top) - 12;
  var elFitsInPanel = elRect.height <= panel.clientHeight;
  var elFullyVisible = elRect.top >= panelRect.top && elRect.bottom <= panelRect.bottom;
  if (!elFullyVisible || !elFitsInPanel) panel.scrollTop = newScroll;
}

function updateActiveNavFromScroll(iDoc, iWin) {
  var scrollY = iWin.scrollY || iWin.pageYOffset || 0;
  var centerY = scrollY + iWin.innerHeight * 0.4;
  var bestSection = null, bestDist = Infinity;
  document.querySelectorAll('.nav-item[data-section]').forEach(function(navItem) {
    var sid = navItem.getAttribute('data-section');
    if (sid === 'home') return;
    var el = iDoc.getElementById(SECTION_IDS[sid] || sid);
    if (!el) return;
    var rect = el.getBoundingClientRect();
    var dist = Math.abs((rect.top + scrollY) - centerY);
    if (dist < bestDist) { bestDist = dist; bestSection = sid; }
  });
  if (bestSection && bestSection !== _currentSection) {
    _currentSection = bestSection;
    document.querySelectorAll('.nav-item').forEach(function(el) { el.classList.remove('active'); });
    var activeNav = document.querySelector('.nav-item[data-section="' + bestSection + '"]');
    if (activeNav) { activeNav.classList.add('active'); activeNav.scrollIntoView({ block: 'nearest', behavior: 'smooth' }); }
    switchFieldsOnly(bestSection);
  }
}

/* ── UI ────────────────────────────────────────────────── */
function showSection(id) {
  _currentSection = id;
  document.querySelectorAll('.nav-item').forEach(function(el) { el.classList.remove('active'); });
  var navItem = document.querySelector('[data-section="' + id + '"]');
  if (navItem) navItem.classList.add('active');
  document.querySelectorAll('.fields-section').forEach(function(el) { el.classList.remove('active'); });
  var fieldsEl = document.getElementById('fields-' + id);
  if (fieldsEl) fieldsEl.classList.add('active');
  if (id === 'home') {
    document.getElementById('sitePreview').style.display = 'none';
    document.getElementById('previewHome').classList.add('visible');
  } else {
    document.getElementById('previewHome').classList.remove('visible');
    initIframe();
    scrollIframeTo(id);
  }
}

function switchFieldsOnly(id) {
  _currentSection = id;
  document.querySelectorAll('.nav-item').forEach(function(el) { el.classList.remove('active'); });
  var navItem = document.querySelector('.nav-item[data-section="' + id + '"]');
  if (navItem) navItem.classList.add('active');
  document.querySelectorAll('.fields-section').forEach(function(el) { el.classList.remove('active'); });
  var fieldsEl = document.getElementById('fields-' + id);
  if (fieldsEl) fieldsEl.classList.add('active');
}

/* ── UTILS ─────────────────────────────────────────────── */
function getPath(obj, path) {
  return path.replace(/\[(\d+)\]/g, '.$1').split('.').reduce(function(o, k) {
    return o && o[k] !== undefined ? o[k] : undefined;
  }, obj);
}

function setPath(obj, path, val) {
  var parts = path.replace(/\[(\d+)\]/g, '.$1').split('.');
  var o = obj;
  for (var i = 0; i < parts.length - 1; i++) {
    var k = parts[i];
    if (o[k] === undefined) o[k] = isNaN(parts[i+1]) ? {} : [];
    o = o[k];
  }
  o[parts[parts.length - 1]] = val;
}

function escHtml(str) {
  return String(str).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;').replace(/"/g,'&quot;');
}

window.addEventListener('resize', function() { if (_iframeReady) scaleIframe(); });

window.onload = function() {
  var saved = sessionStorage.getItem('_ap');
  if (saved) {
    _pwd = saved;
    document.getElementById('authOverlay').style.display = 'none';
    document.getElementById('adminApp').classList.add('visible');
    showSection('home');
    loadContent();
  }
};
</script>
```

---

## §AdminInputRules — правила генерации полей

- Значение < 80 символов → `<input type="text">`
- Значение ≥ 80 символов или HTML-поле → `<textarea>`
- HTML-поле → добавить `<div class="field-hint">Для форматирования: &lt;strong&gt;, &lt;em&gt;, &lt;br&gt;</div>`
- Телефон → `<input type="tel">`, email → `<input type="email">`

Правила для полей секции:

1. Для каждой секции из `content.json` (кроме `_meta`, `_seo`) — создать `<div class="fields-section" id="fields-СЕКЦИЯ">`
2. Скалярные поля группировать в `.field-group` по смыслу (основные тексты / кнопки / контакты)
3. Массивы объектов — `renderArray()` с полной схемой `[{key, label, long?}]`
4. Строковые массивы — `renderStringArray()`
5. `renderAllArrays()` — вызвать renderArray/renderStringArray для всех массивов, в конце `resubscribeArrayInputs()`
6. В `SECTION_IDS` прописать маппинг если id секции в admin отличается от id в лендинге
7. `events` / `мероприятия` — добавить класс `highlight` в nav-item
