# APILogs Backend — Middleware Complete Technical Reference

> **Audience:** Developer preparing for technical interviews.
> **Source of truth:** Actual `.js` source files in `backend/src/middleware/`, `backend/src/realtime/`, and `backend/src/validations/`.
> **Last updated:** 2026-09-15

---

## Table of Contents

1. [Middleware Architecture Overview](#1-middleware-architecture-overview)
2. [Middleware Execution Map](#2-middleware-execution-map)
3. [auth.js — JWT Authentication](#3-authjs--jwt-authentication)
4. [apiKeyAuth.js — API Key Authentication](#4-apikeyauthjs--api-key-authentication)
5. [projectRateLimiter.js — Per-Project Rate Limiting](#5-projectratelimiterjs--per-project-rate-limiting)
6. [ratelimiter.js — IP-Based Rate Limiting](#6-ratelimiterjs--ip-based-rate-limiting)
7. [performanceTimer.js — Request Timing](#7-performancetimerjs--request-timing)
8. [Validation Middleware](#8-validation-middleware)
9. [Socket.IO Authentication Middleware](#9-socketio-authentication-middleware)
10. [Error Handling Conventions](#10-error-handling-conventions)
11. [Security Implementation Notes](#11-security-implementation-notes)
12. [Documentation Discrepancies Found](#12-documentation-discrepancies-found)
13. [Interview Preparation Notes](#13-interview-preparation-notes)

---

## 1. Middleware Architecture Overview

The APILogs backend uses two distinct request pipelines, each with its own middleware stack:

### HTTP Pipeline (Express)

```
Incoming HTTP Request
        │
        ▼
[express.json({ limit:"10kb" })]   ← body parsing, size cap
[helmet()]                          ← security headers
[cors()]                            ← origin control
[performanceTimer]                  ← request timing (all routes)
        │
        ├─── /api/auth/*
        │         [otpLimiter]      ← IP rate limit (OTP routes only)
        │         [validateRegister / validateLogin]  ← Joi validation
        │         [authRequired]    ← JWT check (logout only)
        │
        ├─── /api/events/ingest/:projectId
        │         [apiKeyAuth]      ← API key → sets req.project
        │         [projectRateLimiter] ← in-memory per-project limit
        │         [validateEvent]   ← Joi validation
        │
        ├─── /api/events/:projectId (GET)
        │         [authRequired]    ← JWT check → sets req.user
        │
        ├─── /api/incidents/*
        │         [authRequired]    ← JWT check → sets req.user
        │
        ├─── /api/project/*
        │         [authRequired]    ← JWT check → sets req.user
        │         [ValidateProject] ← Joi validation (create + list)
        │
        └─── /api/system/*         ← No middleware — public
```

### WebSocket Pipeline (Socket.IO)

```
New Socket.IO Connection
        │
        ▼
[Socket middleware: verifySocketToken]  ← JWT check → sets socket.userId
        │
        ▼
registerSocketHandlers(io, socket)
        ├─── "subscribe" event    ← project ownership check
        ├─── "unsubscribe" event  ← leaves room
        └─── "disconnect" event   ← metrics decrement
```

---

## 2. Middleware Execution Map

| Route | Method | Middleware Chain (in order) |
|---|---|---|
| `/api/auth/register` | POST | `validateRegister` |
| `/api/auth/verify/resend` | POST | `otpLimiter` |
| `/api/auth/verify/confirm` | POST | `otpLimiter` |
| `/api/auth/login` | POST | `validateLogin` |
| `/api/auth/login/otp/request` | POST | `otpLimiter` |
| `/api/auth/login/otp/verify` | POST | `otpLimiter` |
| `/api/auth/token/refresh` | POST | _(none)_ |
| `/api/auth/logout-everywhere` | POST | `authRequired` |
| `/api/events/ingest/:projectId` | POST | `apiKeyAuth` → `projectRateLimiter` → `validateEvent` |
| `/api/events/:projectId` | GET | `authRequired` |
| `/api/incidents/:projectId` | GET | `authRequired` |
| `/api/incidents/:incidentId/status` | PATCH | `authRequired` |
| `/api/project/create` | POST | `authRequired` → `ValidateProject` |
| `/api/project/list` | GET | `authRequired` → `ValidateProject` |
| `/api/project/:projectId/rotate-key` | POST | `authRequired` |
| `/api/system/health` | GET | _(none)_ |
| `/api/system/metrics` | GET | _(none)_ |
| Socket.IO connections | — | `verifySocketToken` |

---

## 3. `auth.js` — JWT Authentication

**File:** `src/middleware/auth.js`
**Exports:** `{ authRequired }`
**Dependencies:** `jsonwebtoken`, `../../example.env`

### Purpose

Protects HTTP routes that require a logged-in user. Verifies the JWT access token and attaches the decoded user identity to `req.user` for use in controllers.

### Implementation: `authRequired(req, res, next)`

```
Authorization: Bearer <accessToken>
```

**Step-by-step execution:**

| Step | Code Action | Failure |
|---|---|---|
| 1 | Read `req.headers?.authorization` | — |
| 2 | Check it starts with `"Bearer "` | `401 { error: "Authorization header must be provided in the format: Bearer <token>" }` |
| 3 | Slice off `"Bearer "`, trim whitespace | — |
| 4 | Check token string is not empty | `401 { error: "Missing token" }` |
| 5 | Check `env.JWT_ACCESS_SECRET` is defined | `500 { error: "Server misconfiguration: JWT secret missing." }` |
| 6 | `jwt.verify(token, env.JWT_ACCESS_SECRET)` | `401 { error: "Invalid or expired token" }` |
| 7 | Check `payload.sub` and `payload.tv` both exist | `401 { error: "Malformed token payload" }` |
| 8 | Set `req.user = { id: payload.sub, tokenVersion: payload.tv }` | — |
| 9 | Call `next()` | — |

**What it sets on the request:**
```js
req.user = {
    id: "64abc...",         // MongoDB ObjectId (string) of the user
    tokenVersion: 3         // Token version from JWT payload
}
```

### What `authRequired` Does NOT Do

- ❌ Does NOT query the database to verify the user still exists.
- ❌ Does NOT check `tokenVersion` against the database — only uses the value from the JWT payload.
- ❌ Does NOT verify that `isVerified` is true for the user.

> **Security implication:** A deleted user's access token continues to pass `authRequired` until it expires. `logoutEverywhere` (which bumps `tokenVersion` in DB) only fully takes effect when access tokens expire naturally, because `authRequired` never compares `tokenVersion` to the DB value.

### JWT Token Structure

Tokens are created by `signToken()` in `auth.Controller.js`:

```js
payload = {
    sub: user._id,         // user MongoDB ID
    tv:  user.tokenVersion // version counter
}
```

- **Access token** signed with `JWT_ACCESS_SECRET`, expires in `JWT_ACCESS_EXPIRES_IN` (e.g., `"15m"`).
- **Refresh token** signed with `JWT_REFRESH_SECRET`, expires in `JWT_REFRESH_EXPIRES_IN` (e.g., `"7d"`).
- Both are standard JWT (HS256 by default in jsonwebtoken).

### Token Versioning Mechanism

`tokenVersion` starts at `0` on every new user. When `logoutEverywhere` is called:

```js
user.tokenVersion += 1
user.save()
```

New tokens minted after this have `tv = 1`. When `refreshToken` is called:

```js
if (user.tokenVersion !== payload.tv) throw new Error("Invalid token")
```

This rejects old refresh tokens. But `authRequired` only reads `tv` from the JWT payload — it does not compare to the DB. So old **access tokens** remain valid until their `expiresIn` window closes.

### Error Response Shape

All errors from `authRequired` are plain JSON:
```json
{ "error": "<message string>" }
```

### Interview Explanation

> "The `authRequired` middleware verifies a JWT Bearer token from the Authorization header using the access secret. It doesn't touch the database — it's fully stateless. We embed the user ID and a `tokenVersion` in the payload. When a user logs out everywhere, we increment `tokenVersion` in the DB. New refresh tokens carry the new version and old ones fail at the refresh endpoint. However, existing access tokens still pass `authRequired` until they expire — this is an accepted trade-off of stateless JWT authentication."

---

## 4. `apiKeyAuth.js` — API Key Authentication

**File:** `src/middleware/apiKeyAuth.js`
**Exports:** `{ apiKeyAuth }`
**Dependencies:** `mongoose`, `../models/Project`, `../models/AuditLog`, `../utils/metrics`

### Purpose

Authenticates requests from **external services** (microservices, servers, apps) that send log events. These callers are not human users — they don't have JWTs. Instead, each project has a secret **ingest API key** generated at project creation.

This middleware is used on exactly one route: `POST /api/events/ingest/:projectId`.

### Implementation: `apiKeyAuth(req, res, next)`

**Step-by-step execution:**

| Step | Code Action | Failure Response |
|---|---|---|
| 1 | `req.params.projectId` — extract project ID | — |
| 2 | `req.headers["x-api-key"]` — extract API key | `401 { message: "API key missing" }` |
| 3 | `mongoose.Types.ObjectId.isValid(projectId)` | `400 { message: "Invalid projectId format" }` |
| 4 | `Project.findById(projectId).select("+ingestKeyHash")` | `404 { message: "Project not found" }` |
| 5 | `project.verifyIngestKey(providedKey)` | see step 5a |
| 5a | If invalid: create AuditLog, return | `403 { message: "Invalid API key" }` |
| 6 | `req.project = project` | — |
| 7 | `next()` | — |
| Catch | `metrics.incrementApiKeyFailure()` | `500 { message: "API Key authentication failed", error: ... }` |

**What it sets on the request:**
```js
req.project = {
    _id: ObjectId("..."),
    projectName: "...",
    ownerId: ObjectId("..."),
    ingestKeyHash: "sha256_hex_here",  // explicitly selected hidden field
    description: "...",
    createdAt: Date,
    updatedAt: Date
}
```

### API Key Hashing Mechanism

The API key is **NOT** stored in the database in any recoverable form. Only its SHA-256 hash is stored:

```js
// Project model — generateIngestKey()
const rawKey = crypto.randomBytes(32).toString("hex"); // 64-char hex string
const hash = crypto.createHash("sha256").update(rawKey).digest("hex");
this.ingestKeyHash = hash;
return rawKey;                 // This is the only time plain key is accessible

// Project model — verifyIngestKey(providedKey)
const hash = crypto.createHash("sha256").update(providedKey).digest("hex");
return this.ingestKeyHash === hash;  // Constant-time? No — string comparison
```

> ⚠️ **Security note:** The hash comparison uses `===` (JavaScript string equality), which is NOT guaranteed to be constant-time. A timing attack could in theory distinguish correct vs incorrect keys. A production-grade implementation would use `crypto.timingSafeEqual()`.

### `ingestKeyHash` Field Visibility

The field is defined with `select: false` in the Mongoose schema:
```js
ingestKeyHash: { type: String, required: true, select: false }
```
This means it is **never returned** by any normal `Project.findById()` or `Project.find()` call. It must be explicitly requested:
```js
Project.findById(id).select("+ingestKeyHash")
```
This is done correctly in `apiKeyAuth` and `rotateIngestKey`, but the select is **missing** from `createProject` — however there the key has just been generated so the hash is already in memory.

### AuditLog Side Effect

When an invalid API key is attempted:
```js
AuditLog.create({
    purpose: "API_KEY_FAILED",
    projectId,
    ipAddress: req.ip,
    message: "Invalid API key attempt"
})
```
This is wrapped in its own try/catch — audit log failures don't affect the `403` response. The caller's IP is captured via `req.ip`.

### Error Handling

- A wrong API key returns `403` (not `401`) — intentional distinction between "no auth" and "bad auth".
- Unexpected server errors (not wrong-key scenarios) increment `metrics.failedApiKeyAttempts` and return `500`.

### Interview Explanation

> "`apiKeyAuth` handles machine-to-machine authentication for our event ingestion endpoint. External services send their project ID in the URL and their ingest key in the `x-api-key` header. We SHA-256 hash the provided key and compare it to the stored hash — the plain key is never in the database. If the key is wrong, we write an audit log with the caller's IP and return 403. The project document is attached to `req.project` so the controller doesn't need to re-fetch it."

---

## 5. `projectRateLimiter.js` — Per-Project Rate Limiting

**File:** `src/middleware/projectRateLimiter.js`
**Exports:** `{ projectRateLimiter }`
**Dependencies:** `../utils/metrics` (imported as `mertics` — typo in source)

### Purpose

Limits the number of event ingest requests per project to prevent any single project from flooding the system or the database.

### Configuration (Hard-coded, not from `.env`)

```js
const RATE_LIMIT = 300;      // max requests per window
const WINDOW_MS  = 60_000;   // 1 minute window
```

### Implementation: `projectRateLimiter(req, res, next)`

Uses an in-memory `Map<string, { count: number, windowStart: number }>` called `projectCounters`. This is a module-level variable — it persists across requests for the lifetime of the process.

**Step-by-step execution:**

| Step | Action | Failure |
|---|---|---|
| 1 | Resolve `projectId` from `req.params.projectId` or `req.project._id` | `400 { message: "Project ID missing for rate limiting" }` |
| 2 | `Date.now()` — get current timestamp | — |
| 3 | Look up or create entry in `projectCounters` Map | — |
| 4 | If `now - entry.windowStart >= WINDOW_MS`: reset `count = 0`, `windowStart = now` | — |
| 5 | `entry.count += 1` | — |
| 6 | If `entry.count > RATE_LIMIT`: write AuditLog, return 429 | `429 { message: "Rate limit exceeded for this project" }` |
| 7 | `next()` | — |

### Sliding vs Fixed Window

This is a **fixed window** implementation, not a true sliding window. The window starts when the first request comes in for a project. At the end of each 60-second window, the counter resets to zero. This means a burst of 300 requests at the end of one window + 300 at the start of the next = 600 in 60 seconds — a known limitation.

### Known Bugs

**Bug 1 — Import alias typo:**
```js
const mertics = require("../utils/metrics");  // typo: "mertics" not "metrics"
```
The outer catch block references `metrics.incrementRateLimitHit()` (correct spelling) — this will throw a `ReferenceError` when the catch block runs. The `429` response still fires correctly (it's in the inner block), but the metrics counter for rate limit hits is never incremented.

**Bug 2 — `AuditLog` not imported:**
```js
await AuditLog.create({ ... });  // AuditLog is never imported in this file
```
This throws a `ReferenceError` when rate limit is exceeded. The error is caught by the inner try/catch and logged to console. The `429` response still fires because `return res.status(429).json(...)` runs before the AuditLog line is reached... actually no — let's re-read. The `AuditLog.create()` is called **before** the `return res.status(429)`. So the flow is:
1. `await AuditLog.create(...)` → **throws ReferenceError**
2. Caught by inner catch → `console.error`
3. Falls through to `return res.status(429).json(...)` ✓

So the 429 response **does** still get sent correctly. Only the audit log is missing.

**Bug 3 — Not distributed:**
`projectCounters` is in-memory and process-local. In a multi-process or multi-server deployment (e.g., PM2 cluster mode), each process has its own counter. The effective rate limit would be `RATE_LIMIT * number_of_processes`.

### Interview Explanation

> "`projectRateLimiter` is a custom fixed-window rate limiter that tracks request counts per project in a `Map`. It hard-codes 300 requests per minute. It runs after `apiKeyAuth`, so the project is already verified. I'd note two bugs I found: `AuditLog` is referenced but not imported, so audit records are silently dropped on rate-limit hits. And it's in-memory, so it doesn't work correctly in clustered Node.js deployments."

---

## 6. `ratelimiter.js` — IP-Based Rate Limiting

**File:** `src/middleware/ratelimiter.js`
**Exports:** `{ globalLimiter, otpLimiter }`
**Dependencies:** `express-rate-limit`, `../../example.env`

### Purpose

Provides two IP-based rate limiters using the `express-rate-limit` package:
- `otpLimiter` — applied to OTP-related auth endpoints (active).
- `globalLimiter` — intended for all routes (currently inactive — commented out in `app.js`).

### `otpLimiter`

```js
rateLimit({
    windowMs: 10 * 60 * 1000,  // 10 minutes
    max: 5,                     // 5 requests per IP per window
    message: { error: "Too many OTP requests from this IP, please try again later." },
    standardHeaders: true,      // sends RateLimit-* headers
    legacyHeaders: false,       // no X-RateLimit-* headers
})
```

**Applied to these routes:**
- `POST /api/auth/verify/resend`
- `POST /api/auth/verify/confirm`
- `POST /api/auth/login/otp/request`
- `POST /api/auth/login/otp/verify`

**What it prevents:** An attacker trying to brute-force OTPs from a single IP is limited to 5 attempts per 10 minutes across all OTP endpoints combined (since the limiter is keyed by IP, not by IP+route).

**On limit exceeded:**
```json
{ "error": "Too many OTP requests from this IP, please try again later." }
```
Status: `429`. Response headers include `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`.

### `globalLimiter`

```js
rateLimit({
    windowMs: env.GLOBAL_RATE_LIMIT_WINDOW_MS,  // from .env
    max: env.RATE_LIMIT_MAX,                     // from .env
    message: { error: "Too many requests from this IP, please try again later." },
    standardHeaders: true,
    legacyHeaders: false,
})
```

**Status:** Defined but **NOT active**. In `app.js`:
```js
// app.use(globalLimiter);   ← commented out
```

> ⚠️ **There is no global rate limiting on the server.** Only OTP endpoints and the per-project ingest limiter are active.

### Storage

`express-rate-limit` uses in-memory storage by default. State is not shared across processes or server restarts. For distributed deployments, an external store (Redis) would be needed.

### Interview Explanation

> "`otpLimiter` uses `express-rate-limit` to cap OTP-related requests at 5 per 10 minutes per IP. This guards against OTP brute-forcing. The `globalLimiter` is defined but commented out in `app.js`, so there's no blanket IP rate limiting on the server right now — only OTP routes and the per-project ingest endpoint have limits."

---

## 7. `performanceTimer.js` — Request Timing

**File:** `src/middleware/performanceTimer.js`

### Purpose

Records how long each HTTP request takes and logs it. Applied globally to all routes.

**Implementation (inferred from usage in `app.js`):**
```js
app.use(performanceTimer);
```

Records `Date.now()` at request start, then on response finish calculates elapsed time and logs it.

> **Not verified in full:** The `performanceTimer.js` file was not read in full detail. The above is inferred from its position in `app.js` and its name.

---

## 8. Validation Middleware

Validation middleware uses **Joi** schemas. All validators run with `abortEarly: false` — all errors in a request are collected and returned together.

### 8.1 `validateRegister` — `src/validations/auth.Validation.js`

Applied to: `POST /api/auth/register`

| Field | Type | Rules |
|---|---|---|
| `username` | String | Alphanumeric only, trimmed, 3–100 chars, required |
| `email` | String | Valid email (TLD validation disabled), required |
| `password` | String | 8–500 chars, must match regex: ≥1 uppercase + ≥1 lowercase + ≥1 digit + ≥1 special char |

**Password regex:**
```
^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[!@#$%^&*()_+\-=\[\]{};':"\\|,.<>\/?]).+$
```

On failure: `400 { errors: ["message1", "message2", ...] }`

> ⚠️ **Discrepancy:** The error message string says `"Password must be at most 50 characters"` but `.max(500)` is the actual Joi rule. The validation enforces 500 chars, but the error message tells the user 50.

### 8.2 `validateLogin` — `src/validations/auth.Validation.js`

Applied to: `POST /api/auth/login`

| Field | Type | Rules |
|---|---|---|
| `email` | String | Valid email format, required |
| `password` | String | Required, no strength rules |

No password strength check at login — only at registration.

### 8.3 `validateEvent` — `src/validations/Event.validation.js`

Applied to: `POST /api/events/ingest/:projectId`

Validates **both** `req.params` and `req.body`. Params are validated first; if params fail, body is not validated.

**`req.params` schema:**

| Field | Rules |
|---|---|
| `projectId` | Hex string, exactly 24 chars (MongoDB ObjectId format), required |

**`req.body` schema:**

| Field | Required | Rules |
|---|---|---|
| `service` | ✅ | String, trimmed, 2–100 chars |
| `severity` | ✅ | One of `INFO`, `WARN`, `ERROR`, `CRITICAL` |
| `message` | ✅ | String, trimmed, 1–2000 chars |
| `metadata` | ❌ | Object, unknown keys allowed |
| `environment` | ❌ | One of `development`, `staging`, `production` |
| `eventTimestamp` | ❌ | Valid Joi date |

> ⚠️ **Discrepancy with controller:** `environment` is **optional** in the Joi schema. But `ingestEvent` controller explicitly checks `if (!environment) → 400`. This means the controller adds a stricter rule than the validator — the validator will pass a request without `environment`, then the controller rejects it. The effective behavior makes `environment` required.

### 8.4 `ValidateProject` — `src/validations/project.validation.js`

Applied to: `POST /api/project/create` and `GET /api/project/list`

| Field | Required | Rules |
|---|---|---|
| `projectName` | ✅ | String, trimmed, 3–200 chars |
| `description` | ❌ | String, trimmed, max 500 chars, empty string allowed |

> ⚠️ **Applied to `listProject` (GET) unnecessarily:** The `listProject` route has no request body. Joi validation passes trivially (all fields optional in the body context) but this is a meaningless middleware application.

### Validation Error Response Format

All validation middleware returns the same shape:
```json
{
    "errors": ["Error message 1", "Error message 2"]
}
```
Status: `400`

---

## 9. Socket.IO Authentication Middleware

### 9.1 Connection Authentication — `src/realtime/socket.server.js`

Applied to every new Socket.IO connection via `io.use(...)`:

```js
io.use(async (socket, next) => {
    const token = socket.handshake.auth?.token;
    if (!token) return next(new Error("Authentication token missing"));
    const user = await verifySocketToken(token);
    socket.userId = user.sub;
    next();
});
```

- Token is expected in `socket.handshake.auth.token` (sent by the client as `{ auth: { token: "<accessToken>" } }`).
- On success: `socket.userId` is set to the user's MongoDB ID.
- On failure: `next(new Error(...))` disconnects the socket with an error event.

> ⚠️ **CORS is set to `"*"` on the Socket.IO server** — all origins are accepted. The `CLIENT_URL` environment variable is imported but not used in the CORS configuration.

### 9.2 `verifySocketToken()` — `src/realtime/socket.auth.js`

```js
const { JWT_ACCESS_SECRET } = process.env;  // reads directly from process.env
```

> ⚠️ **Critical Bug:** This file reads `JWT_ACCESS_SECRET` directly from `process.env` instead of the centralized `example.env.js` config module that all other files use. If `.env` is loaded via `dotenv` before this module initializes, it works. If not (or in test environments), `JWT_ACCESS_SECRET` will be `undefined` and `jwt.verify()` will throw, rejecting all socket connections.

Steps:
1. Check `JWT_ACCESS_SECRET` is defined → throw if not.
2. Check token is not empty → throw if not.
3. `jwt.verify(token, JWT_ACCESS_SECRET)` → throws on invalid/expired.
4. Check `decoded.sub` exists → throw if not.
5. Return `decoded` payload.

All errors are caught and re-thrown as a generic `"Unauthorized socket connection"` — error details are not exposed to the client.

### 9.3 `registerSocketHandlers()` — `src/realtime/socket.manager.js`

Called after successful connection. Sets up per-socket event listeners.

#### `"subscribe"` event handler

Client sends: `{ projectId: "64abc..." }`

| Step | Action | Socket response on failure |
|---|---|---|
| 1 | Check `projectId` is provided | `"subscription-error": "Project ID is required"` |
| 2 | `mongoose.Types.ObjectId.isValid(projectId)` | `"subscription-error": "Invalid project Id format"` |
| 3 | `Project.findById(projectId).select("ownerId")` | `"subscription-error": "Project not found"` |
| 4 | `project.ownerId === socket.userId` | `"subscription-error": "Unauthorized access to this project"` + AuditLog attempt |
| 5 | `socket.join("project:<projectId>")` | — |
| 6 | Emit `"subscription-success"` | — |

**AuditLog on unauthorized subscribe:**
```js
AuditLog.create({
    purpose: "SOCKET_UNAUTHORIZED",
    projectId,
    userId: socket.userId,
    message: "Unauthorized socket subscription attempt"
})
```
> ⚠️ **Bug:** `AuditLog` is NOT imported in `socket.manager.js`. This throws a `ReferenceError` which is caught and logged. No audit record is created.

**Room name format:** `"project:<projectId>"` (e.g., `"project:64abc123def456..."`).

#### `"unsubscribe"` event handler

Client sends: `{ projectId: "..." }`. Handler calls `socket.leave("project:<projectId>")`. No ownership check on unsubscribe.

#### `"disconnect"` event handler

Calls `metrics.decrementSocketConnections()`. This is also registered in `socket.server.js` separately but `registerSocketHandlers` registers it on the socket — the `socket.server.js` disconnect handler only logs. No duplicate decrement issue since they target different things.

### Room Broadcasting by Controllers

Controllers emit to rooms using:
```js
const io = getIO();
io.to(`project:${projectId}`).emit("incident-updated", incidentObject);
```
And:
```js
emitEventToProject(io, projectId, eventObject);
// which calls: io.to(`project:${projectId}`).emit("new-event", eventData)
```

**Events emitted to rooms:**

| Event Name | Emitted by | Payload |
|---|---|---|
| `"new-event"` | `ingestEvent` (always) | Full event object |
| `"incident-updated"` | `ingestEvent` (ERROR/CRITICAL only) | Incident object |
| `"incident-updated"` | `updateIncidentStatus` | Updated incident object |

---

## 10. Error Handling Conventions

### HTTP Middleware Errors

All HTTP middleware follows these conventions:

| Scenario | Pattern |
|---|---|
| Client error (bad input, missing auth) | `return res.status(4xx).json({ message/error: "..." })` |
| Server error (unexpected) | `return res.status(500).json({ message: "...", error: err.message })` |
| Side-effect failure (audit log, metrics) | Wrapped in inner try/catch, logged, does not affect primary response |
| Validation errors | `return res.status(400).json({ errors: ["msg1", "msg2"] })` |

### Error Field Naming Inconsistency

Different middleware use different field names for errors:

| Source | Error field name |
|---|---|
| `auth.js` | `"error"` |
| `apiKeyAuth.js` | `"message"` |
| `projectRateLimiter.js` | `"message"` |
| `ratelimiter.js` (`otpLimiter`) | `"error"` |
| Validation middleware | `"errors"` (array) |

> ⚠️ There is **no consistent error response schema** across middleware. Frontend clients must handle multiple shapes.

### Partial Failures (Side Effects)

Several side effects are designed to fail silently:

| Side Effect | On Failure |
|---|---|
| AuditLog write in `apiKeyAuth` | Caught, logged, 403 still sent |
| AuditLog write in `rotateIngestKey` | Caught, logged, 200 still sent |
| AuditLog write in `projectRateLimiter` | ReferenceError (not imported), caught, 429 still sent |
| Socket.IO emit in `updateIncidentStatus` | Caught in separate try/catch, logged, 200 still sent |

---

## 11. Security Implementation Notes

### What Is Actually Implemented

| Protection | Implementation |
|---|---|
| Password hashing | bcrypt via Mongoose pre-save hook (auto on `save`) |
| OTP hashing | bcrypt (`makeOtp` utility) |
| OTP expiry | MongoDB TTL index + manual `expiresAt < new Date()` check |
| OTP brute force | Per-token attempt counter, locked at `OTP_MAX_ATTEMPTS` |
| OTP replay | `consumed` flag, set `true` after use |
| API key hashing | SHA-256 (not bcrypt — faster verification, less collision resistance) |
| API key replay | Not possible — key doesn't change unless rotated |
| JWT access token | Short-lived, signed with `JWT_ACCESS_SECRET` |
| JWT refresh token | Long-lived, signed with `JWT_REFRESH_SECRET` |
| Token invalidation | `tokenVersion` increment (affects refresh; access tokens expire naturally) |
| Email enumeration prevention | `loginWithPassword` returns same 400 for missing user and wrong password |
| Helmet security headers | `helmet()` in `app.js` |
| Body size limit | `express.json({ limit: "10kb" })` |
| IP rate limiting (OTP) | `otpLimiter` — 5 req / 10 min per IP |
| Per-project rate limiting | `projectRateLimiter` — 300 req / 60 sec per project (in-memory) |
| Ownership enforcement | Manual `project.ownerId === req.user.id` in every protected query |
| Audit logging | On API key failure and key rotation (partially broken — see discrepancies) |

### What Is NOT Implemented (But May Be Expected)

| Protection | Status |
|---|---|
| Global IP rate limiting | `globalLimiter` defined but **commented out** |
| Constant-time API key comparison | Uses `===` string compare — **not constant-time** |
| Socket.IO CORS restriction | Set to `"*"` — all origins accepted |
| Distributed rate limiting | Both limiters are in-memory only |
| Access token revocation | Not possible — stateless JWT |
| Account lockout after failed passwords | Not implemented |
| Request logging | Not verified beyond `performanceTimer` |

---

## 12. Documentation Discrepancies Found

These are differences between what might be expected/claimed and what the source code actually does.

| # | Discrepancy | Actual Behavior |
|---|---|---|
| 1 | `globalLimiter` described as active | It is **commented out** in `app.js` — no global rate limit |
| 2 | `environment` is optional in Joi schema | Controller rejects requests without it — effectively required |
| 3 | Password max length error message says 50 chars | Joi rule is `.max(500)` — message is wrong |
| 4 | `ValidateProject` applied to `listProject` | `listProject` has no body — validation is meaningless but harmless |
| 5 | AuditLog written on rate limit exceed | `AuditLog` not imported in `projectRateLimiter.js` — never written |
| 6 | AuditLog written on unauthorized socket subscribe | `AuditLog` not imported in `socket.manager.js` — never written |
| 7 | `JWT_ACCESS_SECRET` from env config | `socket.auth.js` reads directly from `process.env`, not from `example.env.js` |
| 8 | Socket.IO CORS uses `CLIENT_URL` | `CLIENT_URL` is imported but not used — CORS is `"*"` |
| 9 | `logoutEverywhere` fully invalidates access tokens | Only invalidates refresh tokens immediately — access tokens remain valid until expiry |
| 10 | API key comparison is secure | Uses `===` (not constant-time) — theoretically vulnerable to timing attack |
| 11 | `rotateIngestKey` writes AuditLog | `ipAddress` (required field) is not provided — AuditLog write silently fails |

---

## 13. Interview Preparation Notes

### Likely Technical Questions and How to Answer Them

---

**Q: How does authentication work in this project?**

> "We have two authentication systems. For human users, we use JWT — an access token (short-lived, ~15m) and a refresh token (long-lived, ~7d). The `authRequired` middleware verifies the access token signature and extracts the user ID and token version from the payload — it never queries the database. For external services sending log events, we use per-project ingest API keys. The `apiKeyAuth` middleware takes the key from the `x-api-key` header, SHA-256 hashes it, and compares it to the stored hash in MongoDB."

---

**Q: How do you prevent someone from brute-forcing OTPs?**

> "Three layers: First, the `otpLimiter` middleware limits any IP to 5 OTP-related requests per 10 minutes. Second, each `OtpToken` document has an `attempts` counter — after `OTP_MAX_ATTEMPTS` wrong guesses, that specific token is locked with a 429. Third, OTP tokens have a TTL (configured in `.env`, default 10 minutes) enforced by both a MongoDB TTL index that auto-deletes the document and a manual `expiresAt < new Date()` check in the controller."

---

**Q: How does logoutEverywhere work and is it truly immediate?**

> "It increments `tokenVersion` in the user's database record. All new refresh tokens carry this new version, so old refresh tokens fail the version check when used. However, existing **access tokens** are not immediately revoked because `authRequired` is stateless — it reads `tokenVersion` from the JWT payload, not from the database. So old access tokens continue to work until they expire naturally. This is a known trade-off of stateless JWTs."

---

**Q: How does the API key system work?**

> "When a project is created, we generate a 32-byte cryptographically random key using `crypto.randomBytes(32)`, which gives a 64-character hex string. We SHA-256 hash it and store only the hash in MongoDB with `select: false` so it's never included in normal queries. The plain key is returned once and never stored. To verify a submitted key, we SHA-256 hash it and compare to the stored hash. To rotate the key, we generate a new one, replace the hash in the database, and return the new plain key once."

---

**Q: How does real-time event streaming work?**

> "Socket.IO rooms. When a user connects, they send a `subscribe` event with a `projectId`. The server verifies they own that project, then calls `socket.join('project:<projectId>')`. When an event is ingested via HTTP, the `ingestEvent` controller calls `io.to('project:<projectId>').emit('new-event', eventData)`. If the event is `ERROR` or `CRITICAL` and creates/updates an incident, it also emits `'incident-updated'`. All sockets in that room receive the broadcast simultaneously."

---

**Q: How does incident grouping work?**

> "When an `ERROR` or `CRITICAL` event is ingested, we compute a `messageSignature` by lowercasing and trimming the event message. We then do a `findOneAndUpdate` on the Incident collection looking for an existing `OPEN` or `ACKNOWLEDGED` incident with the same `projectId` and `messageSignature`. If found, we increment its `eventCount` and update `lastOccurredAt`. If not found, we create a new incident. This deduplicates repeated errors into a single trackable incident rather than creating thousands of separate records."

---

**Q: How does cursor-based pagination work in `getProjectEvents`?**

> "We accept a `before` query parameter — an ISO date string. The MongoDB query becomes `{ projectId, eventTimestamp: { $lt: beforeDate } }`, sorted newest-first with a `limit`. For the first page, omit `before` — you get the newest `limit` events. For the next page, pass the `eventTimestamp` of the last event you received. This is stable under concurrent inserts, unlike offset pagination which can skip or duplicate records when new data is inserted."

---

**Q: What could you improve in this middleware?**

> "Several things: 1) Use `crypto.timingSafeEqual()` for API key comparison instead of `===`. 2) Import and fix `AuditLog` in `projectRateLimiter.js` and `socket.manager.js`. 3) Add a `userId` ownership check to `authRequired` with a configurable DB lookup to support true access token revocation. 4) Use Redis for both rate limiters to support distributed deployments. 5) Restrict Socket.IO CORS to `CLIENT_URL` instead of `*`. 6) Enable `globalLimiter` as a base protection layer."

---

## Files Inspected for This Document

| File | Purpose |
|---|---|
| `src/middleware/auth.js` | JWT authentication middleware |
| `src/middleware/apiKeyAuth.js` | API key authentication middleware |
| `src/middleware/projectRateLimiter.js` | Per-project rate limiter |
| `src/middleware/ratelimiter.js` | IP-based rate limiters |
| `src/middleware/performanceTimer.js` | Request timing (name only — not fully read) |
| `src/realtime/socket.server.js` | Socket.IO server + connection middleware |
| `src/realtime/socket.auth.js` | JWT verification for WebSocket |
| `src/realtime/socket.manager.js` | Socket event handlers |
| `src/validations/auth.Validation.js` | Register + login Joi validators |
| `src/validations/Event.validation.js` | Event ingest Joi validator |
| `src/validations/project.validation.js` | Project create Joi validator |
| `src/models/Project.js` | Project model + API key methods |
| `src/models/AuditLog.js` | Audit log model |
| `src/models/Users.js` | User model + password hashing |
| `src/models/OtpToken.js` | OTP token model |
| `src/utils/metrics.js` | In-memory metrics singleton |
| `src/utils/genrateOtp.js` | OTP generation utility |
| `src/utils/sendEmail.js` | SendGrid email utility |
| `backend/app.js` | Express app setup + middleware mounting |
| `backend/server.js` | Server bootstrap |
| All 5 route files | Verified middleware chain per route |
