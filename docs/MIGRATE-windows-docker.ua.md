# Перенесення вузла indexd між Windows і Docker

Цей посібник описує, як перенести працюючий вузол indexd Smart Autopilot разом з усіма контрактами з установлення на Windows у Docker і як повернутися назад. Контракти зберігаються в блокчейні, а все, що indexd знає про них (слеби, акаунти, ключі застосунків, налаштування), лежить у базі PostgreSQL. Гаманець виводиться з recovery phrase. Тому перенесення зводиться до двох речей: та сама recovery phrase і база даних.

> **Що перевірено.** Windows → Docker: дамп і відновлення бази, форк-міграція під час першого запуску образу, та сама адреса й баланс гаманця, ті самі 13 контрактів. Відкіт на Windows із перевіркою, що контракти не змінилися. **Не перевірено:** тривала робота Docker-вузла з увімкненою системою, підключення s3d до Docker-indexd (перевірено лише читання адреси з `s3d.db`) і перенесення в зворотний бік (Docker → Windows).

## Правила, які не можна порушувати

1. **Один гаманець — один працюючий вузол.** Два вузли з однією recovery phrase й увімкненою системою змінюють одні й ті самі контракти та заважають один одному. Поки працює один вузол, у другого система вимкнена або він зупинений.
2. **Дамп знімається лише із зупиненого indexd.**
3. **Docker-вузол має бути синхронізований і зупинений уже після зняття дампа.** Інакше в його базі ланцюга може не виявитися останнього блока (див. розділ «Якщо в лозі `resetting chain state`»).
4. **Базу попереднього вузла не видаляйте й не змінюйте, поки новий не перевірено.**

## Що потрібно заздалегідь

- Docker-вузол установлено за [інструкцією](INSTALL-docker.uk.md), у майстрі введено **ту саму recovery phrase**, що й у вузла, який переносите, і він повністю синхронізований (`synced: true`). Перевірка: `GET /api/wallet` в обох вузлів повертає одну й ту саму адресу.
- Версія образу **не старша** за версію бази, яку переносите. Міграції форку застосовуються вперед під час старту, а образ, старіший за базу, відмовиться запускатися з помилкою `fork schema version N is newer than expected M`.
- Місце для дампа (зазвичай кілька МБ) і для копії `s3d.db`.
- Не зволікайте з перенесенням: поки система вимкнена, контракти не продовжуються, а вони закінчуються за кілька тижнів.

## Перенесення Windows → Docker

### Крок 1. Зупинка Windows-вузла

Якщо запущено s3d, надішліть у Sia все, що чекає в буфері, і зупиніть його:

```powershell
cd C:\S3D
.\s3d.exe flush
```

Потім **Ctrl+C** у вікні s3d. Після цього зупиніть indexd на Windows (**Ctrl+C**, дочекайтеся `shutdown complete`). **Docker-вузол поки не зупиняйте:** він має працювати, доки не знято дамп.

Перевірка, що нічого не запущено:

```powershell
Get-Process indexd, s3d -ErrorAction SilentlyContinue
```

Команда не повинна нічого вивести.

### Крок 2. Дамп Windows-бази та контрольні числа

```powershell
$env:PGPASSWORD = "<пароль postgres>"
$w = "E:\Backup\windows-final-$(Get-Date -Format 'yyyyMMdd-HHmm').dump"
& "C:\Program Files\PostgreSQL\18\bin\pg_dump.exe" -U postgres -h 127.0.0.1 -Fc -f $w indexd
Get-Item $w | Select-Object Name, Length
& "C:\Program Files\PostgreSQL\18\bin\psql.exe" -U postgres -h 127.0.0.1 -d indexd -At -c "SELECT (SELECT count(*) FROM contracts), (SELECT count(*) FROM hosts), (SELECT count(*) FROM sectors), (SELECT version FROM fork_schema);"
Remove-Item Env:PGPASSWORD
```

Запишіть чотири числа: контракти, хости, сектори, версія схеми форку. Розмір дампа має бути порядку кількох мегабайтів.

### Крок 3. Зупинка Docker-вузла

Тепер, коли дамп знято, зупиніть indexd у Docker. Дайте йому час завершити запис у базу ланцюга:

```powershell
cd <папка з docker-compose.yml>
docker compose stop -t 90 indexd
docker compose ps -a indexd
```

Статус має бути `Exited (0)`. Контейнер PostgreSQL залишається запущеним (`docker compose ps postgres` показує `healthy`).

### Крок 4. Відновлення бази в Docker

