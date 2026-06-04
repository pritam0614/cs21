# Crowd FAQ — Architecture

## 1. System Overview

**Type:** Real-time AI-powered FAQ knowledge portal
**Pattern:** Client-server with pub/sub real-time updates
**LLM Strategy:** Local Ollama (dev) / OpenAI (prod) — same service interface

```
┌─────────────────────────────────────────────────────────┐
│  Browser (React SPA)                                     │
│  ┌──────────┐    ┌─────────────┐    ┌──────────────┐   │
│  │ ChatBot  │    │ FAQBrowser  │    │  FAQCard     │   │
│  └────┬─────┘    └──────┬──────┘    └──────┬───────┘   │
│       │                  │                  │           │
│       └────────────┬─────┴──────────────────┘           │
│                    │ Socket.io / REST                    │
└────────────────────┼────────────────────────────────────┘
                     │ HTTPS
┌────────────────────┼────────────────────────────────────┐
│  Express API       │  http://localhost:5001              │
│                    │                                     │
│  ┌─────────────────▼──────────────────┐                 │
│  │  POST /api/chat                    │                 │
│  └─────────────────┬──────────────────┘                 │
│                    │                                     │
│  ┌─────────────────▼──────────────────┐                 │
│  │  aiResolver.js                     │                 │
│  │  1. Embed question                 │                 │
│  │  2. Cosine similarity vs DB       │                 │
│  │  3. Generate answer (if new)      │                 │
│  │  4. Broadcast via Socket.io        │                 │
│  └─────────────────┬──────────────────┘                 │
│                    │                                     │
│  ┌─────────────────▼──────────────────┐                 │
│  │  ollama.js                          │                 │
│  │  LLM: qwen2.5 (chat + category)    │                 │
│  │  Embedding: nomic-embed-text       │                 │
│  └─────────────────────────────────────┘                 │
└──────────────────────────────────────────────────────────┘
                     │
          ┌──────────┴──────────┐
          │                      │
┌─────────▼──────┐    ┌─────────▼──────────┐
│  MongoDB        │    │  Ollama (local)    │
│  crowd_faq      │    │  qwen2.5:7b        │
│  • faqs         │    │  nomic-embed-text  │
│  • users        │    └────────────────────┘
│  • activities   │
│  • categories   │
└─────────────────┘
```

---

## 2. Frontend Architecture

### Stack
React 18, Vite, React Router, Tailwind CSS, Lucide icons, Socket.io-client, Framer Motion

### Pages

| Page | Route | Purpose |
|---|---|---|
| Landing | `/` | Marketing / login entry |
| ChatBot | `/chat` | AI FAQ assistant — text + voice |
| FAQBrowser | `/faqs` | Live-updating FAQ grid |
| Login | `/login` | JWT auth |
| Register | `/register` | User sign-up |

### Key Components

| Component | Purpose |
|---|---|
| `ChatBot.jsx` | Voice recording (Web Speech API), markdown render, source badge |
| `FAQBrowser.jsx` | Grid with real-time Socket.io updates, filter/search |
| `FAQCard.jsx` | 🤖 AI badge, ✨ New sparkle badge (30s), category tag |
| `ToastContext.jsx` | Global toast notifications (purple AI theme) |
| `SocketContext.jsx` | Socket.io event bus provider |

### Real-Time Flow

```
aiResolver saves new FAQ
        ↓
Socket.io emits: { type: 'ai_faq_created', faq }
        ↓
SocketContext receives
        ↓
FAQBrowser prepends card + Toast shows
```

---

## 3. Backend Architecture

### Stack
Node.js, Express, Mongoose, Socket.io, JWT, form-data

### Layers

```
Routes → aiResolver (service) → ollama.js (LLM service)
                          ↓
                    Mongoose Models
                          ↓
                      MongoDB
```

### Directory Structure

```
server/
  index.js              ← Express app + Socket.io + CORS
  .env                  ← Environment config (not committed)
  models/
    User.js             ← name, email, password, role
    FAQ.js              ← question, answer, category, embedding (vector), isAI, createdBy
    Activity.js         ← type, userId, faqId, metadata
    Category.js         ← name, description
  routes/
    auth.js             ← POST /api/auth/register, /api/auth/login
    chat.js             ← POST /api/chat (text + voice)
    faqs.js             ← CRUD /api/faqs, /api/faqs/:id
  services/
    ollama.js           ← Unified LLM interface (Ollama / OpenAI / Anthropic)
    aiResolver.js       ← Core pipeline: embed → similarity → generate/suggest
    auth.js             ← JWT sign + verify
  middleware/
    auth.js             ← Bearer token verification
  seeds/
    seed.js             ← Demo users + categories
```

### API Reference

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/api/auth/register` | No | Create account |
| POST | `/api/auth/login` | No | Returns JWT |
| POST | `/api/chat` | Yes | Text/voice → AI FAQ response |
| GET | `/api/faqs` | Yes | List FAQs (filter, search, pagination) |
| POST | `/api/faqs` | Yes | Create FAQ manually |
| PUT | `/api/faqs/:id` | Yes | Update FAQ |
| DELETE | `/api/faqs/:id` | Yes | Delete FAQ |
| GET | `/api/health` | No | Health check |

### `/api/chat` Contract

**Request (text):**
```json
{ "message": "How do I reset my password?" }
```

**Request (voice — transcribed client-side):**
```json
{ "voiceData": "<base64>", "spokenText": "How do I reset my password?" }
```

**Response:**
```json
{
  "reply":      "Navigate to settings → security → reset password...",
  "faqId":      "6789abc...",
  "source":     "existing | generated",
  "category":   "General",
  "similarity": 0.87,
  "isNew":      false,
  "question":   "How do I reset my password?"
}
```

---

## 4. AI / LLM Architecture

### Local (Dev — Ollama)

```
User question
      ↓
