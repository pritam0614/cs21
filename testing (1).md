# Crowd FAQ — Testing Guide

## Testing Philosophy

Test the critical paths: auth works, AI resolves questions, FAQs deduplicate correctly, real-time updates fire. Coverage over edge cases at the unit level; critical user flows at the integration level.

---

## Test Environments

| Env | When | Setup |
|---|---|---|
| **Local dev** | Every commit | `npm run dev` in both client + server |
| **Staging (future)** | Pre-production | Deploy to a staging Render app |
| **Production** | Final verification | `https://crowd-faq-frontend.onrender.com` |

---

## Manual Test Cases

Run through these after every meaningful code change.

### Auth

| # | Test | Steps | Expected |
|---|---|---|---|
| A1 | Register with valid data | Submit name + email + 8+ char password | `201`, token returned, redirect to chat |
| A2 | Register with duplicate email | Re-register with same email | `409 Conflict` error shown |
| A3 | Register with weak password | Submit 5-char password | `400` validation error |
| A4 | Login with correct credentials | `demo@crowd.faq` / `password123` | `200`, JWT in response |
| A5 | Login with wrong password | Correct email, wrong password | `401 Unauthorized` |
| A6 | Login with unregistered email | Any email not in DB | `401 Unauthorized` |
| A7 | Protected route without token | Call `/api/chat` with no `Authorization` header | `401 Unauthorized` |
| A8 | Protected route with expired token | Call with old/invalid JWT | `401 Unauthorized` |

---

### AI Chat (`POST /api/chat`)

| # | Test | Steps | Expected |
|---|---|---|---|
| C1 | Text question — new FAQ | Ask something unique: "How does blockchain confirm transactions?" | `source: "generated"`, `isNew: true`, FAQ saved in DB |
| C2 | Text question — duplicate detected | Ask a paraphrase of an existing FAQ | `source: "existing"`, `isNew: false`, no duplicate saved |
| C3 | Exact duplicate | Ask an exact question from existing FAQ | `source: "existing"`, similarity > 0.95 |
| C4 | Empty message | Send `{ "message": "" }` | `400` validation error |
| C5 | Very long question | Send 2000+ character question | Accepted or truncated gracefully |
| C6 | Unauthenticated chat | Call without JWT | `401 Unauthorized` |
| C7 | Voice message | Send `{ "voiceData": "...", "spokenText": "..." }` | Same response as text |
| C8 | Ollama offline | Stop Ollama (`pkill -f ollama`), send question | `503 Service Unavailable` |
| C9 | Category assigned | Ask a programming question | `category: "Programming"` |
| C10 | Non-English question | Ask in Hindi or Spanish | Returns answer (if multilingual prompt active) |

---

### FAQ Browser

| # | Test | Steps | Expected |
|---|---|---|---|
| B1 | Load FAQs | Open `/faqs` while logged in | Grid of FAQ cards loads |
| B2 | Empty state | Clear DB, open `/faqs` | "No FAQs yet. Be the first to ask!" message |
| B3 | Search by keyword | Type "password" in search | Only FAQs matching "password" shown |
| B4 | Search no results | Search for "xyzabc123" | "No results" empty state |
| B5 | Category filter | Select "AI/ML" category | Only AI/ML FAQs shown |
| B6 | Filter + search combined | Filter "Programming" + search "sort" | Only programming FAQs matching "sort" |
| B7 | Reset filter | Clear category filter | All categories shown again |
| B8 | Pagination | Create 25+ FAQs, visit `/faqs` | Page 1 of 20, pagination controls visible |

---

### FAQ Cards & Real-Time

| # | Test | Steps | Expected |
|---|---|---|---|
| R1 | AI badge visible | View a known AI-generated FAQ card | 🤖 badge visible |
| R2 | No AI badge on manual | View a manually created FAQ | No 🤖 badge |
| R3 | New badge appears | Create FAQ via chat, immediately check browser | ✨ sparkle badge + purple glow |
| R4 | New badge fades | Wait 35 seconds | ✨ badge gone |
| R5 | Toast fires | Create new FAQ via chat | Purple toast slides in from top-right |
| R6 | Real-time update | Open `/faqs` in two tabs, ask in one | Second tab updates without refresh |
| R7 | Card expand | Click any FAQ card | Expands to show full answer |
| R8 | Category badge color | Each category shows distinct color | Consistent color per category |

---

### Manual FAQ CRUD

