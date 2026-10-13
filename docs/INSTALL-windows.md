# Installing indexd Smart Autopilot and s3d on Windows

This guide covers a native installation without Docker: PostgreSQL 18 first, then indexd Smart Autopilot, followed by s3d Smart Autopilot and, optionally, rclone to mount the storage as a regular drive in Windows File Explorer. The steps have been tested on a working installation, with notes for potential issues.

The examples use the following paths and addresses. You may change them, but update them consistently throughout the guide.

| Item | Path / address |
|---|---|
| indexd (executable and config) | `C:\IndexD` |
| indexd data | `D:\IndexD\data` |
| s3d (executable and config) | `C:\S3D` |
| s3d data (upload buffer) | `D:\S3D\data` |
| indexd ports | admin `9980`, синхронизатор `9981`, application `9982` |
| s3d ports | S3 `8000`, admin `8001` |

## Installation requirements

- 64-bit Windows 10 or 11.
- PostgreSQL 18 for Windows.
- At least 60 GB of free space, excluding the blockchain consensus database, which currently occupies about 89 GB.
- Space for the s3d upload buffer: up to 10 GiB by default.
- Siacoin in the indexd wallet to pay for contracts.
- The `indexd.exe` and `s3d.exe` binaries from [Releases](https://github.com/MainQuestion/indexd-smart-autopilot/releases).

---

## 1. Install PostgreSQL 18

indexd stores all its data—hosts, contracts, slabs, and settings—in PostgreSQL, so start there.

1. Download the PostgreSQL 18 installer for Windows: https://www.postgresql.org/download/windows/ (EDB distribution).
2. In the installer, select **PostgreSQL Server** and **Command Line Tools**; pgAdmin and Stack Builder are not needed.
3. Set a password for the `postgres` superuser and save it; you will need it later.
4. Keep port **5432**.

To verify that everything works, open a new PowerShell window:

```powershell
Get-Service *postgres* | Select-Object Name, Status
& "C:\Program Files\PostgreSQL\18\bin\psql.exe" --version
```

The `postgresql-x64-18` service should be `Running`, and `psql` should report version 18.x.

## 2. Create the indexd database and user

indexd uses its own database user. Choose a **separate** password for it—not the `postgres` password:

```powershell
$psql = "C:\Program Files\PostgreSQL\18\bin\psql.exe"
$env:PGPASSWORD = "<postgres password>"
& $psql -U postgres -h localhost -c "CREATE ROLE indexd WITH LOGIN PASSWORD '<indexd password>';"
& $psql -U postgres -h localhost -c "CREATE DATABASE indexd OWNER indexd;"
$env:PGPASSWORD = "<indexd password>"
& $psql -U indexd -h localhost -d indexd -c "SELECT current_user, current_database();"
Remove-Item Env:PGPASSWORD
```

The last command should return `indexd | indexd`, confirming that the user was created and can access its database. Garbled Cyrillic output is only a console-encoding issue.

## 3. Restrict PostgreSQL network access

The EDB installer often allows PostgreSQL connections from all network interfaces. Since indexd runs on the same computer, it is safer to keep the database server off the network. Run PowerShell **as Administrator**:

```powershell
$env:PGPASSWORD = "<postgres password>"
& "C:\Program Files\PostgreSQL\18\bin\psql.exe" -U postgres -h localhost -c "ALTER SYSTEM SET listen_addresses = 'localhost';"
Restart-Service postgresql-x64-18
Remove-Item Env:PGPASSWORD
netstat -ano | Select-String ':5432'
```

After restarting, `netstat` should show only `127.0.0.1:5432` and `[::1]:5432`.

---

## 4. Set up indexd

Create the directories and place the release's `indexd.exe` in `C:\IndexD`:

```powershell
New-Item -ItemType Directory -Path C:\IndexD, D:\IndexD\data -Force | Out-Null
```

It is convenient to keep the configuration file next to the executable. Tell indexd where to find it:

```powershell
[Environment]::SetEnvironmentVariable('INDEXD_CONFIG_FILE', 'C:\IndexD\indexd.yml', 'User')
$env:INDEXD_CONFIG_FILE = 'C:\IndexD\indexd.yml'
```

> Set the data directory in the setup wizard (next step) and **do not** also set `INDEXD_DATA_DIR`. The environment variable takes precedence, so the path in the configuration file would be silently ignored.

## 5. Initial configuration

The setup wizard asks several questions and creates `indexd.yml`:

```powershell
cd C:\IndexD
.\indexd.exe config
```

| Wizard prompt | What to enter |
|---|---|
| Data directory | `D:\IndexD\data` |
| Recovery phrase (wallet) | Generate a new one and write it on paper. Do not reuse another node's seed |
| Admin API password | Choose a password for signing in to the web interface |
| Application API advertise URL | `http://localhost:9982` — the address applications on this computer use to connect to indexd |
| Admin API Address | `127.0.0.1:9980` — include the host and port; the wizard will not accept just `9980` |
| Application API Address | `127.0.0.1:9982` |
| Database user / name | `indexd` / `indexd` |
| Database password | The **indexd user password**. If you enter the `postgres` password by mistake, indexd will fail to start with `SQLSTATE 28P01` |
| SSL mode | `disable` — local PostgreSQL runs without a certificate, so `verify-full` will not connect |

The wallet seed is stored in plain text in `indexd.yml`. Do not publish this file or sync it to the cloud.

You can check the result without displaying secrets:

```powershell
Get-Content C:\IndexD\indexd.yml | Select-String -NotMatch 'password|seed|recoveryPhrase|mnemonic|key:'
```

## 6. Speed up synchronization (optional)

A full blockchain sync from scratch takes many hours. You can speed it up by using an official snapshot of the chain database (bootstrap):

1. If indexd has already run, stop it with **Ctrl+C** and wait for `shutdown complete`.
2. Extract the snapshot and verify its SHA-256 checksum if one is provided on the download page.
3. Check the chain database filename in the data directory (`Get-ChildItem D:\IndexD\data`), rename the existing file—for example, to `consensus.db.old`—and place the snapshot under the original filename.

If the snapshot is on another drive, create a symbolic link instead of copying it (run cmd as Administrator):

```cmd
mklink "D:\IndexD\data\consensus.db" "X:\путь\к\снимку\consensus.db"
```

> A chain database file can be used by only one node. Do not link to a database that is already open in another program.

## 7. First launch

```powershell
cd C:\IndexD
.\indexd.exe
```

The PowerShell window remains occupied while indexd is running. The log shows the version (`INFO indexd {"version": …}`), the PostgreSQL connection, schema migrations (`migration complete`, `fork migration complete`), and finally `node started`. `waiting for wallet to be scanned` means indexd is catching up with the chain; this is normal.

The web interface opens automatically at http://127.0.0.1:9980.

What to expect during the first few hours:

- **The host list is initially empty.** Hosts appear as scanning progresses; by default, the page shows only usable hosts.
- A single **`indexer is offline, skipping scans`** message indicates a brief network interruption. If it repeats, check access to `1.1.1.1:443`, `www.google.com:443`, and `www.amazon.com:443`; indexd uses these endpoints to check internet connectivity.
- **Your own host on the same LAN may not be scanned.** The router must support NAT loopback; this is not an indexd issue.
- **New hosts start with uptime 0.9, exactly the usability threshold.** On a fresh installation, one missed scan can remove a host from the pool for about a week. Let indexd build up several days of history before enabling the system.

## 8. Initial settings

While the system is disabled, configure the main settings:

1. **Configuration → Pricing.** Set price limits for storage, upload, and download (they can be pegged to USD). Leave a margin above your target prices: new contracts are formed only if the host price is within the buffer—80% of the limit by default. Do not set **Min collateral** too close to the collateral of low-cost hosts: with USD pegging, a small exchange-rate change can put a host below the minimum, causing it to be blocked and all its data to be migrated.
2. **Configuration → Contracts.** Set **Wanted contracts** to at least the total number of shards (N) in the scheme s3d will use, preferably with some margin.
3. **Configuration → Placement.** Configure the price buffer for new contracts, minimum distance between hosts (GeoIP location is only accurate to about city level), radius, and reference point.
4. **Scoring.** Smart mode: `Shadow` only logs what a pause would do; `Active` pauses a host when its price is exceeded instead of blocking it. Configure migration speed limits and, if needed, a redundancy policy below the standard scheme here as well.
5. **Hosts → Candidates.** Review which hosts indexd would select with the current settings and why others were filtered out.
6. **Wallet.** Fund the wallet using the address shown there.
7. Enable the system using the **System is currently disabled** toggle.

After a few minutes, the first contracts should appear on the Contracts page, and the log should show `filtered new contract candidates` with a selection summary.

---

## 9. s3d

s3d is an S3-compatible gateway. Tools such as rclone, Cyberduck, and AWS CLI use it like a regular S3 endpoint; s3d encrypts and encodes the data, then distributes it across hosts with which indexd has contracts.

### 9.1 Place the files

Place the release's `s3d.exe` in `C:\S3D` and create the directories:

```powershell
New-Item -ItemType Directory -Path C:\S3D, D:\S3D\data -Force | Out-Null
[Environment]::SetEnvironmentVariable('S3D_CONFIG_FILE', 'C:\S3D\s3d.yml', 'User')
$env:S3D_CONFIG_FILE = 'C:\S3D\s3d.yml'
```

### 9.2 Application connection key

An application can connect to indexd only with a key issued by the indexer owner. In the indexd web interface, open **Keys** and create an **app connect key**. You will need it shortly in the “App password” field.

### 9.3 Register s3d with indexd

```powershell
cd C:\S3D
.\s3d.exe login
```

- **Data directory** — `D:\S3D\data`.
- **Addresses** — S3 at `127.0.0.1:8000` (or `0.0.0.0:8000` for LAN access), admin at `127.0.0.1:8001`. An admin password is required.
- **Indexer URL** — `http://localhost:9982`. The default is `https://sia.storage`, which points to a third-party indexer.
- **Recovery phrase** — leave it empty and s3d will generate a new one. Write it down; without it, the data cannot be decrypted.
- s3d prints a link such as `…/auth/connect/…`. Open it in a browser, enter the app connect key, and click **Accept**. The connection request expires quickly, so create the key in advance and approve the request immediately.

### 9.4 Create a user and S3 keys

```powershell
.\s3d.exe users create <имя>
.\s3d.exe keys create <имя>
```

The second command returns an Access Key and Secret Key. The Secret Key is shown **only once**; save it in a password manager.

### 9.5 Redundancy scheme

В `C:\S3D\s3d.yml`, в разделе `sia`:

```yaml
    dataShards: 10      # K — шарды с данными
    parityShards: 20    # шарды чётности; всего N = K + parityShards
```

The default scheme is 10-of-30. The minimum allowed scheme for K = 10 is `parityShards: 15` (10-of-25). Below-standard schemes such as 3-of-5 are accepted only when the custom redundancy policy is enabled in indexd (**Scoring → Redundancy**). Each slab requires N distinct hosts eligible for uploads.

### 9.6 Start s3d

```powershell
cd C:\S3D
.\s3d.exe
```

В логе появятся версия, строка `erasure coding scheme accepted` с принятой схемой, `indexer transfer limits` и `server started`. Check upload status from a second window using `.\s3d.exe status`. Admin API s3d в браузере не откроется — он рассчитан на программы; используйте `s3d status` или `curl.exe -u ":<пароль>" http://127.0.0.1:8001/stats/uploads`.

### 9.7 LAN access

If S3 listens on `0.0.0.0:8000`, allow inbound connections in Windows Firewall. The network must be marked **Private** (`Get-NetConnectionProfile` → `NetworkCategory: Private`). Run PowerShell as Administrator:

```powershell
New-NetFirewallRule -DisplayName "s3d S3 API (LAN)" -Direction Inbound -Protocol TCP -LocalPort 8000 -Profile Private -Action Allow
```

Do not forward this port to the internet: S3 uses plain HTTP without encryption.

---

## 10. rclone: mount storage as a drive in File Explorer (optional)

1. Install the WinFsp filesystem driver: `winget install WinFsp.WinFsp`. rclone cannot mount a drive without it.
2. Create a connection (`rclone config` or the rclone web interface): type **S3**, provider **Other**, keys from step 9.4, endpoint `http://127.0.0.1:8000` (or the IP address of the computer running s3d), region `us-east-1`, and `force_path_style = true`. To see empty folders, enable `directory_markers = true`.
3. Bucket names may contain only lowercase Latin letters, digits, and hyphens; underscores are not allowed.
4. Mount the drive with **full VFS caching**, otherwise applications such as Total Commander may report write errors:

   ```powershell
   rclone.exe mount "<подключение>:" S: --vfs-cache-mode full --network-mode --cache-dir D:\rclone\cache
   ```

   You can also use the newer rclone web interface: run `rclone.exe gui --addr 127.0.0.1:5580 --api-addr 127.0.0.1:5572 --user gui --pass <password> --cache-dir D:\rclone\cache`, then mount the drive through its API with `vfsOpt='{"CacheMode":"full"}'`.

A few S3 behaviors may be surprising at first:

- **S3 has no real folders.** A “folder” is a common prefix in object names, so an empty folder is invisible until it contains a file (or `directory_markers` is enabled).
- **Moving means copy and delete.** For large files, this may require downloading data from the Sia network.
- **Small files are not uploaded immediately.** s3d buffers them until a full slab is ready. Force an immediate upload with `s3d flush`.

---

## 11. What to back up separately

Store the following in a secure location, not only on this computer:

- the indexd wallet seed;
- the s3d recovery phrase;
- users' S3 keys;
- regular dumps of the indexd database:
  ```powershell
  & "C:\Program Files\PostgreSQL\18\bin\pg_dump.exe" -U postgres -h 127.0.0.1 -Fc -f E:\Backup_indexd\indexd-<дата>.dump indexd
  ```
- backups of `D:\S3D\data`.

## 12. Updating

1. Stop s3d, then indexd (**Ctrl+C** in their respective windows).
2. Create a database dump: schema migrations during an update are irreversible, and a dump is the only rollback option.
3. Replace `indexd.exe`, start it, and verify that the log shows the new version and `fork migration complete`.
4. Replace `s3d.exe` and start it. If the release also updates the upstream s3d version, back up `D:\S3D\data` first.
5. Hard-refresh the web interface with **Ctrl+Shift+R**.

## 13. Common issues

| Symptom | Cause and solution |
|---|---|
| indexd fails with `SQLSTATE 28P01` | The database password in `indexd.yml` is wrong, usually because the `postgres` password was entered instead of the `indexd` password |
| The wizard reports `missing port in address` | Enter the host and port together: `127.0.0.1:9980` |
| Your host appears without prices or coordinates | indexd is on the same LAN and the router does not support NAT loopback |
| A host left the pool after a power outage | Uptime fell below 0.9; the host should return after about a week of stable operation |
| A host is blocked with reason `Collateral` | Its collateral is below the minimum, often due to exchange-rate changes when pegged to USD |
| `connection request expired` during `s3d login` | Create the connection key in advance and approve the request immediately |
| s3d Admin API returns `Unauthorized` in a browser | The browser does not send the password; use `s3d status` or `curl.exe -u` |
| `InvalidBucketName` when creating a bucket | Bucket names cannot contain underscores |
| Write errors on a mounted drive | The drive was mounted without `--vfs-cache-mode full` |
| Interface sections disappear while navigating menus | The version is outdated; update to the latest release |
