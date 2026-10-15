# Перенос узла indexd между Windows и Docker

Это руководство описывает, как перенести работающий узел indexd Smart Autopilot вместе со всеми контрактами из установки на Windows в Docker и как вернуться обратно. Контракты хранятся в блокчейне, а всё, что indexd знает о них (слэбы, аккаунты, ключи приложений, настройки), лежит в базе PostgreSQL. Кошелёк выводится из recovery phrase. Поэтому перенос сводится к двум вещам: та же recovery phrase и база данных.

> **Что проверено.** Windows → Docker: дамп и восстановление базы, форк-миграция при первом запуске образа, тот же адрес и баланс кошелька, те же 13 контрактов. Откат на Windows с проверкой, что контракты не изменились. **Не проверено:** работа Docker-узла с включённой системой в течение долгого времени, подключение s3d к Docker-indexd (проверено только чтение адреса из `s3d.db`) и перенос в обратную сторону (Docker → Windows).

## Правила, которые нельзя нарушать

1. **Один кошелёк — один работающий узел.** Два узла с одной recovery phrase и включённой системой изменяют одни и те же контракты и мешают друг другу. Пока работает один узел, у второго система выключена или он остановлен.
2. **Дамп снимается только с остановленного indexd.**
3. **Docker-узел должен быть синхронизирован и остановлен уже после снятия дампа.** Иначе в его базе цепи может не оказаться последнего блока (см. раздел «Если в логе `resetting chain state`»).
4. **Базу прежнего узла не удаляйте и не меняйте, пока новый не проверен.**

## Что нужно заранее

- Docker-узел установлен по [инструкции](INSTALL-docker.md), в мастере введена **та же recovery phrase**, что у переносимого узла, и он полностью синхронизирован (`synced: true`). Проверка: `GET /api/wallet` у обоих узлов возвращает один и тот же адрес.
- Версия образа **не старше** версии переносимой базы. Миграции форка применяются вперёд при старте, а образ старее базы откажется запускаться с ошибкой `fork schema version N is newer than expected M`.
- Место для дампа (обычно несколько МБ) и для копии `s3d.db`.
- Не затягивайте с переносом: пока система выключена, контракты не продлеваются, а они заканчиваются через несколько недель.

## Перенос Windows → Docker

### Шаг 1. Остановка Windows-узла

Если запущен s3d, отправьте в Sia всё, что ждёт в буфере, и остановите его:

```powershell
cd C:\S3D
.\s3d.exe flush
```

Затем **Ctrl+C** в окне s3d. После этого остановите indexd на Windows (**Ctrl+C**, дождитесь `shutdown complete`). **Docker-узел пока не останавливайте:** он должен работать, пока не снят дамп.

Проверка, что ничего не запущено:

```powershell
Get-Process indexd, s3d -ErrorAction SilentlyContinue
```

Команда не должна ничего вывести.

### Шаг 2. Дамп Windows-базы и контрольные числа

```powershell
$env:PGPASSWORD = "<пароль postgres>"
$w = "E:\Backup\windows-final-$(Get-Date -Format 'yyyyMMdd-HHmm').dump"
& "C:\Program Files\PostgreSQL\18\bin\pg_dump.exe" -U postgres -h 127.0.0.1 -Fc -f $w indexd
Get-Item $w | Select-Object Name, Length
& "C:\Program Files\PostgreSQL\18\bin\psql.exe" -U postgres -h 127.0.0.1 -d indexd -At -c "SELECT (SELECT count(*) FROM contracts), (SELECT count(*) FROM hosts), (SELECT count(*) FROM sectors), (SELECT version FROM fork_schema);"
Remove-Item Env:PGPASSWORD
```

Запишите четыре числа: контракты, хосты, секторы, версия схемы форка. Размер дампа должен быть порядка нескольких мегабайт.

### Шаг 3. Остановка Docker-узла

Теперь, когда дамп снят, остановите indexd в Docker. Дайте ему время завершить запись в базу цепи:

```powershell
cd <папка с docker-compose.yml>
docker compose stop -t 90 indexd
docker compose ps -a indexd
```

Статус должен быть `Exited (0)`. Контейнер PostgreSQL остаётся запущенным (`docker compose ps postgres` показывает `healthy`).

### Шаг 4. Восстановление базы в Docker

Прежде чем стирать Docker-базу, снимите её дамп (порядок описан в разделе «Обновление» [инструкции Docker](INSTALL-docker.md)): он понадобится при откате.

