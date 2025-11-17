# Setting up automatic PostgreSQL backups using cron

This guide shows how to schedule automated PostgreSQL backups with cron, compress the dumps, keep a retention policy, and secure credentials. Replace placeholders (database names, paths, usernames, passwords, retention days) with values for your environment.

---

## Overview

- Create a backup directory with correct ownership and permissions.
- Use a small shell script to run pg_dump, compress the output, and rotate old backups.
- Schedule the script in the postgres user's crontab (or root crontab referencing the postgres user).
- Test manually and verify logs and file permissions.
- Prefer using a .pgpass file (or IAM/role-based access) rather than storing cleartext passwords in scripts.

---

## 1) Prepare backup directory and permissions

Example:
```bash
# create backup dir and set secure ownership/permissions
sudo mkdir -p /backups/postgres
sudo chown postgres:postgres /backups/postgres
sudo chmod 700 /backups/postgres
```

---

## 2) Use .pgpass for secure password storage (recommended)

On the postgres user account, create `~/.pgpass`:
```
# format: hostname:port:database:username:password
localhost:5432:*:postgres:YOUR_POSTGRES_PASSWORD
```
Set permissions:
```bash
sudo -u postgres bash -c 'echo "localhost:5432:*:postgres:YOUR_POSTGRES_PASSWORD" > ~/.pgpass && chmod 600 ~/.pgpass'
```
Notes:
- .pgpass allows pg_dump/pSQL tools to authenticate non-interactively without exporting PGPASSWORD into the environment.
- Keep this file readable only by the postgres user (chmod 600).

---

## 3) Create a backup script (recommended)

Create `/usr/local/bin/pg_backup.sh` (owned by root) and make it executable. Example script:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Config - edit these
BACKUP_DIR="/backups/postgres"
DB_NAME="styraiipl_apdcl_test"        # change to your DB name, or loop over multiple DBs
DB_USER="postgres"
DB_HOST="localhost"
RETENTION_DAYS=30                     # remove backups older than this

# Timestamp for filename
TIMESTAMP="$(date +'%Y%m%d_%H%M%S')"
OUTFILE="${BACKUP_DIR}/${DB_NAME}_${TIMESTAMP}.sql.gz"

# Ensure backup dir exists and permissions are correct
mkdir -p "${BACKUP_DIR}"
chown postgres:postgres "${BACKUP_DIR}"
chmod 700 "${BACKUP_DIR}"

# Run pg_dump as postgres user and compress
# Uses .pgpass if present in postgres home
sudo -u postgres pg_dump -h "${DB_HOST}" -U "${DB_USER}" "${DB_NAME}" | gzip -9 > "${OUTFILE}"

# Rotate old backups
find "${BACKUP_DIR}" -type f -name "${DB_NAME}_*.sql.gz" -mtime +"${RETENTION_DAYS}" -print -delete

# Optional: set ownership of the new file to postgres
chown postgres:postgres "${OUTFILE}"
```

Install the script and set permissions:
```bash
sudo tee /usr/local/bin/pg_backup.sh > /dev/null <<'EOF'
# (paste the script contents here)
EOF

