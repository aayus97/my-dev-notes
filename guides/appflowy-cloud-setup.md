# AppFlowy Cloud Setup Guide

This guide gets you from a **fresh clone** to a **working AppFlowy-Cloud** at [http://localhost](http://localhost), including:

- Sane `.env` setup  
- A small `docker-compose.override.yml` that prevents GoTrue crashes  
- MinIO behind Nginx with presigned uploads  
- Large file upload settings (so big Notion ZIPs don’t 413/timeout)  
- Verifications & health checks  
- A clean Notion import path  
- A big “errors & fixes” section (the stuff that actually goes wrong)  

Tested on **macOS** with **Docker Desktop** & **Apple Silicon**.  
Commands use **docker compose (Compose V2)**.

---

## 1) Prerequisites

- Docker Desktop installed & running  
- Free local ports: `80`, `443`, `5432`, `6379`, `9000`, `9999`  
- A terminal (`zsh` or `bash`)

---

## 2) Clone and Prepare Files

```bash
git clone https://github.com/AppFlowy-IO/AppFlowy-Cloud.git
cd AppFlowy-Cloud
cp dev.env .env
```

### Edit `.env`

Open `.env` and make **only these edits** (leave the rest as-is):

```bash
# --- Core URLs (these three are critical) ---
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

# --- Email: make local life easy (disable confirmations) ---
GOTRUE_MAILER_AUTOCONFIRM=true
APPFLOWY_MAILER_SMTP_HOST=
APPFLOWY_MAILER_SMTP_USERNAME=
APPFLOWY_MAILER_SMTP_EMAIL=
APPFLOWY_MAILER_SMTP_PASSWORD=
APPFLOWY_MAILER_SMTP_TLS_KIND=

# --- MinIO (dev defaults) ---
AWS_ACCESS_KEY=minioadmin
AWS_SECRET=minioadmin

# --- Optional: silence AI indexer warnings if you don't have an OpenAI key ---
APPFLOWY_INDEXER_ENABLED=false
```

### Why these matter

- `APPFLOWY_BASE_URL` & `APPFLOWY_WEB_URL` are used by the backend for links and CORS.  
- `APPFLOWY_S3_PRESIGNED_URL_ENDPOINT` must match how Nginx exposes MinIO: `http://localhost/minio-api`.  
- `API_EXTERNAL_URL=/gotrue` makes GoTrue behave behind Nginx.  
- `GOTRUE_MAILER_AUTOCONFIRM=true` lets you sign in without SMTP during local dev.

---

## 3) Prevent GoTrue from Crashing on Empty Booleans

Create `docker-compose.override.yml`:

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

✅ **Fixes:**  
`strconv.ParseBool: parsing "": invalid syntax` — GoTrue hates empty bools.

---

## 4) Nginx: Allow Large ZIP Uploads & Long Timeouts

Confirm these settings inside `nginx/nginx.conf`:

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

### Verify inside the container:
```bash
docker compose exec nginx sh -lc 'nginx -T | sed -n "1,220p" | grep -nE "client_max_body_size|/minio-api/|/api/import|proxy_request_buffering|read_timeout|send_timeout"'
```

---

## 5) Bring Everything Up

```bash
docker compose up -d
docker compose ps
```

You should see:

- postgres → healthy  
- gotrue → healthy  
- appflowy_cloud → Up  
- nginx, minio, redis → Up  
- (optional) admin_frontend, appflowy_web, appflowy_worker → Up  

### Health checks
```bash
# GoTrue
curl -i http://localhost/gotrue/health

# AppFlowy backend ready = 200
curl -i http://localhost/api/server-info

# MinIO UI
open http://localhost/minio
```

---

## 6) Sign in to Admin Console

Open: [http://localhost/console](http://localhost/console)

Login:
```
Email: admin@localhost
Password: ChangeMe123!
```

If you get “Unexpected error,” see *Troubleshooting → Admin console* below.

---

## 7) Create a User & Log In to AppFlowy Web

Visit [http://localhost](http://localhost) and sign up/in as a **regular user** (not admin@localhost).

---

## 8) Import a Notion Export (the Correct Way)

In Notion, export as:

- **Format:** Markdown & CSV ✅  
- **Do not unzip** the ZIP.

In AppFlowy Web:

1. Open your workspace  
2. Click **Import → Notion**  
3. Select the Notion Markdown ZIP

Watch logs:

```bash
docker compose logs -f nginx appflowy_cloud appflowy_worker | \
egrep -i '/minio-api/|PUT|import|zip|error|failed'
```

Expected output includes:

- `PUT /minio-api/...` in nginx logs  
- Worker lines with “finish unzip …” and **no errors**  

Check MinIO:  
[http://localhost/minio](http://localhost/minio) → bucket `appflowy` → object with size > 0.

---

# Troubleshooting (Real Errors & Fixes)

---

### 1) GoTrue keeps dying with fatal env error
**Symptom:**
```
strconv.ParseBool: parsing "": invalid syntax
```
**Fix:** Use the override file from step 3 and restart:
```bash
docker compose up -d gotrue
docker compose logs --tail=120 gotrue
```

---

### 2) Backend keeps restarting: APPFLOWY_WEB_URL has not been set
**Fix:**
```bash
APPFLOWY_WEB_URL=http://localhost
APPFLOWY_BASE_URL=http://localhost
docker compose up -d appflowy_cloud
```

---

### 3) Can’t login to /console (Admin Frontend)
Check:
```bash
curl -i http://localhost/gotrue/health
curl -i http://localhost/api/server-info
```

Ensure `.env` has:
```bash
API_EXTERNAL_URL=/gotrue
```

Use `http://localhost` everywhere (not 127.0.0.1).

---

### 4) MinIO login fails
**Default credentials:**
```
Username: minioadmin
Password: minioadmin
```
If changed, sync with `AWS_ACCESS_KEY` / `AWS_SECRET` in `.env`.

---

### 5) “Network Error” when uploading large ZIPs

Ensure:

- `client_max_body_size 2G`
- `proxy_request_buffering off`
- Long `read/send` timeouts.

Check logs live:
```bash
docker compose logs -f nginx | egrep -i '/minio-api/|413|timeout'
```

---

### 6) “3 import tasks are pending. Please wait…”

Inspect and clear:
```bash
docker compose exec -T postgres psql -U appflowy -d appflowy -c \
"SELECT task_id, status, file_size, left(file_url,120) AS file_url, created_at
 FROM public.af_import_task
 ORDER BY created_at DESC LIMIT 30;"

docker compose exec -T postgres psql -U appflowy -d appflowy -c \
"DELETE FROM public.af_import_task WHERE status=0;"
```

Restart:
```bash
docker compose restart appflowy_cloud appflowy_worker nginx
```

---

### 7) Worker error: “OpenAI API key is not set”
Harmless.  
Silence it:
```bash
APPFLOWY_INDEXER_ENABLED=false
```

---

### 8) SMTP errors

If you see `535 BadCredentials`, clear SMTP settings as in step 2 or configure a real SMTP.

---

### 9) Nginx “nginx.conf not found”
Check:
```bash
docker compose exec nginx nginx -T
docker compose exec nginx nginx -s reload
```

---

### 10) Sanity Checklist

```bash
docker compose ps
curl -i http://localhost/gotrue/health
curl -i http://localhost/api/server-info
```

Verify critical envs:
```bash
docker compose exec appflowy_cloud env | \
grep -E 'APPFLOWY_BASE_URL|APPFLOWY_WEB_URL|APPFLOWY_S3_PRESIGNED_URL_ENDPOINT'
```

Expect:
```
APPFLOWY_BASE_URL=http://localhost
APPFLOWY_WEB_URL=http://localhost
APPFLOWY_S3_PRESIGNED_URL_ENDPOINT=http://localhost/minio-api
```

Verify nginx directives:
```bash
docker compose exec nginx sh -lc 'nginx -T | sed -n "1,220p" | \
grep -nE "client_max_body_size|/minio-api/|/api/import|proxy_request_buffering|read_timeout|send_timeout"'
```

---

### 11) Reset Everything (optional)
```bash
docker compose down -v
docker compose up -d
```
(`-v` deletes postgres & minio data)

---

## 12) TL;DR Quick Success Path

```bash
cp dev.env .env
```

Set:

```bash
APPFLOWY_BASE_URL=http://localhost
APPFLOWY_WEB_URL=http://localhost
APPFLOWY_S3_PRESIGNED_URL_ENDPOINT=http://localhost/minio-api
GOTRUE_ADMIN_EMAIL=admin@localhost
GOTRUE_ADMIN_PASSWORD=ChangeMe123!
GOTRUE_JWT_SECRET=super_secret_change_me
API_EXTERNAL_URL=/gotrue
GOTRUE_MAILER_AUTOCONFIRM=true
APPFLOWY_INDEXER_ENABLED=false
```

Create override file (all SSO booleans false).  
Then:

```bash
docker compose up -d
```

Check:
```bash
curl -i http://localhost/gotrue/health
curl -i http://localhost/api/server-info
```

Then open:
- [http://localhost/console](http://localhost/console) → login as admin  
- [http://localhost](http://localhost) → login as user  

Export Notion as Markdown & CSV → Import → verify PUT `/minio-api` in logs.

If anything fails:
```bash
docker compose logs --tail=200 nginx appflowy_cloud appflowy_worker gotrue
```

…and paste the last ~20 lines to debug.

---

✅ **You now have a working AppFlowy Cloud with Notion import and admin console running locally.**
