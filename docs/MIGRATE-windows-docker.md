# Moving an indexd node between Windows and Docker

This guide describes how to move a running indexd Smart Autopilot node, with all its contracts, from a Windows installation to Docker, and how to go back. The contracts live on the blockchain, while everything indexd knows about them (slabs, accounts, application keys, settings) is stored in the PostgreSQL database. The wallet is derived from the recovery phrase. So moving a node comes down to two things: the same recovery phrase and the database.

> **What has been tested.** Windows → Docker: dumping and restoring the database, the fork migration on the first start of the image, the same wallet address and balance, the same 13 contracts. Rolling back to Windows, with a check that the contracts had not changed. **Not tested:** running the Docker node with the system enabled for a long time, connecting s3d to the Docker indexd (only reading the address from `s3d.db` was checked), and moving in the opposite direction (Docker → Windows).

## Rules you must not break

1. **One wallet, one running node.** Two nodes with the same recovery phrase and the system enabled change the same contracts and interfere with each other. While one node runs, the system on the other is off or the node is stopped.
2. **Take the dump only from a stopped indexd.**
3. **The Docker node must be synchronized and stopped only after the dump is taken.** Otherwise its chain database may lack the latest block (see "If the log shows `resetting chain state`").
4. **Do not delete or change the database of the previous node until the new one is verified.**

## What you need beforehand

- The Docker node is installed following the [guide](INSTALL-docker.en.md), the **same recovery phrase** as the node you are moving was entered in the wizard, and it is fully synchronized (`synced: true`). Check: `GET /api/wallet` returns the same address on both nodes.
- The image version is **not older** than the version of the database you are moving. The fork's migrations are applied forward at start, while an image older than the database refuses to start with `fork schema version N is newer than expected M`.
- Space for the dump (usually a few MB) and for a copy of `s3d.db`.
- Do not postpone the move: while the system is off, contracts are not renewed, and they expire in a few weeks.

## Moving Windows → Docker

### Step 1. Stop the Windows node

If s3d is running, upload everything waiting in its buffer to Sia and stop it:

```powershell
cd C:\S3D
.\s3d.exe flush
```

Then press **Ctrl+C** in the s3d window. After that, stop indexd on Windows (**Ctrl+C**, wait for `shutdown complete`). **Do not stop the Docker node yet:** it must keep running until the dump is taken.

Check that nothing is running:

```powershell
Get-Process indexd, s3d -ErrorAction SilentlyContinue
```

The command should print nothing.

### Step 2. Dump the Windows database and note the control numbers

```powershell
$env:PGPASSWORD = "<postgres password>"
$w = "E:\Backup\windows-final-$(Get-Date -Format 'yyyyMMdd-HHmm').dump"
& "C:\Program Files\PostgreSQL\18\bin\pg_dump.exe" -U postgres -h 127.0.0.1 -Fc -f $w indexd
Get-Item $w | Select-Object Name, Length
& "C:\Program Files\PostgreSQL\18\bin\psql.exe" -U postgres -h 127.0.0.1 -d indexd -At -c "SELECT (SELECT count(*) FROM contracts), (SELECT count(*) FROM hosts), (SELECT count(*) FROM sectors), (SELECT version FROM fork_schema);"
Remove-Item Env:PGPASSWORD
```

Write down the four numbers: contracts, hosts, sectors, fork schema version. The dump should be a few megabytes.

### Step 3. Stop the Docker node

Now that the dump is taken, stop indexd in Docker. Give it time to finish writing to the chain database:

```powershell
cd <folder with docker-compose.yml>
docker compose stop -t 90 indexd
docker compose ps -a indexd
```

The status should be `Exited (0)`. The PostgreSQL container stays running (`docker compose ps postgres` shows `healthy`).

### Step 4. Restore the database in Docker