```powershell
docker compose cp $w postgres:/tmp/win.dump
docker compose exec postgres psql -U indexd -d postgres -c "DROP DATABASE indexd WITH (FORCE);"
docker compose exec postgres psql -U indexd -d postgres -c "CREATE DATABASE indexd OWNER indexd;"
docker compose exec postgres pg_restore -U indexd -d indexd --no-owner --no-privileges /tmp/win.dump
docker compose exec postgres rm /tmp/win.dump
docker compose exec postgres psql -U indexd -d indexd -At -c "SELECT (SELECT count(*) FROM contracts), (SELECT count(*) FROM hosts), (SELECT count(*) FROM sectors), (SELECT version FROM fork_schema);"
```

`pg_restore` без сообщений — это нормально. Последняя команда должна вернуть **те же четыре числа**, что и шаг 2. Если схема форка в дампе старше, чем в образе, образ сам доведёт её при первом запуске.

### Шаг 5. Выключить систему до первого запуска

Восстановленные настройки несут состояние Windows-узла, где система была включена. Если запустить Docker-узел как есть, он сразу начнёт формировать контракты и блокировать хосты.

```powershell
docker compose exec postgres psql -U indexd -d indexd -c "UPDATE global_settings SET contracts_maintenance_enabled = false;"
docker compose exec postgres psql -U indexd -d indexd -At -c "SELECT contracts_maintenance_enabled FROM global_settings;"
```

Вторая команда должна вывести `f`.

### Шаг 6. Запуск и проверка

```powershell
docker compose up -d
Start-Sleep 30
docker compose logs --tail 20 indexd
curl.exe -s -m 20 -u ":<пароль admin>" http://127.0.0.1:9980/api/state
curl.exe -s -m 20 -u ":<пароль admin>" http://127.0.0.1:9980/api/wallet
(curl.exe -s -u ":<пароль admin>" "http://127.0.0.1:9980/api/contracts?limit=100" | ConvertFrom-Json) | Group-Object good | Select-Object Name, Count
```

Порт `9980` замените на порт admin вашей установки. Что должно получиться:

- в логе версия образа, при необходимости `fork migration complete` и `node started`, но **нет** строки `resetting chain state`;
- в `api/state`: `synced: true`, `scanHeight` почти равен `syncHeight` (не 0);
- кошелёк: тот же адрес и тот же баланс, что у Windows-узла;
- контракты: то же число, столько же `True` и `False`.

### Шаг 7. Включить систему

Включите переключатель **System is currently disabled** в веб-интерфейсе Docker-узла. Через несколько минут проверьте лог: ошибок быть не должно, а новые контракты формироваться не должны, если цель `Wanted contracts` уже достигнута.

С этого момента считайте перенос окончательным: узел начал работать с контрактами, и откат возможен, только пока он их не менял (раздел «Откат»).

### Шаг 8. Переключение s3d на Docker-indexd

Адрес indexd хранится не в `s3d.yml`, а в базе s3d (`s3d.db`, таблица `global_settings`, поле `indexer_url`), там же ключ приложения. **Не запускайте `s3d login` повторно:** при новой регистрации может создаться новая recovery phrase, и прежние данные перестанут расшифровываться.

Остановите s3d, сделайте копию базы и замените адрес (нужен `sqlite3.exe` с sqlite.org):

```powershell
Copy-Item D:\S3D\data\s3d.db E:\Backup\s3d.db.bak
sqlite3 D:\S3D\data\s3d.db "SELECT indexer_url FROM global_settings;"
sqlite3 D:\S3D\data\s3d.db "UPDATE global_settings SET indexer_url = 'http://localhost:29982';"
```

Порт `29982` — порт Application API Docker-узла на хосте (`INDEXD_APP_PORT`). Запустите s3d: в логе должны быть `erasure coding scheme accepted` и `indexer transfer limits`. Затем прочитайте любой загруженный ранее файл (например, через rclone): это единственная проверка, что данные доступны. *Этот шаг на практике ещё не проверялся: проверено только чтение адреса.*

### Шаг 9. Windows-узел

Чтобы его случайно не запустить рядом с Docker-узлом, переименуйте программу. Базу в Windows-PostgreSQL не удаляйте, она нужна для отката:

```powershell
Rename-Item C:\IndexD\indexd.exe indexd-windows-disabled.exe
```

