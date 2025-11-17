# N8N Installation and configuration using Docker

This document describes installing Docker on Ubuntu, preparing an n8n project directory, configuring n8n with a Postgres database, running n8n with Docker Compose, configuring NGINX (with and without a subpath), basic Docker commands, checking workflows in Postgres, installing Ollama, and troubleshooting 2FA issues.

---

## 1) Install Docker on Ubuntu

Update the system
```bash
sudo apt update && sudo apt upgrade -y
```

Install dependencies
```bash
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common gnupg
```

Add Docker's official GPG key
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/docker.gpg
```

Add Docker's official APT repo
```bash
echo "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install Docker CE and Compose plugin
```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

Enable and start Docker
```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Test Docker
```bash
docker --version
```

---

## 2) Create n8n Project Directory and .env file

Create the project directory and open the .env file for editing:
```bash
mkdir -p ~/n8n
cd ~/n8n
nano .env
```

Example `.env` contents (edit placeholders to your environment):
```env
N8N_HOST=https://<URL>
N8N_PORT=5678
N8N_PROTOCOL=https

# This tells n8n it is served from a subpath
N8N_PATH=<sub-path-name>

# These must also use the subpath
WEBHOOK_URL=https://<URL>/<Sub-path-URL>/
N8N_EDITOR_BASE_URL=https://<URL>/<Sub-path-URL>/
N8N_API_URL=https://<URL>/<Sub-path-URL>/

# Authentication
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=m0N1Y|>d!bkX9>|

# Cookie secure flag (should be true if using HTTPS)
N8N_SECURE_COOKIE=true

# Database config
DB_TYPE=postgresdb
DB_POSTGRESDB_HOST=<database-server-IP>
DB_POSTGRESDB_PORT=5432
DB_POSTGRESDB_DATABASE=<database-name>
DB_POSTGRESDB_USER=<database-username>
DB_POSTGRESDB_PASSWORD=<database-password>

# Runner
N8N_RUNNERS_ENABLED=true
```

NOTE: Create the PostgreSQL database and user beforehand; use the same credentials in the `.env` file.

---

## 3) Create Docker Compose File

Create `docker-compose.yml`:
```bash
nano docker-compose.yml
```

Example `docker-compose.yml`:
```yaml
version: "3.7"

services:
  n8n:
    image: n8nio/n8n
    ports:
      - "5678:5678"
    env_file:
      - .env   # Load ALL variables from .env into the container
    volumes:
      - n8n_data:/home/node/.n8n
      - /root/n8n/custom-nodes:/home/node/.n8n/custom
    restart: unless-stopped

volumes:
  n8n_data:
```

---

## 4) Start n8n

Start the service in the background:
```bash
docker compose up -d
```

Check running containers:
```bash
docker ps
```

Get n8n version inside the container:
```bash
docker exec -it <container_id> n8n --version
```

If the database is on a different server, verify connectivity from inside the container:
```bash
docker exec -it <container_id> sh
# inside container shell:
nc -zv <database_IP> <database_port>   # Should show open
```

---

## 5) Docker service & common commands

Systemd service control
```bash
sudo systemctl start docker
sudo systemctl stop docker
sudo systemctl restart docker
sudo systemctl enable docker
sudo systemctl status docker
```

Containers
```bash
docker ps
docker ps -a
docker start <container_id>
docker stop <container_id>
docker restart <container_id>
docker rm <container_id>
docker inspect <container_id>
docker stats
```

Logs & exec
```bash
docker logs <container_id>
docker logs -f <container_id>
docker logs --tail 100 <container_id>
docker exec -it <container_id> sh
docker exec -it <container_id> bash
docker exec -it <container_id> <command>
```

Images
```bash
docker images
docker rmi <image_id>
docker pull <image_name>
docker build -t myimage .
docker history <image_id>
```

Volumes & networks
```bash
docker volume ls
docker volume inspect <volume>
docker volume rm <volume>

docker network ls
docker network inspect <network>
docker network rm <network>
```

