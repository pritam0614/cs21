# Crowd FAQ — AI-Powered FAQ Knowledge Portal

A real-time, AI-driven FAQ resolution system that automatically answers user questions, deduplicates semantically similar queries, and keeps a living FAQ knowledge base in sync — powered by local LLM inference.

---

## 🧠 What It Does

```
User asks a question (text or voice)
         ↓
   ┌─────────────────────────────────────┐
   │  1. Voice → Web Speech API transcribes  │
   │  2. Embed question (nomic-embed-text)   │
   │  3. Cosine similarity against all FAQs  │
   └─────────────────────────────────────┘
              ↓                              ↓
      Similarity ≥ 0.82              No close match
              ↓                              ↓
    Return existing answer       →  LLM generates formal answer
    (FAQ reused, no dup)              (qwen2.5:latest on Metal GPU)
                                           ↓
                               FAQ auto-categorized (8 categories)
                                           ↓
                               Saved to MongoDB with embedding
                                           ↓
                               Socket.io → browser updates live
                                           ↓
                               Purple "New" badge + toast notification
```

---

## ✨ Features

| Feature | Description |
|---|---|
| **Local LLM** | `qwen2.5:latest` (7.6B Q4_K_M) via Ollama — no API costs, fully offline |
| **Semantic Search** | Cosine similarity against all FAQ embeddings — catches paraphrases |
| **Deduplication** | Threshold 0.82 — no duplicate FAQs for semantically identical questions |
| **Auto-categorization** | LLM classifies into 8 categories: AI/ML, Programming, Finance, Education, Healthcare, Cloud/DevOps, Design, General |
| **Voice Input** | Browser Web Speech API (Chrome/Edge) — real-time transcription, no server round-trip |
| **Live Grid Updates** | New AI FAQs appear in the FAQ browser instantly via Socket.io |
| **Toast Notifications** | Purple AI-themed toast every time a new FAQ is auto-created |
| **AI Badges** | 🤖 on existing AI FAQs, ✨ "New" sparkle badge for 30s after creation |
| **Formal FAQ Answers** | LLM system prompt enforces professional, concise FAQ style |
| **Full History** | Chat threads stored in MongoDB with user attribution |

---

## 🏗️ Architecture

```
client/                     server/
  src/
    pages/
      ChatBot.jsx      ←  AI FAQ assistant (text + voice)
      FAQBrowser.jsx   ←  Live-updating FAQ grid
    components/
      FAQCard.jsx      ←  Card with AI/New badges
      Layout.jsx
    context/
      ToastContext.jsx ←  Global toast notifications
      SocketContext.jsx←  Socket.io event bus
    App.jsx

server/
  services/
    ollama.js          ←  LLM chat, embeddings, category detection
    aiResolver.js      ←  Similarity check, FAQ upsert, broadcast
  routes/
    chat.js            ←  POST /api/chat (text + voice)
  models/
    FAQ.js             ←  question, answer, category, embedding, isAI
    Activity.js        ←  faq_created, ai_response, ai_reuse events
```

**Ollama models used:**
- `qwen2.5:latest` → FAQ answer generation + category detection
- `nomic-embed-text` → 768-dim question embeddings

---

## 🚀 Getting Started

### Prerequisites

