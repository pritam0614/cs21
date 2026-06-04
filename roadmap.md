# Crowd FAQ — Roadmap

## Overview

Sorted by priority. Each item is marked with status, priority, and effort. The goal is a fully self-sustaining knowledge portal that grows without manual FAQ creation.

---

## Currently Shipped (MVP)

| Feature | Status |
|---|---|
| Text question → AI FAQ answer | ✅ Done |
| Voice input (Web Speech API) | ✅ Done |
| Semantic similarity deduplication (0.82 threshold) | ✅ Done |
| Cosine similarity search | ✅ Done |
| Auto-categorization (8 categories) | ✅ Done |
| Real-time Socket.io FAQ grid updates | ✅ Done |
| Purple toast on new FAQ | ✅ Done |
| AI badge + sparkle New badge | ✅ Done |
| JWT authentication | ✅ Done |
| Manual FAQ CRUD | ✅ Done |
| FAQ search + category filter | ✅ Done |
| Ollama ↔ OpenAI service abstraction | ✅ Done |
| Render deployment blueprint | ✅ Done |

---

## In Progress

| Feature | Owner | Notes |
|---|---|---|
| Production deployment to Render | Nancy | Pending Render account + Atlas setup |

---

## High Priority

### 1. Threaded Chat Conversations

**What:** Multi-turn conversations where the AI remembers prior messages in a thread.

**Why:** FAQ resolution often needs follow-up questions. Currently every message is treated independently.

**Implementation:**
```json
// New model field
{
  "threadId": "abc123",
  "messages": [
    { "role": "user", "content": "How do I reset my password?" },
    { "role": "assistant", "content": "Navigate to settings..." },
    { "role": "user", "content": "What if I forgot my email?" }
  ]
}
```

**Effort:** Medium | **Files:** `server/models/Chat.js`, `server/routes/chat.js`, `client/src/pages/ChatBot.jsx`

---

### 2. FAQ Upvoting / Downvoting

**What:** Users upvote helpful FAQs, downvote inaccurate ones.

**Why:** Builds quality signal, surfaces best answers. Enables future "top FAQ" sorting.

**Schema:**
```js
{
  faqId: ObjectId,
  userId: ObjectId,
  vote: 1 | -1,
  createdAt: Date
}
```
Prevent duplicate votes per user per FAQ (unique compound index).

**Effort:** Low | **Files:** `server/models/Vote.js`, `server/routes/faqs.js`, `FAQCard.jsx`

---

### 3. FAQ View Tracking

**What:** Increment `views` count every time a FAQ card is opened/expanded.

**Why:** Enables "Most Viewed" sort, surfaces popular content.

**Implementation:** `FAQBrowser.jsx` calls `PATCH /api/faqs/:id/view` on expand. Debounce to avoid rapid-fire increments.

**Effort:** Low | **Files:** `server/routes/faqs.js`, `FAQCard.jsx`

---

## Medium Priority

### 4. User Comments on FAQs

**What:** Comment thread on each FAQ for community discussion / corrections.

**Schema:**
```js
{
  faqId: ObjectId,
  userId: ObjectId,
  content: String,     // max 500 chars
  createdAt: Date,
  updatedAt: Date
}
```

**Effort:** Low | **Files:** `server/models/Comment.js`, `server/routes/comments.js`

---

### 5. Multilingual FAQ Responses

**What:** Detect user language, respond in the same language.

**How:** Pass detected BCP-47 language tag to `ollama.js`, add to system prompt: `"Respond in the same language as the user's question."`

**Supported languages (phase 1):** English, Hindi, Spanish, French

**Effort:** Medium | **Files:** `ollama.js`, `ChatBot.jsx` (language detection via `navigator.language`)

---

### 6. Admin Dashboard Analytics

**What:** Admin-only page with:
- FAQs created per day (bar chart)
- Category distribution (pie chart)
- Top 10 most viewed FAQs
- AI reuse rate (% of questions answered by existing FAQs)
- Duplicate detection hit rate

**Stack:** Simple chart component (Recharts or Chart.js), seeded from existing `activities` collection.

**Effort:** Medium | **Files:** `client/src/pages/AdminPanel.jsx`, `server/routes/analytics.js`

---

### 7. Password Reset / Forgot Password

**What:** User submits email → receives reset link → sets new password.

**Stack:** Nodemailer (or SendGrid/Mailgun) for email delivery, random token stored in DB with expiry (1 hour).

**Effort:** Medium | **Files:** `server/routes/auth.js`, email template, `client/src/pages/ForgotPassword.jsx`

---

## Lower Priority

### 8. RAG Pipeline — FAQ from Documents

**What:** Upload a PDF/Doc and have AI extract and generate FAQs from it.

**How:**
1. User uploads file → server extracts text (pdf-parse or mammoth)
2. Chunk text into sections (500 tokens each)
3. For each chunk: generate 2-3 FAQ Q&A pairs via LLM
4. Save to `faqs` collection with `source: 'document'`

**Effort:** High | **Files:** `server/routes/docs.js`, PDF parser dependency, new UI for upload

---

### 9. Redis Caching for Hot FAQs

**What:** Cache the 100 most-viewed FAQ embeddings in Redis.

**Why:** Avoid hitting MongoDB for every similarity search. Speed up FAQ resolution.

**Stack:** Redis (Upstash free tier), cache key: `faq:embedding:<faqId>`, TTL: 1 hour.

**Effort:** Medium | **Files:** `server/services/aiResolver.js`

---

### 10. Text-to-Speech for Answers

**What:** User taps a "Listen" button on a FAQ answer to hear it spoken aloud.

**How:** Browser [SpeechSynthesis API](https://developer.mozilla.org/en-US/docs/Web/API/SpeechSynthesis) — no external service needed.

**Effort:** Low | **Files:** `FAQCard.jsx` (add play/pause button)

---

## Future Speculative

| Feature | Notes |
|---|---|
| **Pinecone / Weaviate vector DB** | Replace MongoDB embeddings with a dedicated vector DB for faster similarity search at scale |
| **Personalized FAQ feed** | Recommend FAQs based on user's question history |
| **AI summarization** | "Summarize this FAQ in one sentence" button |
| **Mobile PWA** | Service worker + manifest for offline-capable mobile app |
| **Export FAQ as PDF** | Generate a downloadable FAQ document |
| **Slack / Discord bot** | Ask questions from Slack, get AI answers back |
| **API for third-party integrations** | Developers can query the FAQ knowledge base via REST |

---

## Version Plan

| Version | Milestone | Target |
|---|---|---|
| **v0.1** | MVP — core FAQ flow, local Ollama | ✅ Done |
| **v0.2** | Production deploy, OpenAI, seeded data | In progress |
| **v1.0** | Chat threads, voting, comments, analytics | Next |
| **v1.1** | Multilingual, TTS, password reset | Near-term |
| **v1.2** | RAG pipeline, document upload | Mid-term |
| **v2.0** | Vector DB, caching, mobile PWA | Long-term |

---

## Contributing to the Roadmap

New features should be:
1. Described in an GitHub Issue with: problem statement, proposed solution, estimated effort
2. Approved by the project owner before implementation
3. Added to this file with status and effort tags

No feature gets merged without a corresponding update to this roadmap.