| # | Test | Steps | Expected |
|---|---|---|---|
| U1 | Create FAQ | POST `/api/faqs` with question + answer + category | `201`, FAQ saved, no embedding needed |
| U2 | Create with missing fields | POST with no answer | `400` validation error |
| U3 | Update own FAQ | PUT `/api/faqs/:id` with new answer | `200`, answer updated |
| U4 | Delete own FAQ | DELETE `/api/faqs/:id` | `200`, FAQ removed from grid |
| U5 | Delete other's FAQ (user) | Try to delete another user's FAQ | `403 Forbidden` |
| U6 | Delete any FAQ (admin) | Login as admin, delete any FAQ | `200` regardless of owner |
| U7 | Get single FAQ | GET `/api/faqs/:id` | Full FAQ object with `createdBy` |

---

### Ollama & LLM Service

| # | Test | Steps | Expected |
|---|---|---|---|
| L1 | Ollama running | `curl localhost:11434/api/tags` | JSON with model list |
| L2 | Embedding model available | Ask any question via chat | Embedding generated, similarity computed |
| L3 | Model not found | Remove model, ask question | `404` from Ollama, `503` to user |
| L4 | LLM returns empty | Force empty response (test error path) | Graceful error: "Could not generate answer" |
| L5 | Switch to OpenAI | Set `LLM_PROVIDER=openai`, `OPENAI_API_KEY=...` | Same behavior, different provider |
| L6 | Switch back to Ollama | Revert env vars | Falls back to local Ollama |

---

### Production Smoke Tests

| # | Test | Steps | Expected |
|---|---|---|---|
| P1 | Backend health | `curl https://crowd-faq-backend.onrender.com/api/health` | `200` with `"status": "ok"` |
| P2 | Frontend loads | Open browser to Render URL | Page loads, no white screen |
| P3 | Login works | Login on production URL | JWT returned, redirected to chat |
| P4 | Chat works production | Ask a question on production | AI answer returned, FAQ appears in browser |
| P5 | Real-time works | Ask in one browser, check another | Socket.io update fires |
| P6 | Cold start | Visit after 15 min idle | ~30s delay on first request |

---

## Automated Testing (Future)

### Unit Tests — `server/`

```bash
npm test   # jest
```

Priority test files:
- `services/ollama.js` — mock Ollama responses, verify routing
- `services/aiResolver.js` — mock DB, verify dedup logic
- `middleware/auth.js` — valid/invalid/expired JWT cases

### Unit Tests — `client/`

```bash
npm test   # vitest + @testing-library/react
```

Priority test files:
- `pages/ChatBot.jsx` — send button disabled when empty
- `context/ToastContext.jsx` — toast queue management

### Integration Tests

- API flow: login → chat → verify FAQ saved
- Real-time: create FAQ → verify Socket.io event received

### E2E Tests (Playwright)

```bash
npx playwright test
```

Cover:
- Register → login → ask question → verify FAQ in browser
- Duplicate detection: ask same question twice → verify one FAQ only

---

## Test Data

### Seeded Accounts

| Email | Password | Role |
|---|---|---|
| `admin@crowd.faq` | `password123` | admin |
| `demo@crowd.faq` | `password123` | user |

### Seeded Categories

`AI/ML`, `Programming`, `Finance`, `Education`, `Healthcare`, `Cloud/DevOps`, `Design`, `General`

### Test Questions (for deduplication testing)

Use these to test similarity thresholds:

```
Q1: "How do I reset my password?"
Q2: "How do I change my password?"        → Should match Q1 (similarity > 0.82)
Q3: "How do I cook perfect rice?"         → Should NOT match Q1 (similarity < 0.82)
Q4: "What is the capital of France?"
Q5: "What is France's capital city?"      → Should match Q4
Q6: "What is the best way to learn React?"
Q7: "How do I learn React framework?"     → Likely matches Q6
```

### Fixtures

Store reusable test data in `server/tests/fixtures/`:
- `demoUser.json` — login credentials
- `sampleFaqs.json` — pre-seeded FAQ objects for testing

---

## Performance Benchmarks

| Metric | Target | How to Measure |
|---|---|---|
| FAQ chat response (local Ollama) | < 3s | `console.time` in `aiResolver.js` |
| Embedding generation | < 500ms | Log `getEmbedding()` duration |
| Similarity search | < 200ms | Log MongoDB query time |
| Frontend page load (prod) | < 2s | Chrome DevTools Network tab |
| Socket.io event delivery | < 100ms | DevTools WebSocket frames |

---

## Bug Report Template

When filing a bug, include:

```markdown
## Steps to Reproduce
1. Go to ...
2. Click on ...
3. Enter ...

## Expected Behavior
...

## Actual Behavior
...

## Environment
- Browser: Chrome/Firefox/Safari
- OS: macOS/Windows
- Local or production
- Ollama running: yes/no

## Console Errors
```
[paste error here]
```

## Screenshots
[attach if UI bug]
```