Перш ніж стирати Docker-базу, зніміть її дамп (порядок описано в розділі «Оновлення» [інструкції Docker](INSTALL-docker.uk.md)): він знадобиться під час відкоту.

```powershell
docker compose cp $w postgres:/tmp/win.dump
docker compose exec postgres psql -U indexd -d postgres -c "DROP DATABASE indexd WITH (FORCE);"
docker compose exec postgres psql -U indexd -d postgres -c "CREATE DATABASE indexd OWNER indexd;"
docker compose exec postgres pg_restore -U indexd -d indexd --no-owner --no-privileges /tmp/win.dump
docker compose exec postgres rm /tmp/win.dump
docker compose exec postgres psql -U indexd -d indexd -At -c "SELECT (SELECT count(*) FROM contracts), (SELECT count(*) FROM hosts), (SELECT count(*) FROM sectors), (SELECT version FROM fork_schema);"
```

`pg_restore` без повідомлень — це нормально. Остання команда має повернути **ті самі чотири числа**, що й крок 2. Якщо схема форку в дампі старша, ніж в образі, образ сам доведе її під час першого запуску.

### Крок 5. Вимкнути систему до першого запуску

Відновлені налаштування несуть стан Windows-вузла, де систему було ввімкнено. Якщо запустити Docker-вузол як є, він одразу почне формувати контракти й блокувати хости.

```powershell
docker compose exec postgres psql -U indexd -d indexd -c "UPDATE global_settings SET contracts_maintenance_enabled = false;"
docker compose exec postgres psql -U indexd -d indexd -At -c "SELECT contracts_maintenance_enabled FROM global_settings;"
```

Друга команда має вивести `f`.

### Крок 6. Запуск і перевірка

```powershell
docker compose up -d
Start-Sleep 30
docker compose logs --tail 20 indexd
curl.exe -s -m 20 -u ":<пароль admin>" http://127.0.0.1:9980/api/state
curl.exe -s -m 20 -u ":<пароль admin>" http://127.0.0.1:9980/api/wallet
(curl.exe -s -u ":<пароль admin>" "http://127.0.0.1:9980/api/contracts?limit=100" | ConvertFrom-Json) | Group-Object good | Select-Object Name, Count
```

Порт `9980` замініть на порт admin вашого установлення. Що має вийти:

- у лозі версія образу, за потреби `fork migration complete` і `node started`, але **немає** рядка `resetting chain state`;
- в `api/state`: `synced: true`, `scanHeight` майже дорівнює `syncHeight` (не 0);
- гаманець: та сама адреса й той самий баланс, що у Windows-вузла;
- контракти: така сама кількість, стільки ж `True` і `False`.

### Крок 7. Увімкнути систему

Увімкніть перемикач **System is currently disabled** у вебінтерфейсі Docker-вузла. Через кілька хвилин перевірте лог: помилок бути не повинно, а нові контракти не мають формуватися, якщо ціль `Wanted contracts` уже досягнуто.

Відтепер вважайте перенесення остаточним: вузол почав працювати з контрактами, і відкіт можливий, лише поки він їх не змінював (розділ «Відкіт»).

### Крок 8. Перемикання s3d на Docker-indexd

Адреса indexd зберігається не в `s3d.yml`, а в базі s3d (`s3d.db`, таблиця `global_settings`, поле `indexer_url`), там само ключ застосунку. **Не запускайте `s3d login` повторно:** під час нової реєстрації може створитися нова recovery phrase, і попередні дані перестануть розшифровуватися.

Зупиніть s3d, зробіть копію бази й замініть адресу (потрібен `sqlite3.exe` з sqlite.org):

```powershell
Copy-Item D:\S3D\data\s3d.db E:\Backup\s3d.db.bak
sqlite3 D:\S3D\data\s3d.db "SELECT indexer_url FROM global_settings;"
sqlite3 D:\S3D\data\s3d.db "UPDATE global_settings SET indexer_url = 'http://localhost:29982';"
```

Порт `29982` — порт Application API Docker-вузла на хості (`INDEXD_APP_PORT`). Запустіть s3d: у лозі мають бути `erasure coding scheme accepted` і `indexer transfer limits`. Потім прочитайте будь-який завантажений раніше файл (наприклад, через rclone): це єдина перевірка, що дані доступні. *Цей крок на практиці ще не перевірявся: перевірено лише читання адреси.*

### Крок 9. Windows-вузол

Щоб його випадково не запустити поряд із Docker-вузлом, перейменуйте програму. Базу у Windows-PostgreSQL не видаляйте, вона потрібна для відкоту:

```powershell
Rename-Item C:\IndexD\indexd.exe indexd-windows-disabled.exe
```