sudo chmod 750 /usr/local/bin/pg_backup.sh
sudo chown root:root /usr/local/bin/pg_backup.sh
```

Notes:
- Running pg_dump via `sudo -u postgres` ensures the dump runs with the postgres user context; adjust if you prefer another user.
- If you have multiple DBs, the script can loop through a list of database names.

---

## 4) Add a cronjob for automated scheduling

Edit the postgres user's crontab. Use one of these options:

- Switch to postgres user and edit its crontab:
```bash
sudo -i -u postgres crontab -e
```

- Or edit root's crontab and run the script as postgres:
```bash
sudo crontab -e
# add:
0 2 * * * /usr/local/bin/pg_backup.sh >> /var/log/pg_backup.log 2>&1
```

Example (in postgres crontab) to run daily at 02:00:
```
# Run backup daily at 02:00
0 2 * * * /usr/local/bin/pg_backup.sh >> /var/log/pg_backup.log 2>&1
```

Notes:
- Redirecting stdout/stderr to a log helps debugging: `/var/log/pg_backup.log` should be writable by the user running the cronjob (root if run from root crontab, or configure log rotation).
- Use a specific crontab for the `postgres` user so the job runs in the postgres context without sudo, or keep it in root crontab and call `sudo -u postgres`.

---

## 5) One-line pg_dump examples (quick, less flexible)

If you prefer a one-liner in crontab (less preferred vs script):

- Without compression:
```
0 2 * * * sudo -u postgres pg_dump -U postgres styraiipl_apdcl_test > /backups/postgres/styraiipl_$(date +\%F).sql
```

- With gzip compression and date in filename:
```
0 2 * * * sudo -u postgres sh -c 'pg_dump -U postgres styraiipl_apdcl_test | gzip -9 > /backups/postgres/styraiipl_$(date +\%F_%H%M%S).sql.gz'
```

Remember to escape percent signs in crontab or wrap the command in `sh -c '...'` so `$(date ...)` runs correctly.

---

## 6) Test the script and cron job

- Run manually to verify:
```bash
sudo /usr/local/bin/pg_backup.sh
# or as postgres user
sudo -i -u postgres /usr/local/bin/pg_backup.sh
```

- Check created file and permissions:
```bash
ls -lh /backups/postgres
file /backups/postgres/styraiipl_*.sql.gz
```

- Verify crontab entry:
```bash
# show postgres crontab
sudo -i -u postgres crontab -l
```

- Check logs if using `pg_backup.log`:
```bash
sudo tail -n 200 /var/log/pg_backup.log
```

---

## 7) Optional enhancements & best practices

- Use WAL archiving (continuous backup) and base backups for point-in-time recovery when needed.
- Consider `pg_basebackup` for full base backups (useful with WAL).
- For production, push backups off-host (S3, object store, or another server) using `aws s3 cp` or similar inside the script after successful dump.
- Use GPG to encrypt backups before transfer:
  - Example: `gpg --encrypt --recipient backup@example.com "${OUTFILE}"`
- Monitor backup success with alerts (email, PagerDuty, monitoring system).
- Keep logs rotated (use logrotate) and do not leave secrets in scripts. Prefer `.pgpass` or cloud IAM roles where applicable.

---

## 8) Troubleshooting

- Permission errors: check directory ownership and file permissions (must be readable/writable by the user running the job).
- Authentication errors: verify `.pgpass` format/permissions or that the PGPASSWORD/ENV are set properly.
- Cron environment: cron runs with a limited environment. Always use absolute paths in scripts and crontab entries (e.g., `/usr/bin/pg_dump`, `/bin/gzip` if necessary).
- Date evaluation in crontab: percent (%) characters are special in crontab; prefer wrapping date evaluation in a script.

---

## Example summary: minimal working setup

1. Prepare folder:
```bash
sudo mkdir -p /backups/postgres
sudo chown postgres:postgres /backups/postgres
sudo chmod 700 /backups/postgres
```

2. Create `.pgpass` for postgres:
```bash
sudo -i -u postgres bash -c 'printf "localhost:5432:*:postgres:%s\n" "YOUR_POSTGRES_PASSWORD" > ~/.pgpass && chmod 600 ~/.pgpass'
```

3. Install script `/usr/local/bin/pg_backup.sh` (as above) and make it executable.

4. Add cron entry (postgres crontab):
```bash
sudo -i -u postgres crontab -l 2>/dev/null; echo "0 2 * * * /usr/local/bin/pg_backup.sh >> /var/log/pg_backup.log 2>&1" | sudo -i -u postgres crontab -
```

5. Test:
```bash
sudo -i -u postgres /usr/local/bin/pg_backup.sh
ls -lh /backups/postgres
```

---

Replace all example values and password placeholders. Take a snapshot or backup before testing on production databases.
