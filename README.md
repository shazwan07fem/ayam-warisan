# Ayam Warisan – Self‑service Food Ordering Kiosk

## Overview
A full‑stack, server‑less food‑ordering system built with **Cloudflare Workers** and **Cloudflare D1**. It consists of:

- **Kiosk UI** (`/`) – displays active categories, menu cards with images (fallback emoji), lets customers select quantity, choose *Makan Sini* (dine‑in) or *Bungkus* (takeaway), submit an order and see a sequential queue number and real‑time status.
- **Admin CMS** (`/admin.html`) – password‑protected UI to manage categories, items, images, shop settings and kitchen order queue.
- **Worker backend** (`src/index.js`) – implements the public API, admin API, image upload, D1 schema initialization & seeding, and all business logic.
- **Static assets** (`public/` folder) – HTML, CSS, JavaScript for both front‑ends.

The project is ready to be deployed to Cloudflare Workers with a single `wrangler publish` after you set the required environment variable `ADMIN_PASSWORD`.

---

## Project Structure
```
ayam-warisan/
├─ public/                 # Static site files
│   ├─ index.html          # Kiosk UI
│   └─ admin.html          # Admin CMS UI
├─ src/
│   └─ index.js            # Worker entry point (API + HTML routing)
├─ package.json            # npm manifest (no runtime deps)
├─ wrangler.toml           # Cloudflare Workers configuration
└─ README.md               # You are here
```

## Setup & Deployment
1. **Install Wrangler** (Cloudflare CLI) – you need it locally.   
   ```bash
   npm install -g wrangler   # or use `pnpm add -g wrangler`
   ```
2. **Configure the project** – edit `wrangler.toml` if you want a custom name or different compatibility date.
3. **Set the admin password** – add an environment variable in your Cloudflare dashboard (Workers → Settings → Environment Variables) called `ADMIN_PASSWORD`.  If it is missing the admin routes will return **503** as required.
4. **Publish** – run:
   ```bash
   wrangler publish
   ```
   The command will:
   - Create the D1 database (binding `DB` as defined in `wrangler.toml`).
   - Run `ensureDb()` on the first request, creating tables and seeding 15 sample menu items.
   - Upload the static site (`public/` folder) which will be served automatically.
5. **Visit** – after deployment you will have two URLs:
   - `https://<YOUR_WORKER>.pages.dev/` – the kiosk front‑end.
   - `https://<YOUR_WORKER>.pages.dev/admin.html` – the admin panel (login with the password you set).

## Important Implementation Details
- **Database initialization** – `ensureDb()` runs once (guarded by a module‑scoped `dbReady` flag) and creates the required tables (`settings`, `categories`, `items`, `images`, `orders`, `order_items`). It also inserts five categories and fifteen sample menu items if the tables are empty.
- **Order processing** – POST `/api/orders` calculates the total price, inserts a row in `orders` and batch‑inserts the order items. The returned `queueNumber` is simply the auto‑increment `orderId`, guaranteeing a sequential line.
- **Image handling** – client‑side canvas is converted to a PNG Blob, base‑64‑encoded, and stored in D1 as a BLOB (`ArrayBuffer`). Retrieval converts the stored blob back to a `Uint8Array` and serves it with `Content‑Type: image/png`.
- **Admin security** – every `/admin*` endpoint checks the `ADMIN_PASSWORD` environment variable and expects a `Authorization: Bearer <password>` header. If the env var is missing the server replies **503**.
- **Category deletion safety** – attempting to delete a category that still contains items returns **409**.
- **Quick price toggle** – the admin UI updates the `active` flag of items/categories instantly via the corresponding API endpoints.

---

## Next Steps / Extensions
- Add WebSocket or Server‑Sent Events for real‑time order status updates in the kiosk UI.
- Implement a printable kitchen ticket view.
- Add image‑compression on the client (canvas `toBlob` with quality control) to stay under the 95 KB limit.
- Deploy a CI/CD pipeline that pushes commits to GitHub and automatically runs `wrangler publish`.

---

> **Tip**: If you prefer a fully automated GitHub‑Actions deployment, create a workflow that runs `wrangler publish --env production` on every push to the `main` branch and stores the `ADMIN_PASSWORD` as a secret.
