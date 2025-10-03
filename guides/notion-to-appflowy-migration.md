
---

## `guides/notion-to-appflowy-migration.md`

```markdown
---
title: "Notion → AppFlowy Migration"
description: "Export Notion as Markdown & CSV, import via AppFlowy Web, and verify uploads with MinIO logs."
---

# 📦 Notion → AppFlowy Migration

This is the **clean import path** that avoids 413s, timeouts, and stuck tasks.

> **Prereq:** A working local AppFlowy-Cloud at [http://localhost](http://localhost).  
> See: [AppFlowy-Cloud Setup](appflowy-cloud-setup.md)

---

## 1) Export from Notion (the right way)

In Notion, export your content with:

- **Format:** **Markdown & CSV** ✅  
- **Do not** export as HTML or PDF  
- Keep the `.zip` **as-is** (do **not** unzip)

---

## 2) Import into AppFlowy Web

1. Open **AppFlowy Web** (http://localhost) and go to your workspace  
2. **Import → Notion**  
3. Select the **Notion Markdown ZIP**

---

## 3) Watch server logs while importing

Use this to confirm the browser is doing a **presigned upload** to MinIO and the worker is unzipping:

```bash
docker compose logs -f nginx appflowy_cloud appflowy_worker | \
egrep -i '/minio-api/|PUT|import|zip|error|failed'