Docker Compose
```bash
docker compose up -d
docker compose down
docker compose restart
docker compose logs -f
docker compose ps
docker compose exec <svc> sh
```

---

## 6) Configure NGINX for n8n (SSL required)

NGINX config without subpath:
```nginx
server {
  server_name <Sub-Domain>;
  access_log /var/log/nginx/n8n-api-access.log;
  error_log /var/log/nginx/n8n-api-error.log;

  location / {
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Server $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_pass http://<Server-IP>:5678;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;
  }
}
```

NGINX config with subpath (example using `/sflow/`; add inside your SSL server block):
```nginx
# n8n served under /sflow/
location /sflow/ {
  rewrite ^/sflow/(.*)$ /$1 break;
  proxy_pass http://<Server-IP>:5678;
  proxy_http_version 1.1;

  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection 'upgrade';
  proxy_set_header Host $host;
  proxy_cache_bypass $http_upgrade;

  proxy_set_header X-Forwarded-For $remote_addr;
  proxy_set_header X-Forwarded-Proto $scheme;
  proxy_set_header X-Forwarded-Host $host;
}
```

If using a subpath, be sure `.env` has `N8N_PATH` and the WEBHOOK / EDITOR / API URLs set to the subpath.

---

## 7) Check workflows in PostgreSQL

List most recent workflows:
```bash
sudo -u postgres psql -d <database_name> -c 'SELECT id, name, "createdAt" FROM workflow_entity ORDER BY "createdAt" DESC LIMIT 5;'
```

Delete a workflow by name:
```bash
sudo -u postgres psql -d <database_name> -c "DELETE FROM workflow_entity WHERE name = 'Workflow_name';"
```

---

## 8) Install Ollama

Update system and install Ollama:
```bash
sudo apt update && sudo apt upgrade -y
curl -fsSL https://ollama.com/install.sh | sh
```

Enable and start the Ollama service:
```bash
sudo systemctl enable ollama
sudo systemctl start ollama
```

Check status:
```bash
systemctl status ollama
```

Verify Ollama is listening on port 11434:
```bash
curl http://localhost:11434/api/tags
```

Pull a model:
```bash
ollama pull llama3.2
```

---

## 9) Connect to Postgres from Docker (one-off)

Example: connect to a remote postgres from a disposable postgres client container:
```bash
sudo docker run -it --rm postgres:12-alpine psql -h 172.31.32.2 -U mgvcl_n8n -d mgvcl_n8n_prod
```

---

## 10) Reset 2FA for a user in n8n (via Postgres)

If a user is locked out due to 2FA, you can remove 2FA entries from their user record.

Remove `twoFactorAuth` from settings (replace email):
```sql
UPDATE "public"."user"
SET "settings" = (settings::jsonb - 'twoFactorAuth')::json
WHERE "email" = 'user@example.com';
```

Set mfaEnabled to false (note: ensure quote characters are standard ASCII quotes):
```sql
UPDATE "public"."user"
SET "mfaEnabled" = false
WHERE "email" = 'user@example.com';
```

If you need to fully reset 2FA fields:
```sql
UPDATE "public"."user"
SET 
  "settings" = (settings::jsonb - 'twoFactorAuth')::json,
  "mfaEnabled" = false,
  "mfaSecret" = NULL,
  "mfaRecoveryCodes" = NULL
WHERE "email" = 'user@example.com';
```

---

## Notes and reminders

- Replace all `<placeholders>` (URLs, IPs, database names, usernames, passwords) with your actual values.
- When exposing services behind a reverse proxy, always use HTTPS and set `N8N_SECURE_COOKIE=true`.
- If you use a subpath, confirm that `N8N_PATH` and the various URL environment variables (`WEBHOOK_URL`, `N8N_EDITOR_BASE_URL`, `N8N_API_URL`) reflect the subpath exactly.
- Backup your database before performing DELETE operations or large changes.