Embed → nomic-embed-text → 768-dim vector
      ↓
Cosine similarity vs all stored FAQ embeddings
(similarity ≥ 0.82 → reuse, < 0.82 → generate new)
      ↓
LLM → qwen2.5:latest → formal FAQ answer + category
      ↓
Save FAQ with embedding to MongoDB
      ↓
Socket.io → Frontend (live update)
```

### Cloud (Prod — OpenAI / Anthropic)

Same pipeline, `ollama.js` routes to OpenAI API instead. Embedding model: `text-embedding-3-small`.

### Ollama Service (`ollama.js`)

- `generateAnswer(question, history)` → FAQ answer string
- `detectCategory(question)` → category name
- `getEmbedding(text)` → 768-dim float array
- `transcribeAudio(buffer)` → text (Whisper via Ollama)

### LLM Model Configuration

| Mode | Provider | Model | Use |
|---|---|---|---|
| Local dev | Ollama | `qwen2.5:7b` or `qwen2.5:3b` | Chat + category |
| Local dev | Ollama | `nomic-embed-text` | Embeddings |
| Production | OpenAI | `gpt-4o-mini` | Chat + category |
| Production | OpenAI | `text-embedding-3-small` | Embeddings |
| Production alt | Anthropic | `claude-3-haiku` | Chat only |

### FAQ Categories (8 total)

`AI/ML` · `Programming` · `Finance` · `Education` · `Healthcare` · `Cloud/DevOps` · `Design` · `General`

---

## 5. Database Architecture

### Collections

**faqs**
```
{
  _id: ObjectId,
  question: String,         ← user query
  answer: String,           ← generated or manual
  category: String,         ← auto-detected or chosen
  embedding: [Float],       ← 768-dim, nomic-embed-text
  isAI: Boolean,            ← true if auto-generated
  createdBy: ObjectId,      ← User ref
  views: Number,            ← view count
  createdAt: Date,
  updatedAt: Date
}
```

**users**
```
{
  _id: ObjectId,
  name: String,
  email: String (unique),
  password: String (hashed),
  role: 'user' | 'admin',
  createdAt: Date
}
```

**activities**
```
{
  _id: ObjectId,
  type: 'faq_created' | 'ai_response' | 'ai_reuse',
  userId: ObjectId,
  faqId: ObjectId (optional),
  metadata: Object,
  createdAt: Date
}
```

**categories**
```
{
  _id: ObjectId,
  name: String (unique),
  description: String
}
```

### Indexes
- `faqs`: 2dsphere index on `embedding` for cosine similarity queries
- `faqs`: text index on `question` + `answer` for keyword search
- `users`: unique index on `email`

---

## 6. Authentication

- **JWT** (jsonwebtoken) — RS256 or HS256
- Tokens expire in 7 days (`JWT_EXPIRES_IN=7d`)
- Bearer token in `Authorization` header for protected routes
- Password hashed with bcrypt
- Auth middleware protects all `/api/*` except `/api/auth/*` and `/api/health`

---

## 7. Real-Time (Socket.io)

### Events Emitted by Server

| Event | Payload | Trigger |
|---|---|---|
| `activity` | `{ type, faq, createdAt }` | New AI FAQ created |

### Client Subscriptions

- `FAQBrowser` → subscribes to `activity`, prepends new FAQ card on `ai_faq_created`
- `ToastContext` → shows toast notification on `ai_faq_created`

---

## 8. Deployment Architecture

### Local Dev
```
Frontend:  http://localhost:5173   (Vite dev server)
Backend:   http://localhost:5001   (Express)
Ollama:    http://localhost:11434  (Ollama serve)
MongoDB:   localhost:27017
```

### Production (Render + MongoDB Atlas)
```
Frontend:  https://crowd-faq-frontend.onrender.com  (static, Render free tier)
Backend:   https://crowd-faq-backend.onrender.com   (Node.js, Render free tier, Port 10000)
LLM:       OpenAI API (gpt-4o-mini + text-embedding-3-small)
DB:        MongoDB Atlas M0 cluster
```

### Render Blueprint
`render.yaml` defines both services. Apply via Render Dashboard → New → Blueprint.

---

## 9. Key Design Decisions

| Decision | Rationale |
|---|---|
| Ollama for local dev | Free, private, no API costs, runs on M3 Metal GPU |
| Cosine similarity ≥ 0.82 threshold | Catches paraphrases without false positives |
| Web Speech API for voice | Zero server round-trip for transcription |
| Socket.io for live updates | FAQ grid updates without page refresh |
| Separate `ollama.js` service | Swap Ollama ↔ OpenAI without changing app logic |
| Embedding stored in MongoDB | Avoids re-embedding on every query |
| 768-dim nomic-embed-text | Good quality / speed tradeoff for M3 |