## Откат (Docker → Windows)

Вернуться можно, **пока Docker-узел не изменил контракты**. Если он их менял (пополнял счета, дописывал секторы, формировал или продлевал контракты), хосты помнят более новые ревизии, чем Windows-база, и запускать Windows-узел нельзя. Проверка: хеш контрактов в обеих базах.

```powershell
$q = "SELECT count(*) AS n, md5(string_agg(j::text, ',' ORDER BY j::text)) AS hash FROM (SELECT (SELECT jsonb_object_agg(k, v) FROM jsonb_each(to_jsonb(c)) AS e(k, v) WHERE k ~ 'revision|remaining|size|capacity|good|state') AS j FROM contracts c) t;"
"Windows:"
$env:PGPASSWORD = "<пароль postgres>"
& "C:\Program Files\PostgreSQL\18\bin\psql.exe" -U postgres -h 127.0.0.1 -d indexd -At -c $q
Remove-Item Env:PGPASSWORD
"Docker:"
docker compose exec postgres psql -U indexd -d indexd -At -c $q
```

Запрос берёт из каждого контракта поля с ревизией, остатком, размером, ёмкостью и статусом (столбцы выбираются по маске) и считает общий хеш. Если число и хеш **совпали**, Docker-узел контрактов не менял, и откат безопасен. Если **различаются**, Windows-узел не запускайте: решайте, остаться ли на Docker-узле (Windows-база устарела).

Порядок отката:

1. Остановите indexd в Docker: `docker compose stop -t 90 indexd`.
2. Верните в Docker прежнюю базу (дамп, снятый на шаге 4), чтобы копия живых контрактов не осталась рядом с Windows-узлом: `DROP DATABASE`, `CREATE DATABASE`, `pg_restore`, как на шаге 4.
3. Верните имя программе Windows-узла: `Rename-Item C:\IndexD\indexd-windows-disabled.exe indexd.exe`.
4. Если меняли адрес в s3d, верните `http://localhost:19982` тем же запросом `UPDATE` или восстановите `s3d.db.bak`.
5. Запустите Windows-indexd. В логе должна быть версия, `node started` и без миграций; `synced: true`, кошелёк с тем же адресом и балансом.

## Перенос в обратную сторону (Docker → Windows)

Выполняется по тем же правилам: Windows-узел подготовлен и синхронизирован, дамп снимается внутри контейнера (`pg_dump -f` и `docker compose cp`, а не перенаправление `>` в PowerShell), база восстанавливается в Windows-PostgreSQL, система выключается до первого запуска. **Это направление не проверялось:** в частности, нужно убедиться, какой ролью восстанавливать базу и что у неё достаточно прав.

## Если в логе `resetting chain state`

Признаки: после запуска `scanHeight` равен 0, баланс кошелька 0, а в логе строка вида `subscriber failed to fetch updates, resetting chain state … missing block at index …`.

Что мы видели: дамп помнит, что кошелёк просканирован до какого-то блока, а в базе цепи Docker-узла этого блока нет (узел был остановлен раньше, чем снят дамп). indexd сбросил состояние и начал пересканировать кошелёк с нуля, что занимает часы. Деньги и контракты при этом не потеряны.

Как исправили (сработало один раз): остановили Docker-indexd штатно (`docker compose stop -t 90 indexd`), восстановили **тот же дамп** заново (шаги 4–5) и запустили: к этому времени цепь Docker-узла уже содержала нужный блок, сброса не было. Чтобы не попадать в эту ситуацию, соблюдайте порядок: Docker-узел работает до конца шага 2 и останавливается только на шаге 3.

## Частые проблемы

| Симптом | Причина и решение |
|---|---|
| `fork schema version N is newer than expected M` при запуске | Образ старее восстановленной базы. Используйте образ не старше базы |
| Сразу после запуска формируются контракты, блокируются хосты | Система не была выключена до первого запуска (шаг 5) |
| `scanHeight` 0 и баланс 0 | См. «Если в логе `resetting chain state`» |
| Windows-узел и Docker-узел работают одновременно | Остановите один из них: у них один кошелёк и одни контракты |
| `pg_restore` пишет, что база используется другими пользователями | Остановите indexd; `DROP DATABASE … WITH (FORCE)` закрывает остальные соединения |
| Хеши контрактов различаются при откате | Docker-узел менял контракты; Windows-узел не запускайте |
