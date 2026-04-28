# Проект: [НАЗВАНИЕ]

## Описание
[Краткое описание бизнеса клиента — 1-2 предложения]
Контактное лицо: [имя]
Дата создания: [YYYY-MM-DD]

## Сайт
URL: [https://...]
Пароль admin: [установить перед деплоем]

## FTP
Host: [ftp.хостинг.ru]
User: [логин]
Password: [пароль]
Remote dir: [/public_html или /имя_домена]

## Загрузка на хостинг
После генерации и проверки — загрузить файлы из output/ на хостинг:
```python
import ftplib, os
host = '[host]'
user = '[user]'
pwd  = '[password]'
files = ['admin.html', 'index.html', 'content.json', 'save.php', 'upload.php']
with ftplib.FTP(host) as ftp:
    ftp.login(user, pwd)
    ftp.cwd('[remote_dir]')
    for f in files:
        with open(f'output/{f}', 'rb') as fp:
            ftp.storbinary(f'STOR {f}', fp)
print('Готово')
```

## Чеклист перед деплоем
- [ ] Заменить CHANGE_ME_PASSWORD в output/save.php
- [ ] Заменить CHANGE_ME_PASSWORD в output/upload.php
- [ ] Проверить output/index.html в браузере (контент загружается)
- [ ] Проверить output/admin.html (вход, редактирование, сохранение)
- [ ] Создать папку images/uploads/ на хостинге (права 755)

## Статус
- [ ] Генерация выполнена
- [ ] Задеплоено на хостинг
- [ ] Пароль сменён на боевой
- [ ] Передано клиенту
