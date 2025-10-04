---
title: "Home"
---

# 🗂️ Migrations & Dev Notes

Welcome! This site tracks my hands-on notes for **Notion → AppFlowy** migrations and a reliable **AppFlowy-Cloud** local setup.

## 📚 Contents

- **Migration Guide**
  - [Notion → AppFlowy Migration](guides/notion-to-appflowy-migration.md)

- **Prereqs & Infra**
  - [AppFlowy-Cloud Setup (localhost)](guides/appflowy-cloud-setup.md)

---

### Quick Start (TL;DR)

1) Set up AppFlowy-Cloud (see guide).  
2) Export Notion as **Markdown & CSV** (keep ZIP intact).  
3) AppFlowy Web → Workspace → **Import → Notion** → select ZIP.  
4) Watch logs for `PUT /minio-api` and worker `finish unzip`.

> If anything looks off, jump to **[Common Errors & Fixes](guides/troubleshooting.md)**.
