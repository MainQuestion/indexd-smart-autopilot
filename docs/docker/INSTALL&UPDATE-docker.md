# Installing and updating indexd Smart Autopilot in Docker

In this setup indexd runs in two containers: PostgreSQL 18 and indexd itself from a ready-made image. You need no source code, Go or build tools: installing and updating come down to a few `docker compose` commands. The image is built for Linux (`linux/amd64`) and is about 58 MB.

The image contains only indexd. The s3d gateway is not shipped in Docker yet: install it separately as before (see the [Windows guide](INSTALL-windows.md), section 9) and point it at indexd using the address from step 1.4.

The image is hosted in the GitHub registry: `ghcr.io/mainquestion/indexd-smart-autopilot:<version>`. The image version is the upstream version followed by the short commit hash of the fork, for example `v0.2.0-alpha.1`. The list of versions is in the Releases section of the repository.

## Requirements

- Docker Desktop (Windows, WSL2 mode) or Docker Engine on Linux, with Docker Compose v2.
- About 90 GB for the blockchain database plus at least 60 GB of free space on top of it. Docker Desktop keeps volumes inside its own virtual disk; if the system drive is small, move that disk in Docker Desktop: **Settings → Resources → Advanced → Disk image location**.
- Siacoin in the indexd wallet to pay for contracts.
- The `docker-compose.yml` and `.env.example` files from the repository's `docker` folder.

---

## 1. Installing from scratch

### 1.1 Preparation

Create an empty folder, put `docker-compose.yml` and `.env.example` in it, then copy the example settings:

```powershell
Copy-Item .env.example .env
notepad .env
```

`.env` has two required values: `POSTGRES_PASSWORD`, the database password (make it long), and `INDEXD_VERSION`, the image version. There are no defaults: until you set them, compose refuses to start. The `.env` file contains a password, so do not publish it or sync it to the cloud.

You do not need to create data folders: compose uses the **named volumes** `indexd-data` and `postgres-data`, and Docker places them itself.

> Do not replace the named volumes with Windows folders (such as `D:\...:/data`). When we tried that, the blockchain database on the mounted folder was corrupted (`panic: page … already freed`). The cause is not proven, but named volumes are more reliable: the data lives in the native Linux file system inside WSL2.

### 1.2 Pulling the images

```powershell
docker compose pull
```

### 1.3 Recovery phrase

```powershell
docker compose run --rm indexd seed
```

The command prints a new wallet recovery phrase. Write it down on paper and do not reuse the phrase of another node. Without it you will have no access to the funds or the contracts.

### 1.4 Setup wizard

```powershell
docker compose run --rm -it indexd config
```

| Wizard question | What to answer |
|---|---|
| Recovery phrase | The phrase from the previous step |
| Admin API password | Choose one — it is the password for the web interface |
| Application API advertise URL | `http://localhost:9982` — the address applications on this computer will use to connect to indexd (s3d running on the same computer) |
| Database host / port | `postgres` / `5432` — the name of the database container |
| Database user / name | `indexd` / `indexd` |
| Database password | The same as `POSTGRES_PASSWORD` in `.env` |
| SSL mode | `disable` — the database is in a neighbouring container and is not published outside |

Leave the listen addresses as the wizard suggests: the container itself listens on `:9980` (admin), `:9981` (syncer) and `:9982` (application), and compose publishes these ports to the outside.

The configuration is saved in the `indexd-data` volume (`/data/indexd.yml`) and contains the seed in plain text. Do not copy it to public places.

### 1.5 Starting and checking

```powershell
docker compose up -d
docker compose logs -f indexd
```

The log shows the image version, the PostgreSQL connection, migrations (`migration complete`, `fork migration complete`) and `node started`. Press **Ctrl+C** to leave the log view; this does not affect the node.

Checking the version:

```powershell
curl.exe -s -u ":<admin password>" http://127.0.0.1:9980/api/state
```

The response should contain your version and `"os":"linux"`. The web interface opens at http://127.0.0.1:9980.

First indexd synchronizes the blockchain, which takes many hours. Until `synced` becomes `true` and `scanHeight` catches up with `syncHeight`, no contracts are formed (the log says `waiting for wallet to be scanned`).

### 1.6 What next

The initial settings (prices, Wanted contracts, Placement, Scoring) are the same as for a Windows installation: section 8 of [that guide](INSTALL-windows.md).

### Ports

| Port | Purpose | Default exposure |
|---|---|---|
| 9980 | Admin API and web interface | `127.0.0.1` only |
| 9981 | Syncer (communication with other nodes) | all interfaces: otherwise the node accepts outgoing connections only |
| 9982 | Application API (for s3d and other applications) | `127.0.0.1` only |