Before wiping the Docker database, take a dump of it (see the "Updating" section of the [Docker guide](https://github.com/MainQuestion/indexd-smart-autopilot/blob/main/docs/docker/INSTALL%26UPDATE-docker.md)): you will need it for a rollback.

```powershell
docker compose cp $w postgres:/tmp/win.dump
docker compose exec postgres psql -U indexd -d postgres -c "DROP DATABASE indexd WITH (FORCE);"
docker compose exec postgres psql -U indexd -d postgres -c "CREATE DATABASE indexd OWNER indexd;"
docker compose exec postgres pg_restore -U indexd -d indexd --no-owner --no-privileges /tmp/win.dump
docker compose exec postgres rm /tmp/win.dump
docker compose exec postgres psql -U indexd -d indexd -At -c "SELECT (SELECT count(*) FROM contracts), (SELECT count(*) FROM hosts), (SELECT count(*) FROM sectors), (SELECT version FROM fork_schema);"
```

`pg_restore` printing nothing is normal. The last command must return **the same four numbers** as step 2. If the fork schema in the dump is older than the one in the image, the image brings it up to date on the first start.

### Step 5. Turn the system off before the first start

The restored settings carry the state of the Windows node, where the system was on. If you start the Docker node as is, it immediately begins forming contracts and blocking hosts.

```powershell
docker compose exec postgres psql -U indexd -d indexd -c "UPDATE global_settings SET contracts_maintenance_enabled = false;"
docker compose exec postgres psql -U indexd -d indexd -At -c "SELECT contracts_maintenance_enabled FROM global_settings;"
```

The second command must print `f`.

### Step 6. Start and check

```powershell
docker compose up -d
Start-Sleep 30
docker compose logs --tail 20 indexd
curl.exe -s -m 20 -u ":<admin password>" http://127.0.0.1:9980/api/state
curl.exe -s -m 20 -u ":<admin password>" http://127.0.0.1:9980/api/wallet
(curl.exe -s -u ":<admin password>" "http://127.0.0.1:9980/api/contracts?limit=100" | ConvertFrom-Json) | Group-Object good | Select-Object Name, Count
```

Replace port `9980` with the admin port of your installation. What you should see:

- in the log, the image version, `fork migration complete` if needed, and `node started`, but **no** `resetting chain state` line;
- in `api/state`: `synced: true`, and `scanHeight` almost equal to `syncHeight` (not 0);
- the wallet: the same address and the same balance as on the Windows node;
- the contracts: the same number, with the same counts of `True` and `False`.

### Step 7. Turn the system on

Switch on **System is currently disabled** in the web interface of the Docker node. After a few minutes check the log: there should be no errors, and no new contracts should be formed if the `Wanted contracts` target is already met.

From this point treat the move as final: the node has started working with the contracts, and a rollback is only possible while it has not changed them (see "Rollback").

### Step 8. Point s3d at the Docker indexd

The indexd address is stored not in `s3d.yml` but in the s3d database (`s3d.db`, table `global_settings`, field `indexer_url`), together with the application key. **Do not run `s3d login` again:** a new registration may create a new recovery phrase, and the existing data would no longer decrypt.

Stop s3d, copy the database and replace the address (you need `sqlite3.exe` from sqlite.org):

```powershell
Copy-Item D:\S3D\data\s3d.db E:\Backup\s3d.db.bak
sqlite3 D:\S3D\data\s3d.db "SELECT indexer_url FROM global_settings;"
sqlite3 D:\S3D\data\s3d.db "UPDATE global_settings SET indexer_url = 'http://localhost:29982';"
```

Port `29982` is the host port of the Docker node's Application API (`INDEXD_APP_PORT`). Start s3d: the log should show `erasure coding scheme accepted` and `indexer transfer limits`. Then read any file uploaded earlier (for example through rclone): this is the only check that the data is available. *This step has not yet been tried in practice: only reading the address was checked.*

### Step 9. The Windows node

To avoid starting it next to the Docker node by accident, rename the program. Do not delete the database in Windows PostgreSQL: you need it for a rollback:

```powershell
Rename-Item C:\IndexD\indexd.exe indexd-windows-disabled.exe
```

## Rollback (Docker → Windows)

You can go back **as long as the Docker node has not changed the contracts**. If it did (funded accounts, appended sectors, formed or renewed contracts), the hosts remember newer revisions than the Windows database has, and the Windows node must not be started. Check: the contracts hash in both databases.

```powershell
$q = "SELECT count(*) AS n, md5(string_agg(j::text, ',' ORDER BY j::text)) AS hash FROM (SELECT (SELECT jsonb_object_agg(k, v) FROM jsonb_each(to_jsonb(c)) AS e(k, v) WHERE k ~ 'revision|remaining|size|capacity|good|state') AS j FROM contracts c) t;"
"Windows:"
$env:PGPASSWORD = "<postgres password>"
& "C:\Program Files\PostgreSQL\18\bin\psql.exe" -U postgres -h 127.0.0.1 -d indexd -At -c $q
Remove-Item Env:PGPASSWORD
"Docker:"
docker compose exec postgres psql -U indexd -d indexd -At -c $q
```

The query takes the revision, remaining allowance, size, capacity and status fields of every contract (the columns are picked by a mask) and computes one overall hash. If the count and the hash **match**, the Docker node has not changed the contracts, and the rollback is safe. If they **differ**, do not start the Windows node: decide whether to stay on the Docker node (the Windows database is stale).

The rollback steps:

1. Stop indexd in Docker: `docker compose stop -t 90 indexd`.
2. Restore the previous database in Docker (the dump taken in step 4), so that a copy of the live contracts is not left next to the Windows node: `DROP DATABASE`, `CREATE DATABASE`, `pg_restore`, as in step 4.
3. Give the Windows node program its name back: `Rename-Item C:\IndexD\indexd-windows-disabled.exe indexd.exe`.
4. If you changed the address in s3d, set `http://localhost:19982` back with the same `UPDATE`, or restore `s3d.db.bak`.
5. Start the Windows indexd. The log should show the version and `node started`, with no migrations; `synced: true`, and the wallet with the same address and balance.

## Moving in the opposite direction (Docker → Windows)

The same rules apply: the Windows node is prepared and synchronized, the dump is taken inside the container (`pg_dump -f` and `docker compose cp`, not the `>` redirection of PowerShell), the database is restored into Windows PostgreSQL, and the system is turned off before the first start. **This direction has not been tested:** in particular, you need to confirm which role to restore the database with and that it has enough privileges.

## If the log shows `resetting chain state`

Symptoms: after the start `scanHeight` is 0, the wallet balance is 0, and the log has a line like `subscriber failed to fetch updates, resetting chain state … missing block at index …`.

What we saw: the dump remembers that the wallet was scanned up to a certain block, but the chain database of the Docker node does not have that block (the node was stopped before the dump was taken). indexd reset its state and started rescanning the wallet from zero, which takes hours. The funds and contracts are not lost.

How we fixed it (it worked once): we stopped the Docker indexd cleanly (`docker compose stop -t 90 indexd`), restored **the same dump** again (steps 4–5) and started it: by then the chain of the Docker node already contained the needed block, and there was no reset. To avoid this situation, keep the order: the Docker node runs until the end of step 2 and is stopped only in step 3.

## Common problems

| Symptom | Cause and solution |
|---|---|
| `fork schema version N is newer than expected M` at start | The image is older than the restored database. Use an image that is not older than the database |
| Contracts are formed and hosts blocked right after the start | The system was not turned off before the first start (step 5) |
| `scanHeight` 0 and balance 0 | See "If the log shows `resetting chain state`" |
| The Windows node and the Docker node run at the same time | Stop one of them: they share one wallet and the same contracts |
| `pg_restore` says the database is being used by other users | Stop indexd; `DROP DATABASE … WITH (FORCE)` closes the remaining connections |
| The contract hashes differ during a rollback | The Docker node changed the contracts; do not start the Windows node |
