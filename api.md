# Crowd FAQ — API Reference

Base URL: `http://localhost:5001` (dev) · `https://crowd-faq-backend.onrender.com` (prod)

All requests except auth and health require `Authorization: Bearer <jwt_token>`.

---

## Auth

### `POST /api/auth/register`

Create a new user account.

**Request:**
```json
{
  "name": "Aria Demo",
  "email": "aria@crowd.faq",
  "password": "password123"
}
```

**Response `201`:**
```json
{
  "success": true,
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "_id": "6789abc...",
    "name": "Aria Demo",
    "email": "aria@crowd.faq",
    "role": "user"
  }
}
```

**Errors:**
- `400` — Validation error (missing fields, invalid email)
- `409` — Email already registered

---

### `POST /api/auth/login`

Authenticate and receive a JWT.

**Request:**
```json
{
  "email": "demo@crowd.faq",
  "password": "password123"
}
```

**Response `200`:**
```json
{
  "success": true,
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "_id": "6789abc...",
    "name": "Demo User",
    "email": "demo@crowd.faq",
    "role": "user"
  }
}
```

**Errors:**
- `400` — Missing email or password
- `401` — Invalid credentials

---

## Chat (AI FAQ Resolution)

### `POST /api/chat`

Ask a question — gets an AI FAQ answer, either from an existing match or a newly generated FAQ.

**Headers:**
```
Authorization: Bearer <token>
Content-Type: application/json
```

**Request — text:**
```json
{
  "message": "How do I reset my password?"
}
```

**Request — voice (pre-transcribed client-side):**
```json
{
  "voiceData": "<base64-encoded audio>",
  "spokenText": "How do I reset my password?"
}
```

**Response `200`:**
```json
{
  "reply":      "To reset your password, navigate to the login page and click 'Forgot Password'. Enter your registered email address and follow the link sent to your inbox. The link expires after 24 hours.",
  "faqId":      "6789abc...",
  "source":     "generated",
  "category":   "General",
  "similarity": 0.0,
  "isNew":      true,
  "question":   "How do I reset my password?"
}
```

**When an existing FAQ is matched (`similarity ≥ 0.82`):**
```json
{
  "reply":      "Navigate to the settings page and click 'Security' to find the password reset option.",
  "faqId":      "6789def...",
  "source":     "existing",
  "category":   "General",
  "similarity": 0.87,
  "isNew":      false,
  "question":   "How do I reset my password?"
}
```

**Response fields:**

| Field | Type | Description |
|---|---|---|
| `reply` | string | FAQ answer text (markdown supported) |
| `faqId` | string | MongoDB ObjectId of the matched/created FAQ |
| `source` | string | `"existing"` or `"generated"` |
| `category` | string | One of 8 predefined categories |
| `similarity` | number | Cosine similarity score (0.0–1.0) |
| `isNew` | boolean | `true` if a new FAQ was created |
| `question` | string | The question that was asked |

**Errors:**
- `400` — Empty `message`
- `401` — Missing or invalid JWT
- `503` — Ollama not running / LLM service unavailable

---

## FAQs

### `GET /api/faqs`

List FAQs with optional filtering, search, and pagination.

**Query parameters:**

| Param | Type | Default | Description |
|---|---|---|---|
| `page` | number | `1` | Page number |
| `limit` | number | `20` | Items per page (max 100) |
| `category` | string | — | Filter by category name |
| `search` | string | — | Keyword search (MongoDB text index) |
| `sort` | string | `newest` | `newest` or `views` |

**Response `200`:**
```json
{
  "faqs": [
    {
      "_id":        "6789abc...",
      "question":   "How does 2FA improve security?",
      "answer":     "Two-factor authentication...",
      "category":   "Cloud/DevOps",
      "isAI":       true,
      "views":      42,
      "createdAt":  "2026-05-29T12:00:00.000Z"
    }
  ],
  "total": 156,
  "page":  1,
  "pages": 8
}
```

---

### `POST /api/faqs`

Create a FAQ manually (authenticated).

**Request:**
```json
{
  "question": "What is the refund policy?",
  "answer":   "Refunds are processed within 5-7 business days...",
  "category": "General"
}
```

**Response `201`** — Created FAQ object.

---

### `PUT /api/faqs/:id`

Update an existing FAQ.

**Request (partial):**
```json
{
  "answer":   "Updated answer text...",
  "category": "Finance"
}
```

**Response `200`** — Updated FAQ object.

**Errors:**
- `404` — FAQ not found
- `403` — Not the FAQ owner (non-admin)

---

### `DELETE /api/faqs/:id`

Delete a FAQ. Admin can delete any; users can only delete their own.

**Response `200`:**
```json
{ "success": true, "message": "FAQ deleted" }
```

**Errors:**
- `404` — FAQ not found
- `403` — Not authorized

---

### `GET /api/faqs/:id`

Get a single FAQ by ID.

**Response `200`:**
```json
{
  "_id":        "6789abc...",
  "question":   "How does 2FA improve security?",
  "answer":     "Two-factor authentication...",
  "category":   "Cloud/DevOps",
  "isAI":       true,
  "views":      43,
  "createdBy":  { "_id": "...", "name": "Demo User" },
  "createdAt":  "2026-05-29T12:00:00.000Z"
}
```

---

## Categories

### `GET /api/categories`

List all categories.

**Response `200`:**
```json
{
  "categories": [
    { "_id": "...", "name": "AI/ML",       "description": "Artificial Intelligence and Machine Learning" },
    { "_id": "...", "name": "Programming", "description": "Software development and coding" },
    { "_id": "...", "name": "Finance",     "description": "Financial topics and accounting" },
    { "_id": "...", "name": "Education",   "description": "Academic and learning topics" },
    { "_id": "...", "name": "Healthcare",  "description": "Health and medical information" },
    { "_id": "...", "name": "Cloud/DevOps","description": "Cloud infrastructure and DevOps" },
    { "_id": "...", "name": "Design",      "description": "UI/UX and product design" },
    { "_id": "...", "name": "General",     "description": "Miscellaneous topics" }
  ]
}
```

---

## System

### `GET /api/health`

Health check (no auth required).

**Response `200`:**
```json
{
  "status":   "ok",
  "mongodb":  "connected",
  "ollama":   "available",
  "timestamp": "2026-06-04T10:30:00.000Z"
}
```

---

## Socket.io Events

Connect to `http://localhost:5001` (dev) or `https://crowd-faq-backend.onrender.com` (prod).

### Server → Client

**`activity`**

Fired when an AI FAQ is created. Frontend FAQ Browser listens for this.

```json
{
  "type":      "ai_faq_created",
  "faq": {
    "_id":       "6789abc...",
    "question":  "How does 2FA improve security?",
    "answer":    "Two-factor authentication adds a second verification step...",
    "category":  "Cloud/DevOps",
    "isAI":      true,
    "views":     0,
    "createdAt": "2026-06-04T10:30:00.000Z"
  },
  "createdAt": "2026-06-04T10:30:00.000Z"
}
```

### Client → Server

No client-initiated events required. Connection alone is sufficient; the server pushes updates.

---

## Error Response Format

All errors follow this structure:

```json
{
  "success": false,
  "error":   "Descriptive error message",
  "code":    "VALIDATION_ERROR"   // optional, internal code
}
```

HTTP status codes: `400` (bad request), `401` (unauthorized), `403` (forbidden), `404` (not found), `409` (conflict), `500` (server error), `503` (service unavailable).