- Node.js 18+
- MongoDB (local or Atlas)
- [Ollama](https://ollama.com) v0.24+

### 1. Install Ollama models

```bash
ollama pull qwen2.5:latest      # 7.6B Q4_K_M, ~4.7GB — runs on M3 Metal GPU
ollama pull nomic-embed-text    # 768-dim embeddings, 274MB
ollama serve
```

> **MacBook Air M3:** `qwen2.5:latest` uses ~4.7GB and runs on the Metal GPU. If you have limited RAM, fall back to `qwen2.5:3b`.

### 2. Configure environment

```bash
# server/.env
MONGO_URI=mongodb://localhost:27017/crowd
JWT_SECRET=your-secret-here
PORT=5001
```

```bash
# client/.env
VITE_API_URL=http://localhost:5001
```

### 3. Start backend

```bash
cd server
npm install
node index.js
# → http://localhost:5001
```

### 4. Start frontend

```bash
cd client
npm install
npx vite --host 0.0.0.0
# → http://localhost:5173
```

### 5. Seed demo data

```bash
# server/seeds/data.js contains demo users and categories
node seeds/seed.js   # or run via MongoDB Compass / mongosh
```

**Demo credentials:**
- `admin@crowd.faq` / `password123`
- `demo@crowd.faq` / `password123`

---

## 📡 API Reference

### Chat — AI FAQ Resolution

```
POST /api/chat
Authorization: Bearer <token>
Content-Type: application/json

// Text question
{ "message": "How do I reset my password?" }

// Voice message (base64 WebM audio)
// Voice is transcribed via browser Web Speech API before sending
{ "voiceData": "<base64 audio>", "spokenText": "How do I reset my password?" }
```

**Response:**

```json
{
  "reply":      "To reset your password, navigate to the login page...",
  "faqId":      "6789abc...",
  "source":     "existing | generated",
  "category":   "General",
  "similarity": 0.87,
  "isNew":      false,
  "question":   "How do I reset my password?"
}
```

### FAQ Deduplication Logic

| Similarity | Outcome |
|---|---|
| ≥ 0.82 | Return existing FAQ answer (no new entry created) |
| < 0.82 | LLM generates new answer, saves FAQ with embedding |

---

## 🔌 Real-Time Events (Socket.io)

The server emits on the `activity` channel:

```json
{
  "type": "ai_faq_created",
  "faq": {
    "_id":      "...",
    "question": "How does 2FA improve security?",
    "answer":   "Two-factor authentication...",
    "category": "Cloud/DevOps"
  },
  "createdAt": "2026-05-29T..."
}
```

The FAQ Browser listens for these and prepends new cards to the grid without refresh.

---

## 🗂️ Key Files

| File | Purpose |
|---|---|
| `server/services/ollama.js` | LLM chat, embeddings, category detection, audio transcription |
| `server/services/aiResolver.js` | Core pipeline — similarity check → generate/reuse → broadcast |
| `server/routes/chat.js` | REST endpoint — handles text + voice, returns structured response |
| `client/src/pages/ChatBot.jsx` | UI — voice recording, Web Speech API, markdown rendering, source badge |
| `client/src/pages/FAQBrowser.jsx` | UI — live grid updates via Socket.io, filter/search, pagination |
| `client/src/components/FAQCard.jsx` | UI — AI badge, New sparkle badge with purple glow animation |
| `client/src/context/ToastContext.jsx` | Global toast notification system |

---

## 🔧 Switching LLM Models

To use a different Ollama model, edit `server/services/ollama.js`:

```js
const LLM_MODEL = 'qwen2.5:14b'   // larger, slower, more accurate
// const LLM_MODEL = 'qwen2.5:3b' // smaller, faster, less accurate
```

Restart the server after changing. The `nomic-embed-text` embedding model is fixed — it is the standard Ollama embedding choice.

---

## 🌐 Deployment

### Cloudflare Tunnel (development)

```bash
# Frontend
cloudflared tunnel --url http://localhost:5173

# Backend
cloudflared tunnel --url http://localhost:5001
```

### Production

Set `VITE_API_URL` in `client/.env` to your deployed backend URL and run:

```bash
cd client && npx vite build
# serve dist/ with nginx / Vercel / Netlify
```

Set `origin` in `server/index.js` CORS config to your frontend domain.

---

## 📦 Tech Stack

- **Frontend:** React 18, Vite, React Router, Tailwind CSS, Lucide icons, Socket.io-client
- **Backend:** Node.js, Express, Mongoose, Socket.io, `form-data`
- **LLM:** Ollama (`qwen2.5:latest`, `nomic-embed-text`) — runs locally via Metal GPU (M3)
- **Database:** MongoDB
- **Voice:** Browser Web Speech API (no external service)
- **Auth:** JWT (jsonwebtoken)