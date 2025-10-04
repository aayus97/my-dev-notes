---
title: "AppFlowy-Cloud Setup (localhost)"
description: "From fresh clone to a working AppFlowy-Cloud at http://localhost, with MinIO, Nginx, and large ZIP imports."
---

# 🚀 AppFlowy-Cloud Setup (localhost)

This is a clean, repeatable path to a working **AppFlowy-Cloud** at [http://localhost](http://localhost), including:
- sane `.env`
- `docker-compose.override.yml` to prevent GoTrue boolean parsing crashes
- MinIO behind Nginx with **presigned uploads**
- large file upload settings (avoid 413/timeout for Notion ZIPs)
- verifications & health checks

> **Tested on:** macOS + Docker Desktop (Apple Silicon)  
> **Compose:** Docker Compose V2 (`docker compose ...`)

---

## 1) Prerequisites

- Docker Desktop running  
- Free ports: `80, 443, 5432, 6379, 9000, 9999`  
- Terminal (`zsh` / `bash`)

---

## 2) Clone and prepare files

```bash
git clone https://github.com/AppFlowy-IO/AppFlowy-Cloud.git
cd AppFlowy-Cloud
cp dev.env .env
```

Edit `.env` with **only these changes (leave the rest):**

```bash
# --- Core URLs (critical) ---
APPFLOWY_BASE_URL=http://localhost
APPFLOWY_WEB_URL=http://localhost
APPFLOWY_S3_PRESIGNED_URL_ENDPOINT=http://localhost/minio-api

# --- GoTrue admin (for /console login) ---
GOTRUE_ADMIN_EMAIL=admin@localhost
GOTRUE_ADMIN_PASSWORD=ChangeMe123!

# JWT secret (any long random string)
GOTRUE_JWT_SECRET=super_secret_change_me

# GoTrue sits behind Nginx at /gotrue
API_EXTERNAL_URL=/gotrue

# --- Email: local dev (no confirmations, no SMTP) ---
GOTRUE_MAILER_AUTOCONFIRM=true
APPFLOWY_MAILER_SMTP_HOST=
APPFLOWY_MAILER_SMTP_USERNAME=
APPFLOWY_MAILER_SMTP_EMAIL=
APPFLOWY_MAILER_SMTP_PASSWORD=
APPFLOWY_MAILER_SMTP_TLS_KIND=

# --- MinIO (dev defaults) ---
AWS_ACCESS_KEY=minioadmin
AWS_SECRET=minioadmin

# --- Optional: silence AI indexer warnings if no OpenAI key ---
APPFLOWY_INDEXER_ENABLED=false
```

**Why these matter:**

- `APPFLOWY_BASE_URL` / `APPFLOWY_WEB_URL` → absolute links & CORS  
- `APPFLOWY_S3_PRESIGNED_URL_ENDPOINT` must match Nginx: `http://localhost/minio-api`  
- `API_EXTERNAL_URL=/gotrue` so GoTrue behaves under `/gotrue`  
- `GOTRUE_MAILER_AUTOCONFIRM=true` = sign-in without SMTP for local dev  

---

## 3) Prevent GoTrue from crashing on empty booleans

Create `docker-compose.override.yml` in the repo root:

```yaml
services:
  gotrue:
    environment:
      GOTRUE_EXTERNAL_GOOGLE_ENABLED: "false"
      GOTRUE_EXTERNAL_GITHUB_ENABLED: "false"
      GOTRUE_EXTERNAL_DISCORD_ENABLED: "false"
      GOTRUE_EXTERNAL_APPLE_ENABLED: "false"
      GOTRUE_SAML_ENABLED: "false"
```

**Fixes:**
```
fatal: strconv.ParseBool: parsing "": invalid syntax
```

---

## 4) Nginx: large ZIP uploads & long timeouts

Ensure your Nginx config (`nginx/nginx.conf`) has:

```nginx
# big uploads globally
client_max_body_size 2G;

# import endpoint (long timeouts + no buffering)
location /api/import {
  proxy_pass $appflowy_cloud_backend;

  proxy_read_timeout 600s;
  proxy_connect_timeout 600s;
  proxy_send_timeout 600s;

  proxy_request_buffering off;
  proxy_buffering off;
  proxy_cache off;
  client_max_body_size 2G;
}

# presigned uploads to MinIO via Nginx
location /minio-api/ {
  proxy_pass $minio_api_backend;
  proxy_set_header Host $minio_internal_host;
  rewrite ^/minio-api/(.*) /$1 break;

  proxy_http_version 1.1;
  proxy_set_header Connection "";
  client_max_body_size 2G;
  proxy_request_buffering off;
  proxy_read_timeout 600s;
  proxy_send_timeout 600s;
}
```

**Verify inside the container after it starts:**

```bash
docker compose exec nginx sh -lc 'nginx -T | sed -n "1,220p" | \
grep -nE "client_max_body_size|/minio-api/|/api/import|proxy_request_buffering|read_timeout|send_timeout"'
```

---

## 5) Bring everything up

```bash
docker compose up -d
docker compose ps
```

**You want to see:**

```
postgres: healthy
gotrue: healthy
appflowy_cloud: Up
nginx, minio, redis: Up
(optional) admin_frontend, appflowy_web, appflowy_worker: Up
```

---

## 6) Health checks

```bash
# GoTrue
curl -i http://localhost/gotrue/health

# AppFlowy backend ready = 200
curl -i http://localhost/api/server-info

# MinIO UI reverse-proxied
open http://localhost/minio
```

---

## 7) Admin Console

Open [http://localhost/console](http://localhost/console)

Login (from `.env`):

```
Email: admin@localhost
Password: ChangeMe123!
```

---

## 8) Create a user & log in to AppFlowy Web

Visit [http://localhost](http://localhost) and sign up as your regular user (not the admin).

---

## 9) Quick success path (TL;DR)

```bash
cp dev.env .env
# Set:
#  APPFLOWY_BASE_URL=http://localhost
#  APPFLOWY_WEB_URL=http://localhost
#  APPFLOWY_S3_PRESIGNED_URL_ENDPOINT=http://localhost/minio-api
#  GOTRUE_ADMIN_EMAIL=admin@localhost
#  GOTRUE_ADMIN_PASSWORD=ChangeMe123!
#  GOTRUE_JWT_SECRET=super_secret_change_me
#  API_EXTERNAL_URL=/gotrue
#  GOTRUE_MAILER_AUTOCONFIRM=true
#  APPFLOWY_INDEXER_ENABLED=false

# Create docker-compose.override.yml with all SSO booleans "false"
docker compose up -d
curl -i http://localhost/gotrue/health
curl -i http://localhost/api/server-info
```
