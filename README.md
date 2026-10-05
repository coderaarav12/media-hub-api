# MediaHub API

Backend API for [MediaHub](https://greatmedia.goelaarav.dpdns.org/) — a streaming discovery platform.

---

## 🚨 CRITICAL NOTES FOR AI AGENTS / DEVELOPERS

**Read this before touching anything.**

### 1. TMDB Auth — Bearer Token ONLY

The `TMDB_KEY` environment variable in Vercel is a **TMDB v4 Read Access Token** (long `eyJ...` JWT string).

✅ **Correct:**
```js
headers: { Authorization: `Bearer ${TMDB_KEY}` }
```

❌ **Wrong (causes 401 on all TMDB calls):**
```js
params: { api_key: TMDB_KEY }
```

Never switch this to `api_key` query param. It will silently break all TMDB endpoints.

---

### 2. Vercel Project URL

The actual Vercel deployment is at:
```
https://media-hub-api-nine.vercel.app/
```

**NOT** `media-hub-api.vercel.app` — that is a completely different, unrelated project.

---

### 3. vercel.json Routing

The `vercel.json` rewrite must point to `/api/index` (the serverless function file), not `/api`:

✅ **Correct:**
```json
{ "source": "/(.*)", "destination": "/api/index" }
```

❌ **Wrong (causes HTML error pages instead of JSON):**
```json
{ "source": "/(.*)", "destination": "/api" }
```

---

### 4. Frontend Config — `config.js` is Gitignored

The frontend (`media-hub-frontend`) loads `config.js` at runtime to set `API_BASE`. This file is **gitignored** and is injected at deploy time by the GitHub Actions workflow via:

```yaml
echo "const API_BASE = '${{ secrets.VERCEL_API_URL }}'" > config.js
```

The GitHub secret `VERCEL_API_URL` must be set to:
```
https://media-hub-api-nine.vercel.app/api
```

If the secret is wrong or stale, the entire frontend will fail silently (all fetch calls will go to the wrong API).

---

### 5. Route Order Matters — Don't Shuffle

Express routes must be declared in this order to avoid conflicts:

```
/api/movie/popular   ← BEFORE /api/movie/:id
/api/tv/popular      ← BEFORE /api/tv/:id
/api/anime/trending  ← BEFORE /api/anime/:id
/api/anime/popular   ← BEFORE /api/anime/:id
/api/anime/search    ← BEFORE /api/anime/:id
```

If you put `/api/movie/:id` before `/api/movie/popular`, Express will match `popular` as an `:id` param and the popular route will never be reached.

---

### 6. Two Separate Repos

| Repo | Purpose | Deployed At |
|------|---------|-------------|
| `media-hub-frontend` | Frontend SPA + local Express proxy (`server.js`) | Cloudflare Pages |
| `media-hub-api` | Vercel serverless API (`api/index.js`) | Vercel |

**Changes to `server.js` in the frontend repo must be manually mirrored to `api/index.js` in the API repo.** They are NOT synced automatically.

Key differences between the two files:
- `server.js` uses `import/export` (ESM) + `app.listen()` + serves static files
- `api/index.js` uses `require()` (CJS) + `module.exports = app` (no `listen`)

---

### 7. `module.exports` Not `app.listen`

The Vercel serverless function must export the app, not start a server:

✅ `module.exports = app;`  
❌ `app.listen(PORT, ...)`

---

### 8. Key Routes Reference

| Route | Description |
|-------|-------------|
| `GET /api/trending?media=movie\|tv\|all` | Trending this week |
| `GET /api/movie/popular` | Popular movies |
| `GET /api/tv/popular` | Popular TV shows |
| `GET /api/movie/:id` | Movie detail (includes `credits,videos,similar`) |
| `GET /api/tv/:id` | TV detail (includes `credits,aggregate_credits,videos,similar`) |
| `GET /api/tv/:id/season/:season` | Season episodes |
| `GET /api/person/:id` | Person detail (includes `combined_credits`, biography fallback) |
| `GET /api/imdb/:id` | OMDb ratings by IMDb ID (uses axios, not fetch) |
| `GET /api/anime/trending` | Trending anime (Anilist) |
| `GET /api/anime/popular` | Popular anime (Anilist) |
| `GET /api/anime/search?q=` | Search anime (Anilist) |
| `GET /api/anime/:id` | Anime detail with characters & voice actors |
| `GET /api/sources?type=movie\|tv\|anime&id=` | Streaming source URLs |

---

### 9. OMDb / IMDb Endpoint

Uses `axios`, NOT the native `fetch` (for Node.js compatibility on Vercel):

```js
const { data } = await axios.get(`https://www.omdbapi.com/?i=${req.params.id}&apikey=thewdb`)
```

---

### 10. TV Shows — Use `aggregate_credits`

Standard `credits.cast` is often empty for TV shows. Always append `aggregate_credits` and fall back on the frontend:

```js
// Backend
append_to_response: "credits,aggregate_credits,videos,similar"

// Frontend
const cast = d.credits?.cast?.length ? d.credits.cast : d.aggregate_credits?.cast || []
```

---

## Stack

- **Runtime:** Node.js (CommonJS)
- **Framework:** Express
- **Data Sources:** TMDB (movies/TV), Anilist (anime), OMDb (ratings)
- **Hosting:** Vercel (serverless)