The host ports can be changed in `.env` (`INDEXD_ADMIN_PORT`, `INDEXD_SYNCER_PORT`, `INDEXD_APP_PORT`). Exposing the admin port to the network is not recommended: it controls the wallet.

---

## 2. Updating to a new fork version

The fork's database schema migrations are applied forward and **cannot be rolled back**. You can return to the previous version only from a dump, so always take one.

1. **Database dump.** It is made inside the container and copied out. The `>` redirection in PowerShell corrupts binary files, so do not use it:

   ```powershell
   $d = Get-Date -Format 'yyyyMMdd-HHmm'
   docker compose exec postgres pg_dump -U indexd -Fc -f /tmp/indexd.dump indexd
   docker compose cp postgres:/tmp/indexd.dump ".\indexd-$d.dump"
   docker compose exec postgres rm /tmp/indexd.dump
   Get-Item ".\indexd-$d.dump" | Select-Object Name, Length
   ```

   The file must not be empty. Keep the dump outside Docker's data folder.

2. **Stopping.** Give indexd time to finish writing to the blockchain database: if the process is killed in the middle of a write, the database may get corrupted.

   ```powershell
   docker compose stop -t 90 indexd
   ```

3. **New version.** In `.env` replace `INDEXD_VERSION` with the new one and pull the image:

   ```powershell
   docker compose pull
   ```

4. **Starting and checking.**

   ```powershell
   docker compose up -d
   docker compose logs -f indexd
   curl.exe -s -u ":<admin password>" http://127.0.0.1:9980/api/state
   ```

   The log should show the new version and `fork migration complete` (if the version has new migrations), and the `api/state` response the same version. Then refresh the web interface page with **Ctrl+Shift+R**.

**Updating from alpha.1.** The fork migrations (8 to 13) are applied in one start: the log shows one `fork migration complete` line per migration, the last one with `forkVersion 13`. The first start after the update may take a little longer than usual.

**s3d.** If you use s3d, update it after indexd to `v0.2.0-alpha.1`: the new s3d sends transfer speed samples; without them the Performance score of the hosts stays at 1 (the other scores work). The new s3d works as before with an older indexd.

The data in the volumes (chain database, PostgreSQL, configuration) is preserved during the update.

---

## 3. Switching from the upstream image to the fork

If you already run indexd from the `ghcr.io/siafoundation/indexd` image, you can switch to the fork on the same database: this path has been tested. The upstream version the fork is based on must not be newer than yours.

1. Take a dump (section 2, step 1) and copy `indexd.yml` to a safe place.
2. Stop indexd: `docker compose stop -t 90 indexd`.
3. In `docker-compose.yml`, in the `indexd` service, replace the `image:` line with `ghcr.io/mainquestion/indexd-smart-autopilot:<version>`. While you are there, check the admin port line: it should look like `"127.0.0.1:9980:9980/tcp"`, not `"0.0.0.0:…"`.
4. Run `docker compose up -d` and check as in section 2, step 4. The log will show the fork migrations from the first to the last.

After that, **do not go back** to the upstream image: it does not know about the fork's tables. To go back, restore the database from the dump.

---

## 4. What to keep separately

- the wallet recovery phrase;
- database dumps (regularly and before every update);
- the `indexd.yml` file from the volume (it contains the seed, treat it like a password): `docker compose cp indexd:/data/indexd.yml .\indexd.yml`;
- the `.env` file.

The blockchain database (`consensus.db`, about 90 GB) does not need to be backed up: it is restored by synchronization.

---

## 5. Common problems

| Symptom | Cause and solution |
|---|---|
| `panic: page … already freed` in the log | The blockchain database (`consensus.db`) is corrupted. Most often the data was on a Windows folder mounted into the container. Use named volumes; delete the corrupted `consensus.db` and synchronization will start again |
| `docker compose pull` says `not found` or `denied` | Wrong version in `.env`, or the image is not yet published in the registry |
| In Git Bash `pg_dump` fails on the path `/tmp/...` | Git Bash converts paths to Windows paths. Run the command with `MSYS_NO_PATHCONV=1` or use PowerShell |
| `docker compose exec indexd sh` does not work | The image has no shell. Take files out with `docker compose cp` |
| indexd does not start after going back to the upstream image | The database has already been migrated by the fork. Restore it from the dump |
| Admin is not reachable from another computer | By design: the port listens on `127.0.0.1` only |
| Running out of disk space | The blockchain database takes most of it; check the location of the Docker Desktop virtual disk |
| The `/api/wallet` request does not respond while the others work | For us this once cleared up by itself after a restart (`docker compose stop -t 90 indexd`, then `up -d`); the cause is not established |
