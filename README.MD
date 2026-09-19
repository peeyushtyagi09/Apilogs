# APILogs

> **A full-stack, real-time API monitoring platform.** External services send structured log events via an authenticated REST endpoint; a Socket.IO layer fans those events out to subscribed browser dashboards in real time; an incident-grouping algorithm automatically coalesces repeated errors into trackable incidents.

---

## Table of Contents

1. [What It Does & Why It Exists](#1-what-it-does--why-it-exists)
2. [System Architecture](#2-system-architecture)
3. [Tech Stack](#3-tech-stack)
4. [Key Engineering Decisions](#4-key-engineering-decisions)
5. [Repository Structure](#5-repository-structure)
6. [Backend — Complete API Reference](#6-backend--complete-api-reference)
7. [Backend — Startup & Request Lifecycle](#7-backend--startup--request-lifecycle)
8. [Database Models & Schema Design](#8-database-models--schema-design)
9. [Authentication & Security Architecture](#9-authentication--security-architecture)
10. [Real-Time Architecture (Socket.IO)](#10-real-time-architecture-socketio)
11. [Frontend Architecture](#11-frontend-architecture)
12. [Event Ingestion Flow (End-to-End)](#12-event-ingestion-flow-end-to-end)
13. [Incident Grouping Algorithm](#13-incident-grouping-algorithm)
14. [Environment Variables Reference](#14-environment-variables-reference)
15. [Known Bugs & Implementation Gaps](#15-known-bugs--implementation-gaps)
16. [Interview Preparation — Deep Technical Q&A](#16-interview-preparation--deep-technical-qa)

---

## 1. What It Does & Why It Exists

Modern services produce vast amounts of structured log data. APILogs solves three problems:

| Problem | APILogs Solution |
|---|---|
| Scattered, opaque service logs | A single ingestion endpoint accepts structured events (`service`, `severity`, `message`, `metadata`, `environment`) from any backend service via a simple HTTP POST |
| No visibility into error patterns | An **incident-grouping engine** normalises error messages and coalesces repeated occurrences into a single tracked `Incident`, so engineers see "this error has happened 47 times" rather than 47 noise entries |
| Delayed awareness of failures | A **Socket.IO real-time layer** pushes new events and incident updates to all subscribed dashboard sessions the instant an event is ingested — no polling, no refresh |

**Who uses it:** Two roles:
- **External services** (backends, SDKs) — authenticate with a per-project SHA-256 API key, POST events to `/api/events/ingest/:projectId`
- **Dashboard users** — authenticate with JWT (email/password or passwordless OTP), subscribe to a project room over WebSocket, see events and incidents live

---

## 2. System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              EXTERNAL SERVICES                               │
│   payment-service, auth-service, api-gateway, etc.                           │
│   POST /api/events/ingest/:projectId   (x-api-key: <sha256-hashed key>)     │
└─────────────────────┬───────────────────────────────────────────────────────┘
                      │ HTTP
                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           EXPRESS BACKEND (Node.js)                          │
│                                                                               │
│  ┌─────────────┐   ┌──────────────────┐   ┌──────────────────────────────┐  │
│  │  app.js     │   │  Route Modules   │   │  Middleware Chain             │  │
│  │  (factory)  │──▶│  /api/auth       │──▶│  cors → helmet →             │  │
│  │             │   │  /api/project    │   │  express.json(10kb) →        │  │
│  │  Helmet     │   │  /api/events     │   │  performanceTimer(>500ms)    │  │
│  │  CORS       │   │  /api/incidents  │   │                              │  │
│  │  JSON 10kb  │   │  /api/system     │   │  authRequired (JWT)          │  │
│  └─────────────┘   └──────────────────┘   │  apiKeyAuth (SHA-256)        │  │
│                                            │  projectRateLimiter (300/min)│  │
│                                            │  otpLimiter (5/10min IP)     │  │
│                                            └──────────────────────────────┘  │
│                                                                               │
│  ┌──────────────────────────────┐   ┌──────────────────────────────────────┐ │
│  │  Controllers                 │   │  Socket.IO Server                    │ │
│  │  auth · project · event      │──▶│  socket.server.js  (init+getIO)      │ │
│  │  incident · system           │   │  socket.auth.js    (JWT verify)      │ │
│  │                              │   │  socket.manager.js (rooms+handlers)  │ │
│  │  Utils                       │   │                                      │ │
│  │  makeOtp() · sendOtpEmail()  │   │  Rooms: "project:<projectId>"        │ │
│  │  MetricsStore (singleton)    │   │  Events: new-event, incident-updated │ │
│  └──────────────────────────────┘   └──────────────────────────────────────┘ │
└──────────────────────────────┬─────────────────────┬───────────────────────-─┘
                               │                     │
                               ▼                     ▼ WebSocket
                     ┌─────────────────┐   ┌──────────────────────────────────┐
                     │    MongoDB      │   │       REACT DASHBOARD            │
                     │                 │   │                                  │
                     │  User           │   │  AuthContext ─── ProjectContext  │
                     │  Project        │   │  EventContext ── RealtimeContext │
                     │  Event          │   │                                  │
                     │  Incident       │   │  ActivityFeed (react-window)     │
                     │  OtpToken       │   │  IncidentList + IncidentSummary  │
                     │  AuditLog       │   │                                  │
                     └─────────────────┘   │  Vite + React 19 + TailwindCSS  │
                                           └──────────────────────────────────┘
```

### Data flows

| Flow | Path |
|---|---|
| Event ingestion | SDK → `POST /api/events/ingest/:projectId` → `apiKeyAuth` → `projectRateLimiter` → `validateEvent` → `ingestEvent` controller → MongoDB → Socket.IO room broadcast |
| Dashboard login | Browser → `POST /api/auth/login` → JWT issued → `RealtimeProvider` opens socket → `subscribe` event → server joins socket to room |
| Live update | Event ingested → controller emits `"new-event"` to room → all subscribed dashboards receive it within 100ms |
| Incident alert | ERROR/CRITICAL event ingested → `findOneAndUpdate` on `Incident` collection → `"incident-updated"` emitted to room |

---

## 3. Tech Stack

### Backend

| Category | Technology | Notes |
|---|---|---|
| Runtime | **Node.js** | — |
| Framework | **Express 5.x** (`^5.2.1`) | Async errors in route handlers auto-forward to error middleware |
| Database | **MongoDB + Mongoose** | 6 collections, compound indexes, TTL indexes |
| Real-time | **Socket.IO v4** | WebSocket server bound to the same HTTP port |
| Auth (users) | **JWT** (access + refresh) | Token versioning for global logout |
| Auth (services) | **SHA-256 hashed API keys** | `crypto.createHash('sha256')`, stored as `ingestKeyHash` |
| OTP | **bcrypt + crypto.randomBytes** | CSPRNG, rejection sampling for unbiased digits |
| Email | **SendGrid** (`@sendgrid/mail`) | Transactional OTP emails only |
| Validation | **Joi** | Per-route request schemas |
| Security headers | **Helmet** | CSP, X-Frame-Options, etc. |
| Rate limiting | **express-rate-limit** (OTP) + custom in-memory Map (per-project) | |
| Environment config | `backend/example.env.js` | Centralised config with production required-var enforcement |

### Frontend

| Category | Technology | Notes |
|---|---|---|
| Build tool | **Vite 7** | `VITE_Backend_URL` env var for both HTTP and WebSocket |
| UI framework | **React 19** | Context API for global state |
| Styling | **TailwindCSS v4** | Utility-first; no component library |
| Animation | **Framer Motion + GSAP** | Landing page animations |
| HTTP client | **axios** | Singleton with request interceptor for Bearer token |
| WebSocket | **socket.io-client v4.8.3** | `transports: ['websocket']` (polling disabled) |
| Virtualisation | **react-window** | `ActivityFeed` renders only visible event rows |
| Routing | **React Router v6** | `ProtectedRoute` guards authenticated pages |

---

## 4. Key Engineering Decisions

### Why SHA-256 for API keys but bcrypt for passwords?

API keys are 32 random bytes (256 bits of entropy) — uniform randomness makes timing attacks on the hash comparison impractical. SHA-256 is fast and appropriate here.

Passwords have low entropy and known guessing patterns. bcrypt's deliberate slowness (configurable cost factor) and per-hash random salt make offline dictionary attacks prohibitively expensive. The algorithm mismatch is intentional.

### Why token versioning (`tokenVersion`) for logout?

JWTs are stateless — once issued, the server cannot revoke them. The solution: each `User` document carries a `tokenVersion` integer. Every JWT embeds `tv: user.tokenVersion`. The `POST /api/auth/logout-everywhere` endpoint increments `tokenVersion`. The `/api/auth/token/refresh` endpoint checks that `payload.tv === user.tokenVersion` before issuing new tokens. All old refresh tokens become invalid immediately.

**Limitation:** Access tokens already in flight remain valid until their natural 15-minute expiry. This is a deliberate trade-off for stateless performance.

### Why Socket.IO rooms instead of broadcasting + client-side filtering?

Broadcasting widely and filtering in the browser puts other tenants' data on the wire — a data leak regardless of whether the UI renders it. With named rooms (`project:<projectId>`), authorization is enforced once at join time via an ownership check (`project.ownerId === socket.userId`). Every subsequent emit is implicitly scoped, and the browser physically never receives data it is not entitled to.

### Why cursor-based pagination for events?

Offset-based pagination (`SKIP + LIMIT`) degrades with large datasets — MongoDB must scan all skipped documents. APILogs uses `?before=<eventTimestamp>` as a cursor: `Event.find({ projectId, eventTimestamp: { $lt: before } }).sort({ eventTimestamp: -1 }).limit(50)`. The compound index `{ projectId: 1, eventTimestamp: -1 }` makes this O(log n).

### Why event batching at 100ms intervals in `RealtimeContext`?

During high-throughput ingestion, a naive approach would call `setState` on every WebSocket message — potentially hundreds of React re-renders per second. Instead, incoming events are pushed into a `useRef` queue (no re-render). A `setInterval` flushes the queue into state every 100ms, coalescing bursty traffic into ~10 renders/second regardless of ingest rate. The event array is capped at 50 items to bound DOM memory.

### Why bcrypt for OTPs?

A database dump would expose plain OTPs, allowing direct account takeover. The OTP is generated with `crypto.randomBytes` (CSPRNG), immediately bcrypt-hashed with a configurable cost factor, and only the hash is persisted. Verification is `bcrypt.compare`. Even a full DB leak cannot be used to authenticate.

**Why rejection sampling in OTP generation?** `byte % 10` on a 0–255 byte biases digits 0–5 (which appear 26/256 times) vs 6–9 (25/256). The code discards bytes ≥ 250 so the accepted range is exactly 0–249 (25 × 10), making every digit equally probable.

---

## 5. Repository Structure

```
Apilogs/
├── backend/
│   ├── server.js                   # Entry point: DB connect → HTTP server → Socket.IO init
│   ├── app.js                      # Express factory: middleware + route mounting
│   ├── example.env.js              # Centralised config module (dotenv + validation)
│   └── src/
│       ├── controllers/
│       │   ├── auth.Controller.js  # Register, OTP verify, login, logout, token refresh
│       │   ├── project.Controller.js # Create project, list projects, rotate API key
│       │   ├── event.controller.js # Ingest event, get paginated events
│       │   ├── incident.controller.js # List incidents, update status
│       │   └── system.controller.js  # Health check, in-memory metrics
│       ├── middleware/
│       │   ├── auth.js             # authRequired — JWT Bearer verification
│       │   ├── apiKeyAuth.js       # API key SHA-256 verification
│       │   ├── ratelimiter.js      # otpLimiter (5/10min), globalLimiter (disabled)
│       │   ├── projectRateLimiter.js # In-memory 300 req/min per projectId
│       │   └── performanceTimer.js  # Logs slow requests >500ms
│       ├── models/
│       │   ├── User.js             # email, password (bcrypt), isVerified, tokenVersion
│       │   ├── Project.js          # projectName, ownerId, ingestKeyHash (select:false)
│       │   ├── Event.js            # service, severity, message, metadata, eventTimestamp
│       │   ├── Incident.js         # messageSignature, status, eventCount, firstOccurredAt
│       │   ├── OtpToken.js         # codeHash, expiresAt (TTL index), attempts, consumed
│       │   └── AuditLog.js         # purpose, projectId, userId, message
│       ├── routes/ (5 modules)
│       ├── realtime/
│       │   ├── socket.server.js    # Server singleton + getIO() factory
│       │   ├── socket.auth.js      # verifySocketToken(token)
│       │   └── socket.manager.js   # subscribe, unsubscribe, disconnect handlers + emitEventToProject()
│       ├── utils/
│       │   ├── genrateOtp.js       # makeOtp() → { plain, hash, expiresAt }
│       │   ├── sendEmail.js        # sendOtpEmail() via SendGrid
│       │   └── metrics.js          # MetricsStore singleton (in-memory counters)
│       └── validations/ (Joi schemas)
├── frontend/
│   └── src/
│       ├── api/                    # axios HTTP adapters (auth, project, event, incident)
│       ├── context/                # AuthContext, ProjectContext, EventContext, RealtimeContext
│       ├── hooks/                  # useRealtime (subscription + pagination lifecycle)
│       ├── services/               # socket.js — createSocketConnection(token) factory
│       ├── components/             # ActivityFeed, IncidentList, IncidentSummary
│       ├── pages/                  # Landing, Dashboard, auth/*, project/*, Event/*
│       └── routes/                 # AppRoutes.jsx, ProtectedRoute.jsx
└── instruction.md                  # Internal documentation requirements
```

---

## 6. Backend — Complete API Reference

### Auth Routes — `/api/auth`

| Method | Path | Auth | Rate Limit | Request Body | Success |
|---|---|---|---|---|---|
| POST | `/register` | None | None | `{ username, email, password }` | `201 { message }` |
| POST | `/verify/resend` | None | otpLimiter 5/10min | `{ email }` | `200 { message }` |
| POST | `/verify/confirm` | None | otpLimiter 5/10min | `{ email, otp }` | `200 { message, accessToken, refreshToken }` |
| POST | `/login` | None | None | `{ email, password }` | `200 { accessToken, refreshToken }` |
| POST | `/login/otp/request` | None | otpLimiter 5/10min | `{ email }` | `200 { message }` |
| POST | `/login/otp/verify` | None | otpLimiter 5/10min | `{ email, otp }` | `200 { accessToken, refreshToken }` |
| POST | `/token/refresh` | None | None | `{ token }` or `Authorization: Bearer <refreshToken>` | `200 { accessToken, refreshToken }` |
| POST | `/logout-everywhere` | JWT Bearer | None | — | `200 { message }` |

**Validation (Joi):** `register` — username 3–100 alphanumeric; password 8–500 chars with uppercase, lowercase, digit, special char. `login` — email format, password non-empty.

### Project Routes — `/api/project`

| Method | Path | Auth | Request Body / Params | Success |
|---|---|---|---|---|
| POST | `/create` | JWT | `{ projectName, description? }` | `200 { message, project, ingestKey, note }` |
| GET | `/list` | JWT | — | `200 { Projects: [...] }` |
| POST | `/:projectId/rotate-key` | JWT | `projectId` param | `200 { message, ingestKey, note }` |

⚠️ The `ingestKey` is returned **only once** (on create or rotate). It is not stored in plaintext — only its SHA-256 hash is persisted.

### Event Routes — `/api/events`

| Method | Path | Auth | Headers | Request Body | Success |
|---|---|---|---|---|---|
| POST | `/ingest/:projectId` | API Key | `x-api-key: <key>` | `{ service, severity, message, environment?, metadata?, eventTimestamp? }` | `201 { message, eventId }` |
| GET | `/:projectId` | JWT | — | Query: `?limit=50&before=<ISO date>` | `200 { count, events: [...] }` |

**Severity enum:** `INFO` · `WARN` · `ERROR` · `CRITICAL`  
**Environment enum:** `development` · `staging` · `production`  
**Rate limit:** 300 requests/minute per `projectId` (in-memory sliding window)

### Incident Routes — `/api/incidents`

| Method | Path | Auth | Request Body | Success |
|---|---|---|---|---|
| GET | `/:projectId` | JWT | — | `200 { incidents: [...] }` |
| PATCH | `/:incidentId/status` | JWT | `{ status: "ACKNOWLEDGED" | "RESOLVED" }` | `200 { message, incident }` |

Status transitions: `OPEN` → `ACKNOWLEDGED` or `RESOLVED`. `OPEN` cannot be set via this endpoint.

### System Routes — `/api/system` (no auth)

| Method | Path | Response |
|---|---|---|
| GET | `/health` | `{ status, uptimeSeconds, memory, cpuLoad, platform, mongo, activeSocketConnections, timestamp }` |
| GET | `/metrics` | `{ totalEventsIngested, eventsLastMinute, failedApiKeyAttempts, rateLimitHits, activeSocketConnections }` |

> ⚠️ Both system endpoints are unauthenticated and expose internal server details (memory, MongoDB host, connection count).

---

## 7. Backend — Startup & Request Lifecycle

### Server Startup Sequence

```
node server.js
  │
  ├─ require('./app')              → Express app configured (middleware + routes mounted)
  │    └─ require('./example.env.js') → dotenv.config() + required-var validation
  │
  ├─ connectdb()                  → mongoose.connect(MONGO_URI)
  │                                 Throws on failure — process exits
  ├─ http.createServer(app)       → HTTP server created (not yet listening)
  ├─ initializeSocketServer(server) → Socket.IO bound to same port
  │                                  io.use(auth middleware) registered
  │                                  io.on('connection') registered
  └─ server.listen(PORT)          → Server starts accepting requests
       └─ console.log('connected to Database ...', 'Server running on port ...', 'SocketIO initialized on same port')
```

**Key architectural details:**
- MongoDB connection is awaited **before** the HTTP server starts listening. A DB failure prevents requests from being served.
- Socket.IO shares the HTTP server's port — no separate WebSocket port needed.
- `getIO()` is the singleton accessor for the Socket.IO server, used by controllers to emit events without circular dependencies.

### Global Middleware Chain (every request)

```
Request → cors → helmet → express.json(limit:10kb) → performanceTimer → Route Handler
```

- **CORS:** origin restricted to `CLIENT_PRO_URL`. Allows credentials. Does not include `x-api-key` in `allowedHeaders` — API key ingestion from a browser may fail CORS preflight.
- **Helmet:** security headers (CSP, X-Frame-Options, HSTS, etc.)
- **express.json(10kb):** bodies larger than 10kb are rejected with 413
- **performanceTimer:** logs `[SLOW REQUEST]` if any route exceeds 500ms (does not reject)
- **globalLimiter:** ⚠️ commented out — IP-level global rate limiting is not active

---

## 8. Database Models & Schema Design

### `User`
```
{ username, email (unique, indexed, lowercase), password (bcrypt, select:false),
  isVerified (default: false), tokenVersion (default: 0, int), createdAt }
Indexes: email (unique), createdAt DESC
```
`tokenVersion` is the global logout mechanism. Pre-save hook: bcrypt-hashes password on change.

### `Project`
```
{ projectName, ownerId (ref: User, indexed), description,
  ingestKeyHash (select: false, do not expose), createdAt }
Indexes: ownerId
Methods: generateIngestKey() → { plainKey, hash }, verifyIngestKey(key) → boolean
```
`ingestKeyHash` has `select: false` — never returned in normal queries. Must be explicitly selected with `.select('+ingestKeyHash')`.

### `Event`
```
{ projectId (ref: Project, indexed), service (indexed), severity (enum, indexed),
  message, metadata (Mixed), environment (enum, indexed), eventTimestamp (indexed), createdAt }
Compound index: { projectId: 1, eventTimestamp: -1 }  ← powers cursor pagination
```

### `Incident`
```
{ projectId (ref: Project), service, severity, messageSignature (lowercase+trimmed message),
  status (enum: OPEN|ACKNOWLEDGED|RESOLVED, default: OPEN),
  firstOccurredAt, lastOccurredAt, eventCount (default: 1) }
Compound index: { projectId: 1, messageSignature: 1, status: 1 }  ← powers dedup query
```

### `OtpToken`
```
{ user (ref: User, indexed), purpose (enum: verify|login, indexed),
  codeHash (minlength: 60 — bcrypt hash), expiresAt (indexed), attempts (default: 0), consumed (default: false) }
TTL index: { expiresAt: 1 }, expireAfterSeconds: 0  ← MongoDB auto-deletes expired tokens
```

### `AuditLog`
```
{ purpose (enum: KEY_ROTATED|API_KEY_FAILED|SOCKET_UNAUTHORIZED), projectId (optional),
  userId, message, createdAt }
```

---

## 9. Authentication & Security Architecture

### Two Auth Mechanisms

```
                    Dashboard Users                    External Services
                         │                                   │
             ┌───────────▼──────────────┐     ┌─────────────▼────────────┐
             │  JWT Bearer token        │     │  x-api-key header        │
             │  (Authorization header)   │     │  (plain key, 64-char hex)│
             └───────────┬──────────────┘     └─────────────┬────────────┘
                         │                                   │
             ┌───────────▼──────────────┐     ┌─────────────▼────────────┐
             │  authRequired middleware  │     │  apiKeyAuth middleware    │
             │  jwt.verify(token,secret)│     │  SHA-256(providedKey)    │
             │  Requires sub + tv       │     │  === project.ingestKeyHash│
             │  Sets req.user.id        │     │  Sets req.project        │
             └───────────┬──────────────┘     └─────────────┬────────────┘
                         │                                   │
                  Protected routes                     Ingest route only
```

### JWT Token Design

```js
payload = { sub: user.id, tv: user.tokenVersion }
accessToken  = jwt.sign(payload, JWT_ACCESS_SECRET,  { expiresIn: "15m" })
refreshToken = jwt.sign(payload, JWT_REFRESH_SECRET, { expiresIn: "7d" })
```

- **Access token:** short-lived (15m), used in `Authorization: Bearer` header
- **Refresh token:** long-lived (7d), used to obtain new access tokens
- **`tv` (token version):** invalidated by `logoutEverywhere` via `tokenVersion++`

### OTP Security Model

```
makeOtp() →
  1. randomDigits(6): crypto.randomBytes + rejection sampling (discard bytes ≥ 250)
  2. bcrypt.hash(plain, saltRounds)
  3. expiresAt = Date.now() + OTP_TTL_SECONDS * 1000

OtpToken stored: { codeHash, expiresAt, attempts: 0, consumed: false }
Plain OTP: sent via SendGrid, never persisted

Verification:
  1. Check token exists and not expired
  2. Check attempts < OTP_MAX_ATTEMPTS (default: 5)
  3. bcrypt.compare(submitted, codeHash)
  4. On success: consumed = true (single-use)
  5. On failure: attempts++ (rate limits brute force)
```

### API Key Security Model

```
Key generation:
  rawKey = crypto.randomBytes(32).toString('hex')  // 64-char hex = 256 bits entropy
  ingestKeyHash = SHA-256(rawKey)
  Store: project.ingestKeyHash (select: false)
  Return: rawKey (shown once only)

Verification:
  SHA-256(providedKey) === project.ingestKeyHash
```

SHA-256 is appropriate (not bcrypt) because the key has 256 bits of entropy — brute force is computationally impossible regardless of hash speed.

---

## 10. Real-Time Architecture (Socket.IO)

### Three-File Backend

| File | Responsibility |
|---|---|
| `socket.server.js` | Creates Socket.IO server singleton; registers auth middleware; exports `initializeSocketServer()` + `getIO()` |
| `socket.auth.js` | `verifySocketToken(token)` — JWT verification for WebSocket handshakes |
| `socket.manager.js` | Per-socket handlers: `subscribe`, `unsubscribe`, `disconnect`; exports `emitEventToProject()` |

### Two-Stage Security on the Socket Channel

```
Stage 1 — Handshake Authentication (io.use middleware):
  Client sends: io(URL, { auth: { token: accessToken } })
  Server reads: socket.handshake.auth.token
  Verifies JWT → sets socket.userId = decoded.sub
  On failure: next(new Error('Authentication failed')) → client gets connect_error

Stage 2 — Room Authorization (subscribe handler):
  Client emits: socket.emit('subscribe', { projectId })
  Server: Project.findById(projectId).select('ownerId')
  Checks: project.ownerId.toString() === socket.userId.toString()
  On success: socket.join('project:<projectId>')
  On failure: emit('subscription-error', 'Unauthorized access to this project')
```

Authentication proves *who* you are. Authorization proves *which rooms* you may enter. Both are required.

### Room Model

All rooms follow the pattern `project:<projectId>` (24-char hex MongoDB ObjectId).

| Broadcast | Room | Emitted by | Condition |
|---|---|---|---|
| `"new-event"` | `project:<id>` | `emitEventToProject()` in `socket.manager.js` | Every event ingested |
| `"incident-updated"` | `project:<id>` | `event.controller.ingestEvent` | Severity is ERROR or CRITICAL |
| `"incident-updated"` | `project:<id>` | `incident.controller.updateIncidentStatus` | User changes incident status |

### Client-Side Socket Lifecycle

```
Login → AuthContext sets accessToken
      → AppWithRealtime passes token prop to RealtimeProvider
      → RealtimeProvider effect: createSocketConnection(token)
        Socket.IO Config: {
          transports: ['websocket'],   // No polling fallback
          auth: { token },             // JWT in handshake, not query string
          reconnection: true,
          reconnectionAttempts: 5,
          reconnectionDelay: 2000
        }
      → Registers 'new-event' handler: push to eventQueueRef
      → Registers 'incident-updated' handler: update incidents state
      → Starts 100ms flush interval

Logout → accessToken = null → RealtimeProvider disconnects socket → clears all state
```

### Event Batching (100ms queue)

```
WebSocket 'new-event' arrives →
  eventQueueRef.current.push(eventData)   [no re-render]

Every 100ms:
  if (queue.length > 0):
    batch = queue; queue = []
    setEvents(prev => {
      combined = [...prev, ...batch]
      return combined.length > 50 ? combined.slice(-50) : combined
    })
```

**Why:** During high-throughput ingestion, this reduces React re-renders from N/second to ≤10/second. The 50-event cap bounds DOM memory.

---

## 11. Frontend Architecture

### Context Layer (Global State)

```
BrowserRouter
  └─ AuthProvider         → user, accessToken, isAuthenticated, login, logout, register
       └─ ProjectProvider  → projects[], fetchProjects, createProject
            └─ EventProvider → ingestEvent, loading, error, success
                 └─ AppWithRealtime (reads accessToken, passes to RealtimeProvider)
                      └─ RealtimeProvider → events[], incidents[], subscribe/unsubscribe actions
                           └─ AppRoutes
```

| Context | Hook | Purpose |
|---|---|---|
| `AuthContext` | `useAuth()` | Session state + login/logout/register actions |
| `ProjectContext` | `useProject()` | Project list + create action |
| `EventContext` | `useEvent()` | Event ingestion action + transient loading/error/success |
| `RealtimeContext` | `useRealtimeContext()` | WebSocket state + all socket actions |

### Axios Request Interceptor (Auth Wiring)

`api/event.api.js` and `api/Project.api.js` both register a request interceptor on the shared axios singleton:

```js
api.interceptors.request.use((req) => {
  const token = localStorage.getItem('token');
  if (token) req.headers.Authorization = `Bearer ${token}`;
  return req;
});
```

Because both files mutate the same singleton, importing either one arms the `Authorization` header for **all requests in the app** — including `incident.api.js`, which registers no interceptor of its own. This is a side-effect of module import order, not an explicit design.

### Route Guard

```jsx
// ProtectedRoute.jsx
const { isAuthenticated, loading } = useAuth();
if (loading) return <p>Loading...</p>;
if (!isAuthenticated) return <Navigate to="/login" replace />;
return children;
```

`isAuthenticated = !!accessToken` — checks token string existence only, not validity or expiry. The real security gate is `authRequired` on the backend.

### `useRealtime` Hook — Subscription Lifecycle

```js
useRealtime(projectId) →
  1. clearEvents()
  2. GET /api/events/:projectId?limit=50 → initializeEvents(reversed)
  3. subscribeToProject(projectId)   // emit 'subscribe' to socket
  4. Returns { loadOlderEvents }     // cursor-based backward pagination

Cleanup:
  4. unsubscribeFromProject(projectId)  // emit 'unsubscribe'
  5. clearEvents()
```

### Key Components

| Component | Data Source | Responsibility |
|---|---|---|
| `ActivityFeed` | `useRealtimeContext().events` | Virtualised list (react-window) of live events, 70vh, 110px rows |
| `IncidentList` | Props (`incidents`) | Incident cards with Acknowledge/Resolve actions |
| `IncidentSummary` | Props (`incidents`) | 3-card stat panel: Open count, Critical count, Error count |

---

## 12. Event Ingestion Flow (End-to-End)

```mermaid
sequenceDiagram
    participant SDK as External Service
    participant MW1 as apiKeyAuth
    participant MW2 as projectRateLimiter
    participant MW3 as validateEvent
    participant Ctrl as ingestEvent Controller
    participant DB as MongoDB
    participant IO as Socket.IO

    SDK->>MW1: POST /api/events/ingest/:projectId (x-api-key: <key>)
    MW1->>DB: Project.findById(projectId).select("+ingestKeyHash")
    MW1->>MW1: SHA-256(providedKey) === ingestKeyHash?
    MW1-->>MW2: req.project = project
    MW2->>MW2: in-memory Map: count < 300/min?
    MW3->>MW3: Joi validate params + body
    Ctrl->>DB: Event.create({ projectId, service, severity, message, ... })
    Ctrl->>Ctrl: metrics.incrementEvent()
    alt severity is ERROR or CRITICAL
        Ctrl->>DB: Incident.findOneAndUpdate({ projectId, messageSignature, status:{$in:["OPEN","ACKNOWLEDGED"]} }, { $inc:{eventCount:1}, $set:{lastOccurredAt} }, { new:true })
        DB-->>Ctrl: existing incident OR null
        opt no existing incident
            Ctrl->>DB: Incident.create({ status:"OPEN", eventCount:1, firstOccurredAt })
        end
        Ctrl->>IO: io.to("project:<id>").emit("incident-updated", incident)
    end
    Ctrl->>IO: emitEventToProject(io, projectId, event) → emit("new-event")
    Ctrl-->>SDK: 201 { message, eventId }
```

---

## 13. Incident Grouping Algorithm

When an event with severity `ERROR` or `CRITICAL` arrives:

```js
const messageSignature = message.trim().toLowerCase();

const incident = await Incident.findOneAndUpdate(
  {
    projectId,
    messageSignature,          // normalised message — the grouping key
    status: { $in: ['OPEN', 'ACKNOWLEDGED'] }  // ignore RESOLVED incidents
  },
  {
    $set: { lastOccurredAt: eventTimestamp },   // update "when"
    $inc: { eventCount: 1 }                     // tally occurrences
  },
  { new: true }                                 // return updated doc
);

if (!incident) {
  // First occurrence — create a new incident
  await Incident.create({
    projectId, service, severity, messageSignature,
    status: 'OPEN', firstOccurredAt: eventTimestamp,
    lastOccurredAt: eventTimestamp, eventCount: 1
  });
}
```

**Why `messageSignature`?** Normalising to lowercase + trimmed removes noise (capitalisation differences, trailing spaces) while preserving the semantic uniqueness of different error messages. Two identical errors spelled differently still group together.

**Why `$in: ['OPEN', 'ACKNOWLEDGED']`?** A `RESOLVED` incident should not be reopened by a recurrence — it should become a new incident, tracking the regression independently.

The compound index `{ projectId: 1, messageSignature: 1, status: 1 }` makes this upsert query an indexed lookup.

---

## 14. Environment Variables Reference

All backend configuration flows through `backend/example.env.js`. Required vars throw at startup when `NODE_ENV=production`.

| Variable | Required in Prod | Default | Purpose |
|---|---|---|---|
| `MONGO_URI` | ✅ | — | MongoDB connection string |
| `PORT` | — | `5000` | HTTP server port |
| `NODE_ENV` | — | `development` | Controls production guards |
| `JWT_ACCESS_SECRET` | ✅ | — | Sign/verify access tokens |
| `JWT_REFRESH_SECRET` | ✅ | — | Sign/verify refresh tokens |
| `JWT_ACCESS_EXPIRES_IN` | — | `15m` | Access token lifetime |
| `JWT_REFRESH_EXPIRES_IN` | — | `7d` | Refresh token lifetime |
| `SENDGRID_API_KEY` | ✅ | — | Transactional email (OTP) |
| `SMTP_FROM` | ✅ | — | Sender email address |
| `CLIENT_PRO_URL` | ✅ | — | CORS allowed origin (Express) |
| `CLIENT_URL` | ✅ | — | Imported by socket.server.js but unused |
| `OTP_LENGTH` | ✅ | `6` | Number of OTP digits |
| `OTP_TTL_SECONDS` | ✅ | `600` | OTP validity window (seconds) |
| `OTP_MAX_ATTEMPTS` | ✅ | `5` | Max wrong OTP guesses per token |
| `SaltValue` | ✅ | `10` | bcrypt cost factor |

**Frontend:**

| Variable | Purpose |
|---|---|
| `VITE_Backend_URL` | Backend URL — used for both axios baseURL (`${URL}/api`) and Socket.IO (`io(URL)`) |

---

## 15. Known Bugs & Implementation Gaps

These are all verified against source code. They are documented here to demonstrate engineering awareness, not to imply the system is broken for its primary use cases.

### Critical / Production-Breaking

| # | Location | Bug | Impact |
|---|---|---|---|
| B1 | `event.controller.js` | `incident` declared with `const` inside the `if` block but reassigned with `= await Incident.create(...)` in the `if (!incident)` block → `TypeError: Assignment to constant variable` | New incidents (first occurrence of any ERROR/CRITICAL message) always throw 500. `eventCount` increments work; first event per signature fails. |
| B2 | `project.Routes.js` | `ValidateProject` (Joi body validator) applied to `GET /api/project/list`. GET requests have no body. The Joi schema has all fields `optional`, so validation passes, but it is semantically wrong and could break if schema changes. | `GET /api/project/list` currently works but is fragile |
| B3 | `socket.manager.js` | `AuditLog` is called (`AuditLog.create(...)`) but never imported. Variable is undefined → `ReferenceError` caught silently | Unauthorized socket subscription attempts are never audited |
| B4 | `projectRateLimiter.js` | `metrics` imported as `mertics` (typo). `metrics.incrementRateLimitHit()` is in the catch block — `ReferenceError` | `rateLimitHits` metric counter never increments |

### Medium — Functional Gaps

| # | Location | Bug | Impact |
|---|---|---|---|
| B5 | `socket.server.js` | Socket.IO CORS set to `origin: "*"` despite `CLIENT_URL` being imported. Express CORS is restricted to `CLIENT_PRO_URL` | Any origin can establish WebSocket connections |
| B6 | `socket.auth.js` | Reads `process.env.JWT_ACCESS_SECRET` directly at module load, bypassing the config module | Works today (dotenv loaded earlier in require chain); breaks in isolation (tests) |
| B7 | `apiKeyAuth.js` | SHA-256 comparison uses `===` (not constant-time). Not bcrypt, so timing side-channel is theoretically possible, though with 256-bit entropy keys the practical risk is minimal | Low practical risk |
| B8 | `frontend/EventContext.jsx` | `ingestEvent` sends a Bearer JWT to `POST /api/events/ingest/:projectId` which requires `x-api-key`. The Event.ingest page always returns 401 | Manual event ingest from the browser dashboard fails |
| B9 | `RealtimeContext + useRealtime` | No room re-join after socket reconnect. `subscribe` is emitted once; reconnection restores the connection but joins no room | After any network blip, the live feed silently stops forever until page reload |
| B10 | `AuthContext` | `refreshToken()` is never called. On access token expiry, all authenticated API calls return 401 silently | Users are effectively logged out after 15 minutes without feedback |

### Low — Code Quality & Security Hardening

| # | Location | Issue |
|---|---|---|
| B11 | `socket.server.js` | Logs the raw JWT and decoded payload on every handshake (`console.log("socket auth token:", token)`) |
| B12 | `App.jsx` | `console.log("AppWithRealtime accessToken:", accessToken)` logs the JWT to the browser console on every render |
| B13 | `server.js` | Unused `net` import |
| B14 | `/api/system/health` + `/api/system/metrics` | Unauthenticated — leaks MongoDB host, memory usage, CPU load, connection counts |
| B15 | `socket.manager.js` | No per-socket rate limiting on `subscribe`. Each emit costs a `Project.findById`. |
| B16 | `auth.Controller.js` | No `try/catch` in any handler — Express 5 auto-forwards rejections; backend has no custom error handler, so all unhandled errors return HTML 500 to a JSON API |
| B17 | `sendEmail.js` | PII (recipient email) written to server logs on every send failure |
| B18 | `services/socket.js` | `subscription-error` and `reconnect_failed` events have no client listener; green "Live updates enabled" banner is shown via 600ms setTimeout, not on server ack — false positive when subscription is rejected |

---

## 16. Interview Preparation — Deep Technical Q&A

**Q: Walk me through the complete flow when an external service ingests a log event.**

> The service POSTs to `/api/events/ingest/:projectId` with an `x-api-key` header. `apiKeyAuth` middleware reads the header, fetches the project from MongoDB with `.select('+ingestKeyHash')` (the hash field is `select:false` by default), SHA-256 hashes the provided key, and compares it to the stored hash. On success it sets `req.project`. The `projectRateLimiter` middleware then checks an in-memory `Map` keyed by `projectId` — 300 requests/minute sliding window. Joi validates the body. The `ingestEvent` controller writes an `Event` document, increments the in-memory `MetricsStore` counters, then for ERROR/CRITICAL events does a `findOneAndUpdate` on `Incident` using `messageSignature` (normalised message) as the grouping key — atomic increment of `eventCount` or create-on-not-found. Finally it broadcasts `"new-event"` and optionally `"incident-updated"` to the Socket.IO room `project:<projectId>`. Connected dashboard clients receive it within 100ms via the event queue flush.

**Q: How does the JWT-based global logout work?**

> Each `User` document has a `tokenVersion` integer. Every issued JWT embeds `{ sub: userId, tv: tokenVersion }`. `POST /api/auth/logout-everywhere` increments `tokenVersion`. The `/token/refresh` endpoint verifies that `payload.tv === user.tokenVersion` before issuing new tokens — if a token was issued before the increment, it fails this check. Access tokens already in flight remain valid for up to 15 minutes, which is an explicit trade-off for stateless performance. Socket connections are also unaffected — the socket middleware only checks the token at connection time.

**Q: How do you ensure one user can't subscribe to another user's project live feed?**

> Two-stage security. First, the Socket.IO handshake `io.use()` middleware verifies the JWT and sets `socket.userId`. Then the `"subscribe"` handler loads the project with `Project.findById(projectId).select('ownerId')` and compares `project.ownerId.toString() === socket.userId.toString()`. If they don't match, a `"subscription-error"` is emitted and the socket is not joined to the room. Authentication proves identity; authorization proves resource access. Without both, any authenticated user could subscribe to any project ID.

**Q: Why does your OTP use bcrypt instead of a simpler hash like SHA-256?**

> SHA-256 is fast by design — millions of hashes per second. A stolen `OtpToken` collection could be brute-forced rapidly. bcrypt is slow by design — the cost factor makes each comparison take ~100ms, making exhaustive search of 6-digit codes expensive even for an attacker with the hash. Beyond hashing, the code uses `crypto.randomBytes` (CSPRNG, not `Math.random`) and applies rejection sampling — bytes ≥ 250 are discarded before `byte % 10`, so all ten digits are equally probable.

**Q: Explain the 100ms event batching in `RealtimeContext`.**

> `RealtimeContext` manages the Socket.IO connection. When a `"new-event"` message arrives, our handler pushes it to a `useRef` array — a ref, not state, so no React re-render occurs. A `setInterval` runs every 100ms; if the queue is non-empty, it drains the queue atomically and calls `setEvents` once with the full batch. During high-throughput ingestion this reduces renders from N/second to ≤10/second. The state array is also capped at 50 events — `slice(combined.length - 50)` discards the oldest events when over the limit.

**Q: How does cursor-based pagination work for events?**

> `GET /api/events/:projectId?limit=50&before=<ISO timestamp>`. The backend builds `{ projectId, eventTimestamp: { $lt: new Date(before) } }` if `before` is provided, sorts by `eventTimestamp DESC`, and caps at `Math.min(limit, 200)`. The compound index `{ projectId: 1, eventTimestamp: -1 }` makes this an indexed range scan — not a full scan. On the client, `useRealtime`'s `loadOlderEvents()` reads `events[events.length - 1].eventTimestamp` as the cursor, fetches the next 50 older events, reverses them (API returns newest-first), deduplicates by `_id`, and calls `prependEvents()` in context.

**Q: What's the most impactful bug in the codebase and how would you fix it?**

> The most impactful is in `event.controller.js`: `incident` is declared `const` inside the severity check block but then reassigned with `const incident = Incident.create(...)` — wait, more precisely the code does `let incident = await Incident.findOneAndUpdate(...)` then in the `if (!incident)` block tries `incident = await Incident.create(...)`. But the outer declaration uses `const`, causing a `TypeError` on every new incident creation. Every first occurrence of an ERROR or CRITICAL message returns 500 to the SDK. The fix is trivial: change `const incident` to `let incident`. The second most impactful is the socket feed silently dying after a network blip because Socket.IO rooms are server-side state and lost on disconnect — the fix is adding a `socket.on('connect', () => { if (currentProjectId) subscribeToProject(currentProjectId); })` listener in `useRealtime`.

---

*Documentation built from direct source code inspection of all 83 markdown files across `backend/` and `frontend/`. All facts verified against actual `.js` source files. Known discrepancies between documentation and implementation are explicitly noted.*
