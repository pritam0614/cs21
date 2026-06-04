# Crowd FAQ — Deployment Guide

## Overview

Two services deploy to Render:
- **Frontend** — React SPA, static build served by Render
- **Backend** — Express API, Node.js runtime

Database: MongoDB Atlas (cloud)
LLM: OpenAI API (production) or Ollama (local dev only)

---

## Prerequisites

- GitHub account with repo `Nancypaul08/crowd-faq`
- [Render account](https://render.com) (free tier)
- [MongoDB Atlas account](https://cloud.mongodb.com) (free M0 tier)
- [OpenAI API key](https://platform.openai.com/api-keys) (for production)

---

## Step 1 — Push to GitHub

Make sure everything is committed and pushed:

```bash
cd crowd/mvp
git add -A
git commit -m "feat: ready for deployment"
git push origin cs21-restore
```

> `cs21-restore` is the current branch. Replace with `main` if that's your deployment branch.

---

## Step 2 — Create MongoDB Atlas Cluster

1. Go to [cloud.mongodb.com](https://cloud.mongodb.com) → **Create Cluster**
2. Choose **M0 Free** tier, **AWS**, **Singapore** region
3. Wait for cluster to provision (~2 min)
4. **Security → Database Access** → Create user (e.g. `crowd_user`) with read/write
5. **Security → Network Access** → Add IP: `0.0.0.0/0` (allow all)
6. **Deployment → Database** → Click your cluster → **Connect** → **Connect your application**
7. Copy connection string, replace `<password>` with your DB user password:

```
mongodb+srv://crowd_user:<PASSWORD>@cluster0.xxxxx.mongodb.net/crowd_faq?retryWrites=true&w=majority
```

---

## Step 3 — Generate JWT Secret

```bash
openssl rand -hex 32
```

Copy the output — you'll paste it in the Render dashboard.

---

## Step 4 — Apply Render Blueprint

1. Log in to [Render Dashboard](https://dashboard.render.com)
2. Click **New → Blueprint**
3. Connect your GitHub account and select `crowd-faq` repo
4. Render auto-detects `render.yaml` — confirm both services:
   - `crowd-faq-backend` (Node.js)
   - `crowd-faq-frontend` (Static)
5. Click **Apply Blueprint**

---

## Step 5 — Set Environment Variables

In the Render Blueprint UI, set these for each service:

### Backend (`crowd-faq-backend`)

| Key | Value | Notes |
|---|---|---|
| `NODE_ENV` | `production` | |
| `PORT` | `10000` | Render sets this, but set anyway |
| `MONGO_URI` | _(Atlas connection string)_ | From Step 2 |
| `JWT_SECRET` | _(output from Step 3)_ | |
| `JWT_EXPIRES_IN` | `7d` | |
| `CLIENT_URL` | _(frontend URL after deploy)_ | Fill after Step 6 |
| `LLM_PROVIDER` | `openai` | |
| `OPENAI_API_KEY` | _(from platform.openai.com)_ | |
| `LLM_MODEL` | `gpt-4o-mini` | |
| `EMBED_MODEL` | `text-embedding-3-small` | OpenAI embedding model |

### Frontend (`crowd-faq-frontend`)

| Key | Value |
|---|---|
| `VITE_API_URL` | `https://crowd-faq-backend.onrender.com` |
| `VITE_GITHUB_USERNAME` | `Nancypaul08` |

---

## Step 6 — Note the Frontend URL

After the first deploy, Render shows the frontend URL (e.g. `https://crowd-faq-frontend.onrender.com`).

1. Copy it
2. Go to **crowd-faq-backend → Environment**
3. Set `CLIENT_URL` to the frontend URL
4. Save → backend auto-redeploys

---

## Step 7 — Verify

```bash
# Backend health
curl https://crowd-faq-backend.onrender.com/api/health

# Should return:
# { "status": "ok", "mongodb": "connected", "ollama": "not_configured" }
```

Open `https://crowd-faq-frontend.onrender.com` in your browser.

- Register a new account
- Go to `/chat`, ask a question
- Check `/faqs` — new card should appear with ✨ badge

---

## Deployment Summary

| Item | Value |
|---|---|
| Frontend URL | `https://crowd-faq-frontend.onrender.com` |
| Backend URL | `https://crowd-faq-backend.onrender.com` |
| Health check | `https://crowd-faq-backend.onrender.com/api/health` |
| MongoDB | MongoDB Atlas M0 (AWS Singapore) |
| LLM | OpenAI `gpt-4o-mini` + `text-embedding-3-small` |

---

## Updating After Code Changes

```bash
git add -A
git commit -m "fix: ..."
git push origin cs21-restore
```

Render auto-deploys on push to the connected branch. No manual trigger needed.

---

## Switching LLM Provider

To use **Anthropic Claude** instead of OpenAI:

```bash
# Backend env vars on Render
LLM_PROVIDER=anthropic
ANTHROPIC_API_KEY=sk-ant-...
LLM_MODEL=claude-3-haiku-20240307
```

The `ollama.js` service handles the routing — no code changes needed.

---

## Removing the Deployment

1. Render Dashboard → select each service → **Delete**
2. Or delete the entire Blueprint from the Blueprint page

This does **not** affect your MongoDB Atlas cluster.

---

## Common Issues

| Symptom | Fix |
|---|---|
| CORS error in browser console | Set `CLIENT_URL` in backend env vars to exact frontend URL (include `https://`) |
| "Invalid token" on login | Generate new `JWT_SECRET` (`openssl rand -hex 32`) and update in Render |
| Backend returns 503 on `/api/chat` | Check `OPENAI_API_KEY` is set correctly in Render dashboard |
| Frontend shows 500 errors | Backend is likely cold-starting (free tier spins down after 15 min). Wait 30s and retry. |
| MongoDB connection error | Verify `MONGO_URI` in Render matches Atlas connection string exactly |
| Frontend still shows old code | Hard refresh (`Cmd+Shift+R`) or invalidate Render cache |

---

## Free Tier Limits

| Limit | Value |
|---|---|
| Service idle time before spin-down | 15 minutes |
| Cold start time | ~30 seconds |
| Build minutes | 500/month (both services combined) |
| Disk | 1GB per service |
| RAM | 512MB (backend), 256MB (static) |

For a personal project or demo, the free tier is sufficient. Upgrade to a paid tier ($7+/month) for always-on backend with no cold starts.