# Deploy Crowd FAQ to Render — Step by Step

> **Estimated time:** 30–45 minutes (mostly waiting for Render to build)

---

## Prerequisites

- [Render account](https://render.com) (free tier is fine)
- [MongoDB Atlas cluster](https://cloud.mongodb.com) (free M0 tier)
- [OpenAI API key](https://platform.openai.com/api-keys) (~$1–5/month for moderate FAQ usage)

---

## Step 1 — Update and Push to GitHub

Make sure all deployment files are committed and pushed to your GitHub repo.

```bash
git add -A
git commit -m "feat: add Render deployment files"
git push origin cs21-restore
```

---

## Step 2 — Set Up MongoDB Atlas

1. Go to [cloud.mongodb.com](https://cloud.mongodb.com) → Create free cluster (M0)
2. Choose **AWS** / Singapore region (closest to Render's Singapore zone)
3. Under **Security → Network Access** → Add IP: `0.0.0.0/0` (allow all IPs)
4. Under **Security → Database Access** → Create a user with read/write access
5. Click **Connect → Connect your application** → Copy the connection string
6. Replace `<password>` with your database user's password

Your connection string will look like:
```
mongodb+srv://myuser:mypassword@cluster0.xxxxx.mongodb.net/crowd_faq?retryWrites=true&w=majority
```

---

## Step 3 — Create a Render Blueprint

1. Log in to [Render Dashboard](https://dashboard.render.com)
2. Click **New → Blueprint**
3. Connect your GitHub account and select the `crowd-faq` repo
4. Render will detect `render.yaml` automatically — confirm the two services:
   - `crowd-faq-backend` (Node.js API)
   - `crowd-faq-frontend` (Static site)
5. Click **Apply Blueprint**

---

## Step 4 — Set Environment Variables

In the Render Blueprint UI, set the following for each service:

### Backend (`crowd-faq-backend`)

| Key | Value | Notes |
|---|---|---|
| `NODE_ENV` | `production` | |
| `PORT` | `10000` | Render sets this automatically, but set it anyway |
| `MONGO_URI` | _(your Atlas connection string)_ | From Step 2 |
| `JWT_SECRET` | `openssl rand -hex 32` | Generate locally and paste |
| `JWT_EXPIRES_IN` | `7d` | |
| `CLIENT_URL` | `https://your-frontend.onrender.com` | Set after frontend deploys |
| `LLM_PROVIDER` | `openai` | |
| `OPENAI_API_KEY` | _(your OpenAI key)_ | From platform.openai.com |
| `LLM_MODEL` | `gpt-4o-mini` | Fast + cheap for FAQ generation |

### Frontend (`crowd-faq-frontend`)

| Key | Value | Notes |
|---|---|---|
| `VITE_API_URL` | `https://your-backend.onrender.com` | Backend URL after deploy |
| `VITE_GITHUB_USERNAME` | `Nancypaul08` | |

---

## Step 5 — Update CLIENT_URL After Frontend Deploys

Once the frontend deploys and you get its URL (e.g. `https://crowd-faq-frontend.onrender.com`):

1. Go to Render Dashboard → **crowd-faq-backend → Environment**
2. Update `CLIENT_URL` to your frontend URL
3. Click **Save Changes** → backend will auto-redeploy

---

## Step 6 — Verify Deployment

After both services are live:

```
Backend health:  https://crowd-faq-backend.onrender.com/api/health
Frontend:        https://crowd-faq-frontend.onrender.com
```

Try asking a question in the AI ChatBot — it should generate an answer via OpenAI.

---

## Architecture on Render

```
User browser
    │
    ├── HTTPS ──► Frontend (Static, Render)
    │                   │
    │                   │ VITE_API_URL
    │                   ▼
    │            Backend API (Node.js, Render, Port 10000)
    │                   │
    │                   │ OpenAI API key
    │                   ▼
    │            OpenAI API (gpt-4o-mini)
    │
    └── HTTPS ──► MongoDB Atlas (cloud.mongodb.com)
```

---

## Troubleshooting

### "CORS error" in browser console
Make sure `CLIENT_URL` in the backend env vars matches your frontend's exact URL (including `https://`). After updating it, the backend needs ~1 min to pick up the change.

### "Invalid token" on login
Generate a new `JWT_SECRET` — it's likely mismatched between deploys. Run `openssl rand -hex 32` locally and paste it into the Render dashboard.

### "OpenAI API error 429"
You've hit your OpenAI rate limit. Check usage at [platform.openai.com](https://platform.openai.com). The `gpt-4o-mini` model is very cheap (~$0.15/1M tokens) but still has per-minute limits on free accounts.

### Backend builds but crashes immediately
Check Render logs under **crowd-faq-backend → Logs**. Common issues:
- Wrong `MONGO_URI` format (must include `mongodb+srv://`)
- `OPENAI_API_KEY` missing or malformed
- MongoDB Atlas IP whitelist not set to `0.0.0.0/0`

### Frontend shows 500 on API calls
Check the backend is running on the free tier (spins down after 15 min of inactivity, cold starts take ~30s). Use a paid tier if cold start latency is unacceptable.

---

## Local Development After Deploying to Render

Copy `server/.env.production.example` to `server/.env` and set:
```bash
VITE_API_URL=http://localhost:5001
```
Then run `npm run dev` in both `client/` and `server/` as usual. Your local instance uses Ollama, production uses OpenAI — same code path.

---

## Files for Deployment

```
render.yaml                   ← Render Blueprint (both services)
server/.env.production.example ← All env vars template
server/index.js               ← Updated CORS for multiple origins
server/services/ollama.js     ← Unified Ollama/OpenAI service
DEPLOY-RENDER.md              ← This file
```