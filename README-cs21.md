# Crowd FAQ — AI-Powered FAQ Knowledge Portal

> **Branch:** `cs21-restore` · **Commit:** `c6eec3d` · **Status:** ✅ Pushed

---

## 🧠 What It Does

```
User asks a question (text or voice)
         ↓
   ┌──────────────────────────────────────┐
   │  1. Generate answer immediately       │
   │  2. Embed question (nomic-embed-text) │
   │  3. Cosine similarity against FAQs    │
   └──────────────────────────────────────┘
              ↓                              ↓
      Similarity ≥ 0.82              No close match
              ↓                              ↓
    Return existing answer       →  LLM generates formal answer
    Mark FAQ as trending              (qwen2.5:3b on Metal GPU)
    (pins to top for 90 min)              ↓
                               FAQ auto-categorized (8 categories)
                                           ↓
                               Saved to MongoDB with embedding
                                           ↓
                               Socket.io → browser updates live
```

---

## ✨ Features

| Feature | Description |
|---|---|
| **Always-Generate-First** | Answer is generated before duplicate check — user never waits on similarity search |
| **Duplicate Detection** | Cosine similarity ≥ 0.82 → no new FAQ created; existing FAQ is marked trending |
| **Trending FAQs** | Duplicate FAQs pin to top of Browse for 90 min, then return to normal position |
| **DuplicateBanner UI** | Amber toast showing "already exists / moved to top of Browse FAQ" |
| **Local LLM** | `qwen2.5:3b` via Ollama — no API costs, fully offline |
| **Semantic Search** | Cosine similarity against all FAQ embeddings — catches paraphrases |
| **Auto-categorization** | LLM classifies into 8 categories: AI/ML, Programming, Finance, Education, Healthcare, Cloud/DevOps, Design, General |
| **Voice Input** | Browser Web Speech API (Chrome/Edge) — real-time transcription |
| **Live Grid Updates** | New AI FAQs appear in FAQ Browser instantly via Socket.io |
| **Toast Notifications** | Purple AI-themed toast every time a new FAQ is auto-created |

---

## 🏗️ Architecture

```
client/                     server/
  src/
    pages/
      ChatBot.jsx      ←  AI FAQ assistant (text + voice)
      FAQBrowser.jsx   ←  Live-updating FAQ grid
    components/
      FAQCard.jsx      ←  Card with AI/Trending badges
      Layout.jsx       ←  Flex layout with conditional padding
      Sidebar.jsx      ←  Hides hamburger on /faqs (PageHeader owns it)
      ToastContext.jsx ←  Global toast notifications
      SocketContext.jsx←  Socket.io event bus

server/
  services/
    ollama.js          ←  LLM chat, embeddings, category detection
    aiResolver.js      ←  Always-generate-first pipeline
  routes/
    chat.js            ←  POST /api/chat
    faqs.js            ←  GET /api/faqs (trending-aware sort)
  models/
    FAQ.js             ←  question, answer, category, embedding,
                          isTrending, trendingUntil
```

**Ollama models:**
- `qwen2.5:3b` → FAQ answer generation + category detection
- `nomic-embed-text` → 768-dim question embeddings

---

## 🚀 Getting Started

### Prerequisites

- Node.js 18+
- MongoDB (local or Atlas)
- [Ollama](https://ollama.com) v0.24+

### 1. Install Ollama models

```bash
ollama pull qwen2.5:3b
ollama pull nomic-embed-text
ollama serve
```

### 2. Configure environment

```bash
# server/.env
MONGO_URI=mongodb://localhost:27017/crowd
JWT_SECRET=***
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

---

## 📡 API Reference

### POST /api/chat

```json
{ "message": "How do I reset my password?" }
```

**Response:**

```json
{
  "reply":      "To reset your password...",
  "faqId":      "6789abc...",
  "source":     "existing | generated",
  "isDuplicate": true,
  "category":   "General",
  "similarity": 0.87,
  "isNew":      false
}
```

---

## 🔌 Real-Time Events (Socket.io)

```json
{
  "type": "ai_faq_created",
  "faq": { "_id": "...", "question": "...", "answer": "...", "category": "..." }
}
```

---

## 🔑 Key Changes in This Commit

- **Always-generate-first:** `aiResolver.resolveFAQ()` now generates the answer before checking duplicates — the user gets a response instantly regardless of DB size
- **Trending duplicates:** FAQ model gains `isTrending` + `trendingUntil` fields; duplicates pin to top of Browse for 90 min
- **FAQ Browser header:** Fixed overlapping hamburger — PageHeader uses flex row with 20px gap, article count below title, no hardcoded paddingLeft
- **Sidebar:** Hides its hamburger on `/faqs` to avoid duplicates with PageHeader
- **db.js:** Added `bufferCommands: false` + explicit timeouts to prevent mongoose query buffering hangs on reconnection
- **New files:** `ChatBot.css`, `FAQDetail.jsx`, `Landing.css`

---

## 📦 Tech Stack

- **Frontend:** React 18, Vite, Tailwind CSS, Framer Motion, Socket.io-client, Lucide icons
- **Backend:** Node.js, Express, Mongoose, Socket.io
- **LLM:** Ollama (`qwen2.5:3b`, `nomic-embed-text`) — runs locally
- **Database:** MongoDB
- **Voice:** Browser Web Speech API (no external service)
- **Auth:** JWT (jsonwebtoken)