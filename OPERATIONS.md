# Production Operations

## Start with automatic restart

Install PM2 once on the server, then run:

```bash
npm install
pm2 start ecosystem.config.cjs
pm2 save
pm2 startup
```

## Zero-downtime reload

After deploying new code:

```bash
npm install
pm2 reload hybrid-parking
```

`reload` rotates clustered workers instead of stopping the service first.

Keep `instances: 1`. Kasir live updates (SSE), cache, and login attempt limits live in process memory; several instances would split them.

If MariaDB is not ready at boot, the server retries the database every 5 seconds and only starts listening once the tables are ready.

Set `USER_ID_SECRET` (a long random string) in `.env` on production. Without it, encrypted IDs use a built-in default key.

## Database setup

Keep credentials only in `.env`. Do not commit that file.

```bash
mariadb -h "$DB_HOST" -P "$DB_PORT" -u "$DB_USER" -p"$DB_PASSWORD" < database.sql
```

## Health check

```bash
curl http://127.0.0.1:3000/api/health
```

The API also applies request timeouts, security headers, and a per-IP rate limit. Put the app behind HTTPS at the reverse proxy in production.

## Local-only pages

The gate API (`/api/gate/*`) and the cashier pages (`/kasir`, `/kasir/*`, `/api/kasir/*`) only answer requests made to the server's IP from the same local subnet. Requests through a domain or from outside get `403 Access denied`. Disable with `GATE_LOCAL_ONLY=off` or `KASIR_LOCAL_ONLY=off` in `.env`.

## Monitor Server (Owner only)

Profile menu → **Monitor Server** shows disk, RAM, CPU, and database size, and offers a database backup download plus server restart/shutdown. Only Owner accounts can open it. Every action needs the CSRF token and the Owner's password again (5 wrong passwords lock actions for 5 minutes). Restart/shutdown also need the word `RESTART` / `SHUTDOWN` typed in, and run 5 seconds after confirmation. Each action is written to the server log with user ID and IP.

Restart/shutdown run `systemctl reboot|poweroff` directly when systemd-logind lets the app user do so without a password (`CanReboot`/`CanPowerOff` = `yes`, normally when that user is logged in on the server machine itself, e.g. desktop session). Otherwise (PM2 at boot, SSH-only servers) they fall back to `sudo -n`; allow only those two commands for the user that runs the app (replace `niagait`):

```bash
echo 'niagait ALL=(root) NOPASSWD: /usr/bin/systemctl reboot, /usr/bin/systemctl poweroff' | sudo tee /etc/sudoers.d/hybrid-parking-power
sudo chmod 440 /etc/sudoers.d/hybrid-parking-power
sudo visudo -c
```

Without this rule the buttons stay disabled. Set `SERVER_POWER=local` in `.env` to allow restart/shutdown only from the local network through the server IP, or `SERVER_POWER=off` to turn them off. Backup uses `mariadb-dump` (or `mysqldump`; override with `DB_DUMP_BIN`) and the database credentials in `.env`.
