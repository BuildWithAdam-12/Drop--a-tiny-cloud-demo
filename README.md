# Drop

A tiny cloud-storage demo. Drag an image onto the page (or paste one with ⌘V / Ctrl+V) and it is uploaded to a serverless API, stored in an object bucket, indexed in a SQL table, and served back from the edge cache — instantly, for anyone who opens the site.

It is deliberately small. The point is that every feature is a real cloud building block rather than a simulation.

## What it demonstrates

| Cloud primitive | Where it lives |
| --- | --- |
| Serverless compute | `src/worker.js` — a Worker answers `/api/*` on demand, with nothing running between requests |
| Object storage | R2 bucket bound as `env.FILES` — the image bytes themselves |
| Managed database | D1 database bound as `env.DB` — filename, type, size, timestamp |
| Static hosting / edge | `public/` served as assets; images ship with a one-year immutable cache header |

## How it works

1. The browser `POST`s the image as multipart form data to `/api/drops`.
2. The Worker streams the bytes into the R2 bucket.
3. It inserts a row into the D1 table: filename, content type, size, timestamp, and a random delete token.
4. `GET /api/drops` returns the gallery list; `GET /api/file/:id` streams an image back out of the bucket.

The table is created automatically on the first request, so a fresh database needs no migration step.

## Requirements

- Node.js 20 or newer (developed on 24)
- npm

No cloud account, API keys, or credentials are needed to run it locally.

## Install and run it
https://pv-6905c5b9674b-5bf3512a-ff9b-4273-a80d-59fd209d9201.style.dev/

To use a different port — useful if 8787 is taken:

```sh
npm run dev -- --port 3000
```

D1 and R2 are **simulated locally** in `.wrangler/state`, so uploads work immediately and survive restarts on your machine. Nothing is sent to the cloud during local development.

## Project structure

```
src/worker.js         Worker: the API routes, then the asset fallback
public/index.html     Page shell
public/styles.css     Styling
public/app.js         Gallery, drag-and-drop upload, lightbox
wrangler.jsonc        Worker config: bindings, assets, compatibility date
sites.manifest.json   Deployment manifest (bundled Worker + assets)
```

There are no runtime dependencies — the Worker and frontend are plain JavaScript, and `wrangler` is the only dev dependency.

## API

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/api/drops` | List the gallery (newest first) with a count and total size |
| `POST` | `/api/drops` | Upload an image; multipart form with a `file` field |
| `GET` | `/api/file/:id` | Stream one image back (edge-cacheable) |
| `DELETE` | `/api/drops/:id` | Delete an image, requires the `x-delete-token` header |

Rejections to expect: non-images get `415`, files over 10 MB get `413`, a wrong or missing delete token gets `404`, and a state-changing request from a foreign site gets `403`. Limits live at the top of `src/worker.js`.

## Deleting your own uploads

The app has no accounts. When you upload, the server returns a random `deleteToken`, the browser keeps it in `localStorage`, and the delete button is only shown for images you uploaded. Clearing your browser storage loses that ability — the image stays in the gallery.

## Building and deploying

```sh
npm run build
```

This bundles the Worker to `dist/index.js`, which `sites.manifest.json` points at alongside `public/`. Upload that manifest to deploy.

`wrangler.jsonc` uses a placeholder `database_id` because local development never contacts the cloud. A hosted deployment matches the `DB` and `FILES` bindings to its own provisioned resources by name, so the placeholder does not need to be filled in for that path.

## Notes

- The gallery is **public and shared** — every visitor sees every image. There is no login.
- `GET /api/drops` returns the most recent 120 uploads.
- `/api/file/:id` is intentionally long-lived and cacheable; the images are immutable once written.