## Відкіт (Docker → Windows)

Повернутися можна, **поки Docker-вузол не змінив контракти**. Якщо він їх змінював (поповнював рахунки, дописував сектори, формував чи продовжував контракти), хости пам'ятають новіші ревізії, ніж Windows-база, і запускати Windows-вузол не можна. Перевірка: хеш контрактів в обох базах.

```powershell
$q = "SELECT count(*) AS n, md5(string_agg(j::text, ',' ORDER BY j::text)) AS hash FROM (SELECT (SELECT jsonb_object_agg(k, v) FROM jsonb_each(to_jsonb(c)) AS e(k, v) WHERE k ~ 'revision|remaining|size|capacity|good|state') AS j FROM contracts c) t;"
"Windows:"
$env:PGPASSWORD = "<пароль postgres>"
& "C:\Program Files\PostgreSQL\18\bin\psql.exe" -U postgres -h 127.0.0.1 -d indexd -At -c $q
Remove-Item Env:PGPASSWORD
"Docker:"
docker compose exec postgres psql -U indexd -d indexd -At -c $q
```

Запит бере з кожного контракту поля з ревізією, залишком, розміром, місткістю та статусом (стовпці добираються за маскою) і рахує загальний хеш. Якщо число й хеш **збіглися**, Docker-вузол контрактів не змінював, і відкіт безпечний. Якщо **відрізняються**, Windows-вузол не запускайте: вирішуйте, чи лишатися на Docker-вузлі (Windows-база застаріла).

Порядок відкоту:

1. Зупиніть indexd у Docker: `docker compose stop -t 90 indexd`.
2. Поверніть у Docker попередню базу (дамп, знятий на кроці 4), щоб копія живих контрактів не залишилася поряд із Windows-вузлом: `DROP DATABASE`, `CREATE DATABASE`, `pg_restore`, як на кроці 4.
3. Поверніть ім'я програмі Windows-вузла: `Rename-Item C:\IndexD\indexd-windows-disabled.exe indexd.exe`.
4. Якщо змінювали адресу в s3d, поверніть `http://localhost:19982` тим самим запитом `UPDATE` або відновіть `s3d.db.bak`.
5. Запустіть Windows-indexd. У лозі має бути версія, `node started` і без міграцій; `synced: true`, гаманець з тією самою адресою та балансом.

## Перенесення у зворотний бік (Docker → Windows)

Виконується за тими самими правилами: Windows-вузол підготовлено й синхронізовано, дамп знімається всередині контейнера (`pg_dump -f` і `docker compose cp`, а не перенаправлення `>` у PowerShell), база відновлюється у Windows-PostgreSQL, система вимикається до першого запуску. **Цей напрямок не перевірявся:** зокрема, треба переконатися, якою роллю відновлювати базу та чи вистачає їй прав.

## Якщо в лозі `resetting chain state`

Ознаки: після запуску `scanHeight` дорівнює 0, баланс гаманця 0, а в лозі рядок на кшталт `subscriber failed to fetch updates, resetting chain state … missing block at index …`.

Що ми бачили: дамп пам'ятає, що гаманець просканований до певного блока, а в базі ланцюга Docker-вузла цього блока немає (вузол зупинили раніше, ніж зняли дамп). indexd скинув стан і почав пересканувати гаманець з нуля, що займає години. Гроші й контракти при цьому не втрачені.

Як виправили (спрацювало один раз): зупинили Docker-indexd штатно (`docker compose stop -t 90 indexd`), відновили **той самий дамп** знову (кроки 4–5) і запустили: на той час ланцюг Docker-вузла вже містив потрібний блок, скидання не було. Щоб не потрапляти в цю ситуацію, дотримуйтеся порядку: Docker-вузол працює до кінця кроку 2 і зупиняється лише на кроці 3.

## Часті проблеми

| Симптом | Причина й рішення |
|---|---|
| `fork schema version N is newer than expected M` під час запуску | Образ старший за відновлену базу. Використовуйте образ не старший за базу |
| Одразу після запуску формуються контракти, блокуються хости | Систему не було вимкнено до першого запуску (крок 5) |
| `scanHeight` 0 і баланс 0 | Див. «Якщо в лозі `resetting chain state`» |
| Windows-вузол і Docker-вузол працюють одночасно | Зупиніть один із них: у них один гаманець і одні контракти |
| `pg_restore` пише, що базу використовують інші користувачі | Зупиніть indexd; `DROP DATABASE … WITH (FORCE)` закриває решту з'єднань |
| Хеші контрактів відрізняються під час відкоту | Docker-вузол змінював контракти; Windows-вузол не запускайте |
