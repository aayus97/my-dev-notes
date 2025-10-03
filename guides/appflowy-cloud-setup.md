---
title: "AppFlowy-Cloud Setup (localhost)"
description: "From fresh clone to a working AppFlowy-Cloud at http://localhost, with MinIO, Nginx, and large ZIP imports."
---

# 🚀 AppFlowy-Cloud Setup (localhost)

This guide explains how to run **AppFlowy-Cloud** locally at [http://localhost](http://localhost), with:

- sane `.env`
- `docker-compose.override.yml` to stop GoTrue crashes
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
Edit .env
Open .env and replace the relevant sections with these values (leave other lines as-is):
# --- Core URLs (critical) ---
APPFLOWY_BASE_URL=http://localhost
APPFLOWY_WEB_URL=http://localhost
APPFLOWY_S3_PRESIGNED_URL_ENDPOINT=http://localhost/minio-api

# --- GoTrue admin (for /console login) ---
GOTRUE_ADMIN_EMAIL=admin@localhost
GOTRUE_ADMIN_PASSWORD=ChangeMe123!

# --- JWT secret (any long random string) ---
GOTRUE_JWT_SECRET=super_secret_change_me

# --- Reverse-proxy path for GoTrue ---
API_EXTERNAL_URL=/gotrue

# --- Email: local dev (autoconfirm, no SMTP) ---
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
3) Prevent GoTrue crashes
Create a file named docker-compose.override.yml in the repo root:
services:
  gotrue:
    environment:
      GOTRUE_EXTERNAL_GOOGLE_ENABLED: "false"
      GOTRUE_EXTERNAL_GITHUB_ENABLED: "false"
      GOTRUE_EXTERNAL_DISCORD_ENABLED: "false"
      GOTRUE_EXTERNAL_APPLE_ENABLED: "false"
      GOTRUE_SAML_ENABLED: "false"
4) Nginx: enable large ZIP uploads & long timeouts
Check your nginx/nginx.conf contains these blocks:
# Allow big uploads globally
client_max_body_size 2G;

# Import endpoint
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

# MinIO presigned uploads
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
To verify inside the container:
docker compose exec nginx sh -lc 'nginx -T | sed -n "1,220p" | \
grep -nE "client_max_body_size|/minio-api/|/api/import|proxy_request_buffering|read_timeout|send_timeout"'
5) Bring everything up
docker compose up -d
docker compose ps
Expected:
postgres → healthy
gotrue → healthy
appflowy_cloud → Up
nginx, minio, redis → Up
(optional) admin_frontend, appflowy_web, appflowy_worker → Up
6) Health checks
# GoTrue
curl -i http://localhost/gotrue/health

# AppFlowy backend
curl -i http://localhost/api/server-info

# MinIO UI (browser)
open http://localhost/minio
7) Admin Console
Open in browser:
http://localhost/console
Login with:
Email: admin@localhost
Password: ChangeMe123!
8) Create a user
Go to:
http://localhost
Sign up as your regular user (not admin).
9) Quick success path (TL;DR)
cp dev.env .env

# edit .env with:
# APPFLOWY_BASE_URL=http://localhost
# APPFLOWY_WEB_URL=http://localhost
# APPFLOWY_S3_PRESIGNED_URL_ENDPOINT=http://localhost/minio-api
# GOTRUE_ADMIN_EMAIL=admin@localhost
# GOTRUE_ADMIN_PASSWORD=ChangeMe123!
# GOTRUE_JWT_SECRET=super_secret_change_me
# API_EXTERNAL_URL=/gotrue
# GOTRUE_MAILER_AUTOCONFIRM=true
# APPFLOWY_INDEXER_ENABLED=false

# create docker-compose.override.yml with all SSO flags set to "false"
cat > docker-compose.override.yml <<'YAML'
services:
  gotrue:
    environment:
      GOTRUE_EXTERNAL_GOOGLE_ENABLED: "false"
      GOTRUE_EXTERNAL_GITHUB_ENABLED: "false"
      GOTRUE_EXTERNAL_DISCORD_ENABLED: "false"
      GOTRUE_EXTERNAL_APPLE_ENABLED: "false"
      GOTRUE_SAML_ENABLED: "false"
YAML

docker compose up -d
curl -i http://localhost/gotrue/health
curl -i http://localhost/api/server-info

---

✅ This is now **fully copyable as one `.md` file**.  

Would you like me to package this into a downloadable `index.md` file (so you can just drop it into your repo), or keep it inline like above?