# Contributing to Crowd FAQ

## Development Setup

### 1. Fork & Clone

```bash
git clone https://github.com/Nancypaul08/crowd-faq.git
cd crowd-faq/mvp
```

### 2. Install Dependencies

```bash
# Backend
cd server && npm install

# Frontend
cd ../client && npm install
```

### 3. Start Infrastructure

```bash
# MongoDB
mongod --dbpath /usr/local/var/mongodb

# Ollama (Mac)
ollama serve

# Pull required models
ollama pull qwen2.5:3b
ollama pull nomic-embed-text
```

### 4. Configure Environment

```bash
# server/.env
cp .env.production.example .env
# Fill in:
#   MONGO_URI=mongodb://localhost:27017/crowd_faq
#   JWT_SECRET=any-secret-here
#   PORT=5001
#   LLM_PROVIDER=ollama
```

```bash
# client/.env
VITE_API_URL=http://localhost:5001
```

### 5. Seed Demo Data

```bash
cd server && node seeds/seed.js
```

**Demo credentials:**
- `admin@crowd.faq` / `password123` (admin role)
- `demo@crowd.faq` / `password123` (user role)

### 6. Run

```bash
# Terminal 1 — Backend
cd server && npm run dev   # nodemon auto-restarts on changes

# Terminal 2 — Frontend
cd client && npm run dev

# Frontend: http://localhost:5173
# Backend:  http://localhost:5001
```

---

## Code Standards

### JavaScript / Node.js

- **No semicolons** — use ASI (standard in this project)
- **2-space indent**
- **Template literals** for string interpolation
- **`const` first**, `let` only when reassignment needed, never `var`
- **Async/await** over raw Promises or `.then()` chains
- **Descriptive names:** `generateAnswer`, not `genAns` or `gA`

### React

- **Functional components** + hooks only (no class components)
- **PropTypes** or TypeScript for component props
- **`use` prefix** for custom hooks: `useToast()`, `useAuth()`
- **Co-locate** related files: `ChatBot/` folder with its component + styles + tests

### File Naming

```
PascalCase — React components:  FAQCard.jsx, ChatBot.jsx
camelCase  — Hooks, utils:      useSocket.js, auth.js
kebab-case — Pages:             FAQBrowser.jsx, Login.jsx
```

---

## Architecture Rules

### Don't Call LLM Directly

All LLM calls go through `server/services/ollama.js`. Never import Ollama or OpenAI directly in routes or controllers.

```
✅ server/services/ollama.js → generateAnswer()
❌ Routes should NOT call fetch(localhost:11434)
```

### Socket.io Only in Context

Socket.io client should only be used through `SocketContext.jsx` and `ToastContext.jsx`. Don't call `io()` directly in page components.

### Auth Middleware on All Protected Routes

Every `/api/` route except `/api/auth/*` and `/api/health` must use the `auth` middleware:

```js
const auth = require('../middleware/auth');
router.post('/chat', auth, require('./chat'));
```

---

## Commit Conventions

Format: `<type>: <short description>`

### Types

| Type | Use for |
|---|---|
| `feat` | New feature (AI chat, voice input, etc.) |
| `fix` | Bug fix (syntax error, crash, etc.) |
| `docs` | Documentation only |
| `refactor` | Code change with no feature/fix (restructure, rename) |
| `style` | Formatting, whitespace (no logic change) |
| `perf` | Performance improvement |
| `test` | Adding or updating tests |
| `chore` | Build, deps, CI, tooling |

### Examples

```bash
git commit -m "feat: add voice input via Web Speech API"
git commit -m "fix: resolve syntax error in ollama.js messages array"
git commit -m "docs: add API reference for POST /api/chat"
git commit -m "refactor: extract SocketContext from App.jsx"
git commit -m "fix: correct CORS origin for production"
```

### Rules

- One commit = one logical change (don't mix feat + fix in one commit)
- Subject line ≤ 72 characters
- No emoji in commit messages
- Use present tense: "add" not "added"

---

## Branch Strategy

```
main          — production-ready code (protected)
cs21-restore  — current development branch (your work)
```

**Create feature branches from `cs21-restore`:**

```bash
git checkout cs21-restore
git checkout -b feat/voice-input
# work work work
git commit -m "feat: ..."
git push origin feat/voice-input
# Open PR → review → merge into cs21-restore
```

**Branch naming:**
- `feat/<name>` — new feature
- `fix/<name>` — bug fix
- `docs/<name>` — documentation
- `refactor/<name>` — code restructuring

---

## Pull Request Process

1. **PR title** follows commit convention: `feat: add voice input`
2. **Description** explains WHAT changed and WHY
3. **Screenshots** for UI changes (before + after)
4. **Tested locally** — describe what you tested
5. **No merge conflicts** with `cs21-restore`
6. At least one reviewer before merge (if collaborating)

### PR Checklist

- [ ] All new code follows project style guidelines
- [ ] `npm run build` succeeds on frontend (no Vite errors)
- [ ] Backend starts without errors (`node index.js`)
- [ ] No `console.log` left in production paths
- [ ] New environment variables documented in `.env.production.example`
- [ ] New routes protected by auth middleware

---

## Directory Structure

```
mvp/
  client/
    src/
      pages/          ← Route-level components
      components/     ← Reusable UI components
      context/        ← React context (Socket, Toast, Auth)
      config/         ← Static config
      App.jsx         ← Router setup
  server/
    models/           ← Mongoose schemas
    routes/           ← Express routes (thin — delegate to services)
    services/         ← Business logic (aiResolver, ollama, auth)
    middleware/       ← Auth, validation
    seeds/            ← Database seeding
    index.js          ← Express app entry
```

---

## Adding a New Route

1. Create `server/routes/myFeature.js`
2. Write handlers using existing services
3. Add `module.exports = router`
4. Mount in `server/index.js`: `app.use('/api/myFeature', require('./routes/myFeature'))`
5. Protect with `auth` middleware if needed
6. Document in `api.md`

---

## Testing

### Manual Testing Checklist

When changing AI behavior, test these cases:

- **New question** → new FAQ created, Socket.io fires, toast shows
- **Near-duplicate question** → existing FAQ returned, no new entry
- **Exact match** → existing FAQ returned with high similarity
- **Empty message** → API returns 400
- **Unauthenticated request** → API returns 401
- **Voice input** → transcription sent, answer generated
- **Ollama offline** → API returns 503 gracefully

### Test Data

Use the seeded demo accounts:
- `admin@crowd.faq` / `password123`
- `demo@crowd.faq` / `password123`

---

## Dependency Management

### Adding a new dependency

```bash
cd client && npm install package-name
# or
cd server && npm install package-name
```

### Auditing

```bash
cd server && npm audit
cd client && npm audit
```

Fix vulnerabilities before merging:
```bash
npm audit fix
```

### Updating Ollama models

```bash
ollama pull qwen2.5:latest
ollama pull nomic-embed-text
```

Document any model changes in `README.md` and `techstack.md`.