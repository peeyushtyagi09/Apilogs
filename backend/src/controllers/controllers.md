# Controllers — Complete Technical Documentation

> **Project:** API Logs (Apilogs)
> **Location:** `backend/src/controllers/`
> **Last Updated:** 2026-09-15
> **Purpose of this document:** This is a full technical reference for every controller in the backend. It documents what each file does, all of its functions, every field it reads/writes, every model it touches, what middleware runs before it, how errors are handled, and what gets sent back to the client. Written so that an outside reader or AI assistant can fully understand the backend without reading source code.

---

## Project Overview (Brief)

This backend is an **API event logging and incident tracking system**. External services (servers, microservices, apps) send log events to this backend using a secret **ingest API key**. The backend stores those events, auto-groups repeated errors into **incidents**, and broadcasts real-time updates over **WebSocket (Socket.IO)**. Human users log in via a dashboard (the frontend) and monitor their projects, events, and incidents.

### Core Resources

| Resource | Description |
|---|---|
| **User** | A registered human user of the dashboard |
| **Project** | A monitored service/app registered by a user. Has a secret ingest key |
| **Event** | A single log entry sent by an external service to a project |
| **Incident** | An auto-grouped record of repeated ERROR/CRITICAL events |
| **OtpToken** | A one-time password record used for email verification and passwordless login |
| **AuditLog** | An immutable record of security-relevant events (failed key attempts, key rotations) |

### Architecture at a Glance

```
External Service
    │
    │ POST /api/events/ingest/:projectId
    │ Header: x-api-key: <ingestKey>
    ▼
[apiKeyAuth middleware]  ← validates key, loads project, sets req.project
[projectRateLimiter]     ← per-project rate limiting
[validateEvent]          ← Joi schema validation
    ▼
event.controller.js (ingestEvent)
    ├── saves Event to DB
    ├── if ERROR/CRITICAL → creates/updates Incident in DB
    └── emits real-time updates via Socket.IO

Human Dashboard User
    │
    │ POST /api/auth/login
    ▼
[validateLogin middleware]
    ▼
auth.Controller.js (loginWithPassword)
    └── returns JWT access + refresh tokens

Authenticated User
    │
    │ GET /api/events/:projectId
    │ Header: Authorization: Bearer <accessToken>
    ▼
[authRequired middleware]  ← verifies JWT, sets req.user
    ▼
event.controller.js (getProjectEvents)
    └── returns paginated events for the project
```

---

## Middleware Used by Controllers

Before reading each controller, understand what middleware runs before the controller functions. This context is critical.

---

### `authRequired` — `src/middleware/auth.js`

**Used by:** Auth (logout), Event (getProjectEvents), Incident (both), Project (all)

**Purpose:** Protects routes that require a logged-in user. Verifies a JWT access token and attaches user info to `req.user`.

**How it works:**
1. Reads the `Authorization` header.
2. Expects the format `Bearer <token>`. Returns `401` if missing or malformed.
3. Verifies the token with `JWT_ACCESS_SECRET` from `.env`.
4. Checks that the payload has `sub` (user ID) and `tv` (token version) fields.
5. On success: sets `req.user = { id: payload.sub, tokenVersion: payload.tv }` and calls `next()`.
6. Any failure (bad format, expired, invalid signature, missing fields) → `401`.

**What it sets on `req`:**
```js
req.user = {
  id: "64abc...",         // MongoDB ObjectId string of the user
  tokenVersion: 3         // Used to validate global logout
}
```

> **Important:** The controller does NOT verify `tokenVersion` against the DB itself — only the JWT payload is checked. The full token invalidation check on logout works by incrementing `tokenVersion` in the DB, which causes all old tokens to fail the check at issue time (when a new token is minted with the updated version).

---

### `apiKeyAuth` — `src/middleware/apiKeyAuth.js`

**Used by:** Event (ingestEvent only)

**Purpose:** Authenticates requests from **external services** (not human users). Uses a project-specific ingest API key instead of a JWT.

**How it works:**
1. Reads `projectId` from `req.params`.
2. Reads the API key from the `x-api-key` request header.
3. Returns `401` if no key is provided.
4. Returns `400` if `projectId` is not a valid MongoDB ObjectId.
5. Finds the project from DB, explicitly selecting the hidden `ingestKeyHash` field (`select: false` in model).
6. Calls `project.verifyIngestKey(providedKey)`:
   - Hashes the provided key with SHA-256
   - Compares the hash to `ingestKeyHash` stored in DB
   - Returns `true` if they match
7. If key is **invalid**: saves an `AuditLog` record with `purpose: "API_KEY_FAILED"` and the caller's IP address. Returns `403`.
8. If key is **valid**: sets `req.project = project` and calls `next()`.
9. Any unexpected error: calls `metrics.incrementApiKeyFailure()` and returns `500`.

**What it sets on `req`:**
```js
req.project = {
  _id: ObjectId("..."),
  projectName: "My Service",
  ownerId: ObjectId("..."),
  ingestKeyHash: "sha256_hash_here",   // hidden field, explicitly selected
  ...
}
```

---

### `projectRateLimiter` — `src/middleware/projectRateLimiter.js`

**Used by:** Event (ingestEvent only)

**Purpose:** Rate-limits ingest requests per project, preventing a single project from flooding the system.

---

### `otpLimiter` — `src/middleware/ratelimiter.js`

**Used by:** Auth (resend OTP, verify OTP, login OTP routes)

**Purpose:** Rate-limits OTP-related endpoints to prevent brute-force and spam attacks.

---

## Models Reference

Each controller works with one or more of these MongoDB models. Understanding their structure is essential for understanding controller logic.

---

### `User` Model — `src/models/Users.js`

```
Collection: users
```

| Field | Type | Notes |
|---|---|---|
| `username` | String | required, unique, trimmed, 3–100 chars |
| `email` | String | required, unique, trimmed, auto-lowercased |
| `password` | String | required, stored as bcrypt hash (auto-hashed on save via pre-hook) |
| `isVerified` | Boolean | default `false`. Must be `true` before user can log in |
| `tokenVersion` | Number | default `0`. Incremented on `logoutEverywhere` to invalidate all active JWTs |
| `createdAt` | Date | auto-managed by Mongoose timestamps |
| `updatedAt` | Date | auto-managed by Mongoose timestamps |

**Model-level behaviours:**
- **Pre-save hook:** Runs `bcrypt.hash(password, saltRounds)` automatically whenever `password` is modified before saving. The salt rounds come from `SaltValue` in `.env`.
- **Instance method `comparePassword(plain)`:** Runs `bcrypt.compare(plain, this.password)`. Returns a boolean.

---

### `OtpToken` Model — `src/models/OtpToken.js`

```
Collection: otptokens
```

| Field | Type | Notes |
|---|---|---|
| `user` | ObjectId → User | required, indexed |
| `purpose` | String | enum: `"login"` or `"verify"` |
| `codeHash` | String | bcrypt hash of the OTP. Never the plain OTP |
| `attempts` | Number | default `0`. Incremented on each wrong guess |
| `consumed` | Boolean | default `false`. Set to `true` when OTP is used or invalidated |
| `expiresAt` | Date | required. MongoDB TTL index automatically deletes the document after this date |

**Model-level behaviours:**
- **TTL index:** `{ expiresAt: 1 }, { expireAfterSeconds: 0 }` — MongoDB automatically removes expired documents.
- **Static method `findValid(userId, purpose)`:** Finds an unconsumed, unexpired OTP for a user/purpose combination.

---

### `Project` Model — `src/models/Project.js`

```
Collection: projects
```

| Field | Type | Notes |
|---|---|---|
| `projectName` | String | required, 3–200 chars |
| `description` | String | optional, max 500 chars, defaults to `""` |
| `ownerId` | ObjectId → User | required, indexed |
| `ingestKeyHash` | String | SHA-256 hash of the ingest API key. `select: false` — never returned by default |
| `createdAt` | Date | auto-managed |
| `updatedAt` | Date | auto-managed |

**Model-level behaviours:**
- **Instance method `generateIngestKey()`:**
  1. Generates 32 cryptographically random bytes → converts to 64-char hex string (this is the raw API key).
  2. SHA-256 hashes the raw key.
  3. Stores the hash on `this.ingestKeyHash`.
  4. Returns the **plain raw key** (only time it is ever accessible).
- **Instance method `verifyIngestKey(providedKey)`:**
  1. SHA-256 hashes the `providedKey`.
  2. Compares the hash with `this.ingestKeyHash`.
  3. Returns `true` or `false`.

> **Security note:** The plain API key is NEVER stored in the database. Only its SHA-256 hash is stored. This means even if the DB is compromised, the actual API keys cannot be recovered.

---

### `Event` Model — `src/models/Event.js`

```
Collection: events
```

| Field | Type | Notes |
|---|---|---|
| `projectId` | ObjectId → Project | required, indexed |
| `service` | String | required, trimmed, max 100 chars, indexed |
| `severity` | String | enum: `"INFO"`, `"WARN"`, `"ERROR"`, `"CRITICAL"`. indexed |
| `message` | String | required, trimmed, max 2000 chars |
| `metadata` | Mixed | optional free-form object, defaults to `{}` |
| `environment` | String | enum: `"development"`, `"staging"`, `"production"`. indexed |
| `eventTimestamp` | Date | defaults to `Date.now`. The actual time the event occurred (may differ from ingestion time) |
| `createdAt` | Date | auto-managed (ingestion time) |

**Indexes:**
- Compound index on `{ projectId: 1, eventTimestamp: -1 }` for fast paginated queries by project.

---

### `Incident` Model — `src/models/Incident.js`

```
Collection: incidents
```

| Field | Type | Notes |
|---|---|---|
| `projectId` | ObjectId → Project | required, indexed |
| `service` | String | required, indexed — which service caused the incident |
| `severity` | String | enum: `"INFO"`, `"WARN"`, `"ERROR"`, `"CRITICAL"`. indexed |
| `messageSignature` | String | lowercased, trimmed version of the error message. Used for deduplication |
| `status` | String | enum: `"OPEN"`, `"ACKNOWLEDGED"`, `"RESOLVED"`. default `"OPEN"`. indexed |
| `firstOccurredAt` | Date | when the first matching event was received |
| `lastOccurredAt` | Date | updated every time a new matching event is received |
| `eventCount` | Number | total number of matching events grouped into this incident |

**Indexes:**
- Compound index on `{ projectId: 1, messageSignature: 1, status: 1 }` — heavily used by `ingestEvent` to quickly find existing open incidents.

**How incidents are deduplicated:**
When a new `ERROR` or `CRITICAL` event arrives, the `messageSignature` field (lowercased + trimmed version of the message) is used as a grouping key. If an `OPEN` or `ACKNOWLEDGED` incident with the same signature already exists for that project, the event is added to that incident (incrementing `eventCount` and updating `lastOccurredAt`) rather than creating a new incident.

---

### `AuditLog` Model — `src/models/AuditLog.js`

```
Collection: auditlogs
```

| Field | Type | Notes |
|---|---|---|
| `purpose` | String | enum: `"API_KEY_FAILED"`, `"SOCKET_UNAUTHORIZED"`, `"RATE_LIMIT_EXCEEDED"`, `"KEY_ROTATED"` |
| `projectId` | ObjectId → Project | required, indexed |
| `userId` | ObjectId → User | optional (not always known for failed requests) |
| `ipAddress` | String | required — the caller's IP |
| `message` | String | required, max 500 chars, human-readable description |
| `createdAt` | Date | auto-managed |

**Purpose:** Provides an immutable security trail. Never mutated after creation.

---

## Environment Variables Used by Controllers

These are read from `.env` (via `example.env.js`):

| Variable | Used By | Purpose |
|---|---|---|
| `JWT_ACCESS_SECRET` | `auth.Controller`, `auth middleware` | Signs and verifies access tokens |
| `JWT_REFRESH_SECRET` | `auth.Controller` | Signs and verifies refresh tokens |
| `JWT_ACCESS_EXPIRES_IN` | `auth.Controller` | Access token lifetime (e.g. `"15m"`) |
| `JWT_REFRESH_EXPIRES_IN` | `auth.Controller` | Refresh token lifetime (e.g. `"7d"`) |
| `OTP_MAX_ATTEMPTS` | `auth.Controller` | Max wrong OTP guesses before lockout |
| `OTP_LENGTH` | `genrateOtp.js` util | Number of digits in generated OTP |
| `OTP_TTL_SECONDS` | `genrateOtp.js` util | OTP expiry time in seconds from `Date.now()` |
| `SALT` | `genrateOtp.js` util | bcrypt salt rounds for hashing OTPs |

---

## Utility Functions Used by Controllers

### `makeOtp()` — `src/utils/genrateOtp.js`

Called by: `auth.Controller.js` (register, resendVerificationOtp, requestLoginOtp)

**What it does:**
1. Generates a cryptographically secure numeric OTP string of length `OTP_LENGTH` (from `.env`, default 6).
   - Uses `crypto.randomBytes` internally (not `Math.random`) for security.
   - Rejects byte values ≥ 250 to avoid modulo bias.
2. Hashes the plain OTP using `bcrypt` with `SALT` rounds.
3. Calculates `expiresAt = Date.now() + OTP_TTL_SECONDS * 1000`.

**Returns:** `{ plain: "482910", hash: "$2b$10$...", expiresAt: Date }`

The `plain` value is what gets emailed to the user. The `hash` is what gets stored in the DB.

---

### `MetricsStore` — `src/utils/metrics.js`

A singleton in-memory counter class (instantiated once at startup, shared across all requests).

| Counter | Incremented by |
|---|---|
| `totalEventsIngested` | `event.controller.js` on every successful ingest |
| `eventsLastMinute` | Same — reset every 60 seconds via `setInterval` |
| `failedApiKeyAttempts` | `apiKeyAuth` middleware on unexpected errors |
| `rateLimitHits` | Rate limiter middleware |
| `activeSocketConnections` | Socket.IO connect/disconnect events |

**Methods exposed to controllers:**
- `metrics.incrementEvent()` — called in `ingestEvent`
- `metrics.getSnapshot()` — called in `system.controller.js getMetrics`
- `metrics.getActiveSocketConnections()` — called in `system.controller.js getHealth`

> **Note:** These counters are **in-memory only**. They reset to zero every time the server restarts. They are not persisted to the database.

---

---

# Controller Files — Detailed Documentation

---

## 1. `auth.Controller.js`

**File:** `src/controllers/auth.Controller.js`  
**Route prefix:** `/api/auth`  
**Imported by:** `src/routes/auth.Routes.js`

### Purpose

This controller owns the entire user authentication lifecycle. It handles:
- New user registration (with email verification)
- Resending a verification OTP
- Confirming email via OTP
- Password-based login
- OTP-based (passwordless) login — two step: request OTP → verify OTP
- Refreshing expired JWT access tokens using a refresh token
- Invalidating all active sessions for a user (global logout)

It uses **two-factor patterns** throughout: whenever a sensitive action is performed (register, login), an OTP is sent to the user's email and must be verified before access tokens are issued.

### Internal Helper: `signToken(user)`

This is a private helper function, not exported. It is called internally whenever the controller needs to issue a token pair.

```js
function signToken(user) {
    const payload = { sub: user.id, tv: user.tokenVersion };
    return {
        accessToken: jwt.sign(payload, env.JWT_ACCESS_SECRET, { expiresIn: env.JWT_ACCESS_EXPIRES_IN }),
        refreshToken: jwt.sign(payload, env.JWT_REFRESH_SECRET, { expiresIn: env.JWT_REFRESH_EXPIRES_IN })
    };
}
```

- `sub` is the user's MongoDB `_id` as a string.
- `tv` (tokenVersion) is the current version number. When `logoutEverywhere` is called, this number is incremented in the DB, making all previously issued tokens invalid — if someone tries to use an old token and then refreshes, the new token will have an outdated `tv` and will fail.
- The access token is short-lived (e.g. 15 minutes) and is used for every API request.
- The refresh token is long-lived (e.g. 7 days) and is only used to obtain a new access token.

---

### Route Map for Auth

| Method | Path | Middleware Before Controller | Controller Function |
|---|---|---|---|
| POST | `/api/auth/register` | `validateRegister` | `register` |
| POST | `/api/auth/verify/resend` | `otpLimiter` | `resendVerificationOtp` |
| POST | `/api/auth/verify/confirm` | `otpLimiter` | `verifyEmail` |
| POST | `/api/auth/login` | `validateLogin` | `loginWithPassword` |
| POST | `/api/auth/login/otp/request` | `otpLimiter` | `requestLoginOtp` |
| POST | `/api/auth/login/otp/verify` | `otpLimiter` | `verifyLoginOtp` |
| POST | `/api/auth/token/refresh` | _(none)_ | `refreshToken` |
| POST | `/api/auth/logout-everywhere` | `authRequired` | `logoutEverywhere` |

---

### `exports.register`

**Route:** `POST /api/auth/register`  
**Middleware:** `validateRegister` (Joi validation runs first)  
**Auth needed:** No

**Purpose:** Creates a new user account and sends a verification OTP to the user's email.

**Request body:**
```json
{
  "username": "john_doe",
  "email": "john@example.com",
  "password": "MySecret123"
}
```

**Step-by-step flow:**

1. **Duplicate check:** `User.findOne({ email })` — if a user with that email already exists, return `400`.
   > Why not check username? The schema enforces uniqueness at the DB level, but the email check is done explicitly here for a cleaner error message.

2. **Create user:** `new User({ username, email, password })` — the `password` is plain text at this point. The `UserSchema.pre("save")` hook will hash it automatically before DB write.

3. **OTP generation:** `makeOtp()` returns `{ plain, hash, expiresAt }`.
   - `plain` is a numeric string like `"482910"`.
   - `hash` is a bcrypt hash of the plain value.
   - `expiresAt` is `Date.now() + OTP_TTL_SECONDS * 1000`.

4. **Save OTP record:** `OtpToken.create({ user: user._id, purpose: "verify", codeHash: hash, expiresAt })`.

5. **Send email:** `sendOtpEmail(user.email, plain, "verify")` sends the plain OTP to the user's email via SendGrid.

6. **Return:** `201` with `{ message: "Registered. Verification OTP sent to email." }`.

**Error responses:**

| Status | Condition |
|---|---|
| `400` | Email already registered |
| `500` | Unexpected DB/email error (unhandled, will bubble up) |

---

### `exports.resendVerificationOtp`

**Route:** `POST /api/auth/verify/resend`  
**Middleware:** `otpLimiter`  
**Auth needed:** No

**Purpose:** Generates and sends a fresh verification OTP. Invalidates any previously active verification OTPs first, so the user always has only one valid OTP at a time.

**Request body:**
```json
{ "email": "john@example.com" }
```

**Step-by-step flow:**

1. Find user by email → `404` if not found.
2. If `user.isVerified === true` → `400` ("Already verified") — no point resending.
3. Invalidate all existing unconsumed `"verify"` OTPs for this user:
   ```js
   OtpToken.updateMany(
     { user: user._id, purpose: "verify", consumed: false },
     { consumed: true }
   )
   ```
4. Generate a new OTP, save it, and send it via email.
5. Return `200` with `{ message: "Verification OTP resent." }`.

**Error responses:**

| Status | Condition |
|---|---|
| `400` | User already verified |
| `404` | User not found |

---

### `exports.verifyEmail`

**Route:** `POST /api/auth/verify/confirm`  
**Middleware:** `otpLimiter`  
**Auth needed:** No

**Purpose:** Confirms the user's email by matching the submitted OTP against the stored hash. On success, marks the user as verified and issues JWTs.

**Request body:**
```json
{
  "email": "john@example.com",
  "otp": "482910"
}
```

**Step-by-step flow:**

1. Find user by email → `404` if not found.
2. Find the **latest unconsumed** OTP for this user with `purpose: "verify"`:
   ```js
   OtpToken.findOne({ user: user._id, purpose: "verify", consumed: false }).sort({ createdAt: -1 })
   ```
3. If no token found or `token.expiresAt < new Date()` → `400` ("OTP expired or not found").
4. If `token.attempts >= OTP_MAX_ATTEMPTS` → `429` ("Max attempts exceeded").
5. Compare: `bcrypt.compare(otp, token.codeHash)`.
   - If **no match:** `token.attempts += 1`, save the token, return `400` ("Invalid OTP").
   - Why increment attempts before comparing more carefully: This brute-force protection means that even if an attacker knows the email, they can only try `OTP_MAX_ATTEMPTS` times before being locked out.
6. On **match:** `token.consumed = true` (save), `user.isVerified = true` (save).
7. Call `signToken(user)` and return `200` with `{ message, accessToken, refreshToken }`.

**Error responses:**

| Status | Condition |
|---|---|
| `400` | OTP expired/not found, or OTP is wrong |
| `404` | User not found |
| `429` | Too many wrong attempts |

---

### `exports.loginWithPassword`

**Route:** `POST /api/auth/login`  
**Middleware:** `validateLogin` (Joi validation)  
**Auth needed:** No

**Purpose:** Standard password-based login. Returns tokens immediately on success.

**Request body:**
```json
{
  "email": "john@example.com",
  "password": "MySecret123"
}
```

**Step-by-step flow:**

1. `User.findOne({ email })` → returns `400` if not found.
   > Why `400` and not `404`? This is intentional. Returning a `404` for "email not found" and `400` for "wrong password" would allow an attacker to enumerate which emails are registered. Using the same `400` for both cases prevents this (known as "credential stuffing protection").

2. `user.comparePassword(password)` — runs `bcrypt.compare` internally → `400` if wrong.

3. `user.isVerified` check → `403` if email is not verified. This case is separate because it gives the user a helpful hint to go verify their email first.

4. `signToken(user)` → returns `200` with `{ accessToken, refreshToken }`.

**Error responses:**

| Status | Condition |
|---|---|
| `400` | Invalid email or wrong password |
| `403` | Email not verified yet |

---

### `exports.requestLoginOtp`

**Route:** `POST /api/auth/login/otp/request`  
**Middleware:** `otpLimiter`  
**Auth needed:** No

**Purpose:** Step 1 of passwordless (OTP-based) login. Sends a fresh login OTP to the user's email. This is an alternative to password login used for step-up authentication or when no password is available.

**Request body:**
```json
{ "email": "john@example.com" }
```

**Step-by-step flow:**

1. Find user by email → `404` if not found.
2. Check `isVerified` → `403` if not (must verify email before using OTP login).
3. Invalidate all existing unconsumed `"login"` OTPs:
   ```js
   OtpToken.updateMany({ user: user._id, purpose: "login", consumed: false }, { consumed: true })
   ```
4. Generate new OTP, create `OtpToken` record with `purpose: "login"`, send email.
5. Return `200` with `{ message: "Login OTP sent." }`.

**Error responses:**

| Status | Condition |
|---|---|
| `403` | Email not verified |
| `404` | User not found |

---

### `exports.verifyLoginOtp`

**Route:** `POST /api/auth/login/otp/verify`  
**Middleware:** `otpLimiter`  
**Auth needed:** No

**Purpose:** Step 2 of passwordless login. Validates the submitted OTP against the stored hash and returns tokens.

**Request body:**
```json
{
  "email": "john@example.com",
  "otp": "839201"
}
```

**Step-by-step flow:**

Identical logic to `verifyEmail`, but:
- Looks for OTPs with `purpose: "login"` instead of `"verify"`.
- Does **not** set `user.isVerified`. The user is already verified.
- On success: returns tokens directly without any user field updates.

**Error responses:** Same as `verifyEmail`.

---

### `exports.refreshToken`

**Route:** `POST /api/auth/token/refresh`  
**Middleware:** None  
**Auth needed:** Refresh token (not a standard JWT access token)

**Purpose:** Issues a new access + refresh token pair using an existing, valid refresh token. This allows users to stay logged in without re-authenticating once their short-lived access token expires.

**Token sources (checked in order):**
1. `req.body.token` — body field
2. `Authorization: Bearer <token>` header

**Step-by-step flow:**

1. Extract the token from body or header → `400` if neither is present.
2. `jwt.verify(token, env.JWT_REFRESH_SECRET)` — verifies signature and expiry.
3. `User.findById(payload.sub)` — fetch the user.
4. Check `user.tokenVersion === payload.tv` — if not equal, it means `logoutEverywhere` was called after this refresh token was issued, so the token is now invalid.
5. On success: `signToken(user)` → return `200` with new `{ accessToken, refreshToken }`.
6. Any error in steps 2–4 → `401`.

> **Why issue a new refresh token too?** Refresh token rotation. Each call to this endpoint invalidates the old refresh token conceptually (the client replaces it) and the new one has a fresh expiry. This limits the window of exposure if a refresh token is stolen.

**Error responses:**

| Status | Condition |
|---|---|
| `400` | No token provided |
| `401` | Invalid/expired token, or user not found, or tokenVersion mismatch |

---

### `exports.logoutEverywhere`

**Route:** `POST /api/auth/logout-everywhere`  
**Middleware:** `authRequired`  
**Auth needed:** Yes (valid access token required)

**Purpose:** Logs the user out of **all** devices and sessions simultaneously by invalidating every JWT ever issued to them.

**How it works technically:**
- Fetches the user from DB using `req.user.id` (set by `authRequired`).
- Increments `user.tokenVersion` by 1.
- Saves the user.
- All previously issued JWTs have `tv = old_version`. When any of those tokens tries to refresh or is verified, the `tv` check will fail because the DB now has a higher version.

**Important:** This does NOT delete any sessions or tokens. JWTs cannot be revoked directly (they're stateless). Instead, the version mismatch makes them useless.

**Error responses:**

| Status | Condition |
|---|---|
| `200` | `{ message: "Logged out from all sessions" }` |
| `404` | User not found (shouldn't happen since `authRequired` passed, but guarded anyway) |

---

---

## 2. `event.controller.js`

**File:** `src/controllers/event.controller.js`  
**Route prefix:** `/api/events`  
**Imported by:** `src/routes/event.Routes.js`

### Purpose

This is the **core data ingestion controller**. It receives structured log events from external services, persists them, and auto-creates/updates incidents for high-severity events. It also pushes real-time updates to the frontend via WebSocket.

### Route Map for Events

| Method | Path | Middleware Before Controller | Controller Function |
|---|---|---|---|
| POST | `/api/events/ingest/:projectId` | `apiKeyAuth` → `projectRateLimiter` → `validateEvent` | `ingestEvent` |
| GET | `/api/events/:projectId` | `authRequired` | `getProjectEvents` |

---

### `ingestEvent`

**Route:** `POST /api/events/ingest/:projectId`  
**Auth:** API Key (`x-api-key` header), handled by `apiKeyAuth` middleware  
**Auth needed:** Yes (API key, not JWT)

**Purpose:** The primary entry point for external services to send log events. Validates the event, stores it, handles incident creation/update logic, and pushes real-time updates.

**By the time this function runs, the middleware has already:**
- Validated the API key and verified ownership
- Rate-checked the request
- Validated the request body structure with Joi
- Set `req.project` to the loaded project document

**Request params:** `:projectId` — MongoDB ObjectId of the target project

**Request body:**
```json
{
  "service": "payment-service",
  "severity": "ERROR",
  "message": "Database connection timeout",
  "environment": "production",
  "metadata": {
    "host": "db-01.internal",
    "retries": 3
  },
  "eventTimestamp": "2026-09-15T15:30:00.000Z"
}
```

**Field details:**

| Field | Type | Required | Allowed Values | Notes |
|---|---|---|---|---|
| `service` | String | ✅ | Any string | Name of the sending service |
| `severity` | String | ✅ | `INFO`, `WARN`, `ERROR`, `CRITICAL` | Validated by Joi before reaching here |
| `message` | String | ✅ | Any string, max 2000 chars | Human-readable event description |
| `environment` | String | ✅ | `development`, `staging`, `production` | Which deployment environment |
| `metadata` | Object | ❌ | Any key-value pairs | Extra context, stored as Mixed type |
| `eventTimestamp` | ISO Date String | ❌ | Valid ISO 8601 date | When the event actually occurred. Defaults to time of ingestion |

**Step-by-step flow:**

**Step 1 — Validate project presence:**
```js
if (!project || !project._id) → 400
```
This is a defensive check. The `apiKeyAuth` middleware should have already set `req.project`, but the controller guards against it being missing.

**Step 2 — Validate required body fields:**
```js
if (!service || !severity || !message || !environment) → 400
```
This is a secondary check. Joi validation should have caught this already, but it's done here as a belt-and-suspenders guard.

**Step 3 — Parse `eventTimestamp`:**
If `eventTimestamp` is provided in the body:
- Parse it with `new Date(eventTimestamp)`.
- If the result is `NaN` (invalid date string) → `400` ("Invalid eventTimestamp").
- If omitted, use `new Date()` (current server time).

**Step 4 — Save the event to DB:**
```js
const event = await Event.create({
    projectId: project._id,
    service,
    severity,
    message,
    metadata: metadata || {},
    environment,
    eventTimestamp: usedEventTimestamp
});
```

**Step 5 — Increment metrics counter:**
```js
metrics.incrementEvent();
```
Increments both `totalEventsIngested` and `eventsLastMinute` in the in-memory `MetricsStore`.

**Step 6 — Get Socket.IO instance:**
```js
const io = getIO();
```
`getIO()` returns the Socket.IO server instance registered in `socket.server.js`. This is used to push real-time events.

**Step 7 — Incident logic (only for `ERROR` or `CRITICAL`):**

This is the most complex part of the function. The goal is to avoid creating a new incident document every time the same error repeats.

```
messageSignature = message.trim().toLowerCase()
```

Look for an existing incident:
```js
Incident.findOneAndUpdate(
    {
        projectId: project._id,
        messageSignature,            ← must match
        status: { $in: ["OPEN", "ACKNOWLEDGED"] }  ← only active incidents
    },
    {
        $set: { lastOccurredAt: usedEventTimestamp },
        $inc: { eventCount: 1 }
    },
    { new: true }
)
```

- If an **existing active incident** matches → update it (bump `eventCount`, update `lastOccurredAt`) and emit via WebSocket.
- If **no existing incident** → create a new one:
  ```js
  Incident.create({
      projectId: project._id,
      service,
      severity,
      messageSignature,
      firstOccurredAt: usedEventTimestamp,
      lastOccurredAt: usedEventTimestamp,
      eventCount: 1
  })
  ```
- In both cases, emit `"incident-updated"` to the `project:<projectId>` Socket.IO room:
  ```js
  io.to(`project:${projectId}`).emit("incident-updated", incident.toObject())
  ```

**Step 8 — Emit the event itself via WebSocket:**
```js
emitEventToProject(io, project._id.toString(), event.toObject())
```
This broadcasts the new event to all connected frontend clients watching that project's room.

**Step 9 — Return `201`:**
```json
{
    "message": "Event ingested successfully",
    "eventId": "64abc..."
}
```

**Error responses:**

| Status | Condition |
|---|---|
| `201` | Success |
| `400` | Missing project, missing fields, or invalid timestamp |
| `500` | Unexpected DB error or Socket.IO failure |

---

### `getProjectEvents`

**Route:** `GET /api/events/:projectId`  
**Auth:** JWT (`authRequired` middleware)  
**Used by:** Dashboard UI to display event feed for a selected project

**Purpose:** Returns a paginated list of events for a specific project. Only the project owner can query this. Supports cursor-based pagination via a `before` timestamp.

**Request params:** `:projectId`

**Request query params:**

| Param | Type | Default | Max | Description |
|---|---|---|---|---|
| `limit` | Number | `50` | `200` | How many events to return |
| `before` | ISO Date String | _(none)_ | — | Return only events with `eventTimestamp < before` |

**Example request:**
```
GET /api/events/64abc123?limit=25&before=2026-09-15T12:00:00Z
Authorization: Bearer <accessToken>
```

**Step-by-step flow:**

1. **Validate `projectId` format:**
   ```js
   if (!mongoose.Types.ObjectId.isValid(projectId)) → 400
   ```

2. **Parse query params:**
   - `limit = Math.min(parseInt(req.query.limit) || 50, 200)` — capped at 200 to prevent memory issues.
   - `before = req.query.before` — optional cursor.

3. **Find project:** `Project.findById(projectId)` → `404` if not found.

4. **Authorization check:** `project.ownerId.toString() !== userId?.toString()` → `403` if the logged-in user doesn't own this project.

5. **Build query:**
   ```js
   const objectId = new mongoose.Types.ObjectId(projectId);
   const query = { projectId: objectId };
   if (before) {
       const beforeDate = new Date(before);
       if (isNaN(beforeDate)) → 400
       query.eventTimestamp = { $lt: beforeDate };
   }
   ```

6. **Fetch events:**
   ```js
   Event.find(query).sort({ eventTimestamp: -1 }).limit(limit).lean()
   ```
   - Sorted newest first.
   - `.lean()` returns plain JS objects instead of Mongoose documents (faster, less memory).

7. **Return:**
   ```json
   {
       "count": 25,
       "events": [ { ... }, { ... }, ... ]
   }
   ```

**How pagination works:**
- Initial request: no `before` param → returns latest `limit` events.
- Next page: take the `eventTimestamp` of the last event in the response and pass it as `before`.
- This is cursor-based pagination — more stable than offset-based under concurrent writes.

**Error responses:**

| Status | Condition |
|---|---|
| `200` | Events returned |
| `400` | Invalid projectId or invalid `before` param |
| `403` | Logged-in user is not the project owner |
| `404` | Project not found |
| `500` | DB error |

---

---

## 3. `incident.controller.js`

**File:** `src/controllers/incident.controller.js`  
**Route prefix:** `/api/incidents`  
**Imported by:** `src/routes/incident.Routes.js`

### Purpose

This controller manages the lifecycle of **Incidents** from the human operator's perspective. Incidents are auto-created by `event.controller.js` — this controller allows users to **view** them and **change their status** (acknowledge or resolve). Status changes are broadcast in real-time via WebSocket so that all dashboard users see the update immediately.

### Route Map for Incidents

| Method | Path | Middleware | Controller Function |
|---|---|---|---|
| GET | `/api/incidents/:projectId` | `authRequired` | `getProjectIncidents` |
| PATCH | `/api/incidents/:incidentId/status` | `authRequired` | `updateIncidentStatus` |

---

### `exports.getProjectIncidents`

**Route:** `GET /api/incidents/:projectId`  
**Auth:** JWT  

**Purpose:** Returns all incidents for a project in reverse chronological order (most recently active first). The caller must be the project owner.

**Request params:** `:projectId`

**Step-by-step flow:**

1. Validate `projectId` format → `400` if invalid ObjectId.
2. `Project.findById(projectId)` → `404` if not found.
3. `project.ownerId.toString() !== userId.toString()` → `403` if not the owner.
4. `Incident.find({ projectId }).sort({ lastOccurredAt: -1 }).lean()` — fetch all incidents sorted by last occurrence.
5. Return:
   ```json
   { "incidents": [ { ... }, { ... } ] }
   ```

> **Why are all incidents returned at once?** Incidents are expected to be significantly fewer in number than events. A typical project might have dozens of incidents but thousands of events. Pagination would add complexity for minimal benefit here.

**Error responses:**

| Status | Condition |
|---|---|
| `200` | `{ incidents: [...] }` |
| `400` | Invalid projectId |
| `403` | Not the project owner |
| `404` | Project not found |
| `500` | DB fetch error |

---

### `exports.updateIncidentStatus`

**Route:** `PATCH /api/incidents/:incidentId/status`  
**Auth:** JWT  

**Purpose:** Allows the project owner to move an incident through its lifecycle: `OPEN → ACKNOWLEDGED → RESOLVED`. The update is immediately broadcast via WebSocket to all connected users watching that project.

**Request params:** `:incidentId`

**Request body:**
```json
{ "status": "ACKNOWLEDGED" }
```
or
```json
{ "status": "RESOLVED" }
```

> **Note:** You cannot set status back to `"OPEN"` via this endpoint. Only `"ACKNOWLEDGED"` and `"RESOLVED"` are accepted. A re-opened incident (if the same error recurs) would create a new incident via `ingestEvent`.

**Step-by-step flow:**

1. Validate `incidentId` format → `400` if invalid.
2. Validate `status` value → `400` if not in `["ACKNOWLEDGED", "RESOLVED"]`.
3. Find incident and populate its `projectId` field:
   ```js
   Incident.findById(incidentId).populate("projectId")
   ```
   → `404` if not found. After `.populate()`, `incident.projectId` is the full `Project` document.

4. Authorization check: `project.ownerId.toString() !== userId.toString()` → `403`.

5. Update and save:
   ```js
   incident.status = status;
   await incident.save();
   ```

6. Prepare the object for WebSocket emission:
   - Call `incident.toObject()` to get a plain JS object.
   - Ensure `projectId` in the emitted object is properly shaped (contains `_id`).

7. Broadcast via WebSocket:
   ```js
   const io = getIO();
   io.to(`project:${project._id}`).emit("incident-updated", incidentForEmit);
   ```
   > **Important:** The Socket.IO `emit` call is wrapped in its own try/catch. Socket errors do NOT cause the HTTP response to fail — the DB update and HTTP `200` response are returned regardless of whether the WebSocket emit succeeds.

8. Return `200`:
   ```json
   {
       "message": "Incident updated successfully",
       "incident": { ... }
   }
   ```

**Error responses:**

| Status | Condition |
|---|---|
| `200` | Incident updated, response includes updated incident object |
| `400` | Invalid `incidentId` format or invalid `status` value |
| `403` | Authenticated user is not the project owner |
| `404` | Incident not found |
| `500` | Save error or unexpected failure |

---

---

## 4. `project.Controller.js`

**File:** `src/controllers/project.Controller.js`  
**Route prefix:** `/api/project`  
**Imported by:** `src/routes/project.Routes.js`

### Purpose

This controller manages the **Project** resource — the top-level container for all events and incidents. Every monitoring target (service, app, server) that a user wants to track must be represented as a project. When a project is created, a cryptographically secure **ingest API key** is generated and returned once. This key is what external services use to authenticate when sending events.

### Route Map for Projects

| Method | Path | Middleware | Controller Function |
|---|---|---|---|
| POST | `/api/project/create` | `authRequired` → `ValidateProject` | `createProject` |
| GET | `/api/project/list` | `authRequired` → `ValidateProject` | `listProject` |
| POST | `/api/project/:projectId/rotate-key` | `authRequired` | `rotateIngestKey` |

---

### `exports.createProject`

**Route:** `POST /api/project/create`  
**Auth:** JWT  

**Purpose:** Creates a new project for the authenticated user and returns a one-time plain-text ingest API key.

**Request body:**
```json
{
  "projectName": "Payment Service",
  "description": "Monitors our payment processing microservice"
}
```

**Step-by-step flow:**

1. **Duplicate check:**
   ```js
   Project.findOne({ projectName, ownerId })
   ```
   → `409 Conflict` if a project with the same name already exists for this user.
   > Two different users can have projects with the same name; only the combination of `projectName + ownerId` must be unique.

2. **Instantiate (but don't save yet):**
   ```js
   const project = new Project({ projectName, description, ownerId });
   ```
   The project is created in memory but not written to DB yet.

3. **Generate API key:**
   ```js
   const apiKey = project.generateIngestKey();
   ```
   Internally this:
   - Generates 32 random bytes → 64-char hex string (the raw key).
   - SHA-256 hashes it → stores hash on `project.ingestKeyHash`.
   - Returns the raw key as `apiKey`.

   At this point, `project.ingestKeyHash` is set but `apiKey` is only in local memory.

4. **Save to DB:**
   ```js
   await project.save();
   ```
   Now `ingestKeyHash` is persisted. The raw `apiKey` is **never** saved to DB.

5. **Return `200`:**
   ```json
   {
     "message": "Project Created Successfully",
     "project": {
       "_id": "64abc...",
       "projectName": "Payment Service",
       "description": "Monitors our payment processing microservice",
       "createdAt": "2026-09-15T..."
     },
     "ingestKey": "7f3a9c...",
     "note": "Store this Api key securely. It will not be shown again."
   }
   ```

> ⚠️ **This is the ONLY time the plain ingest key is ever accessible.** Once this response is sent, the raw key is gone forever. If lost, the user must rotate the key.

**Error responses:**

| Status | Condition |
|---|---|
| `200` | Project created, API key returned once |
| `409` | Duplicate project name for this owner |
| `500` | DB error |

---

### `exports.listProject`

**Route:** `GET /api/project/list`  
**Auth:** JWT  

**Purpose:** Returns all projects belonging to the authenticated user, sorted newest-first.

**Step-by-step flow:**

1. `Project.find({ ownerId }).sort({ createdAt: -1 })` — fetch all owned projects.
2. Return:
   ```json
   { "Projects": [ { ... }, { ... } ] }
   ```

> **Note:** `ingestKeyHash` has `select: false` in the model schema, so it is **never** returned in this list (or any normal query unless explicitly selected with `.select("+ingestKeyHash")`).

**Error responses:**

| Status | Condition |
|---|---|
| `200` | `{ Projects: [...] }` |
| `500` | DB error |

---

### `exports.rotateIngestKey`

**Route:** `POST /api/project/:projectId/rotate-key`  
**Auth:** JWT  

**Purpose:** Replaces the project's existing ingest API key with a new one. Used when a key is suspected to be compromised, or as regular security rotation. The old key stops working immediately after this call. An `AuditLog` entry is created to record the rotation event.

**Request params:** `:projectId`

**Step-by-step flow:**

1. Validate `projectId` is present → `400` if missing.

2. Find project, explicitly selecting the hidden field:
   ```js
   Project.findById(projectId).select("+ingestKeyHash ownerId")
   ```
   → `404` if not found.

3. Ownership check: `project.ownerId !== userId` → `403` if not the owner.

4. Generate and store a new key:
   ```js
   const newApiKey = project.generateIngestKey();
   ```
   This overwrites `project.ingestKeyHash` with a new hash.

5. Save the project (old hash is now overwritten in DB).

6. Attempt to create an `AuditLog` record:
   ```js
   AuditLog.create({
       purpose: "KEY_ROTATED",
       projectId,
       userId,
       message: "API key rotated successfully"
   })
   ```
   > **Important:** This is wrapped in its own try/catch. If the audit log creation fails (e.g., DB constraint issue — `ipAddress` is required in the schema but not provided here), the error is logged to console but does NOT fail the overall response. The key rotation itself succeeds regardless.

7. Return `200`:
   ```json
   {
     "message": "API key rotated successfully",
     "ingestKey": "a9f02c...",
     "note": "Store this API key securely. It will not be shown again."
   }
   ```

**Error responses:**

| Status | Condition |
|---|---|
| `200` | New key returned (one time only) |
| `400` | `projectId` not provided |
| `403` | Not the project owner |
| `404` | Project not found |
| `500` | Rotation/save error |

---

---

## 5. `system.controller.js`

**File:** `src/controllers/system.controller.js`  
**Route prefix:** `/api/system`  
**Imported by:** `src/routes/system.routes.js`

### Purpose

Provides internal observability endpoints. These endpoints do not require authentication (public by default based on current routes), but they expose detailed server internals. They are intended for internal monitoring tools, dashboards, or uptime monitors.

### Route Map for System

| Method | Path | Middleware | Controller Function |
|---|---|---|---|
| GET | `/api/system/health` | _(none)_ | `getHealth` |
| GET | `/api/system/metrics` | _(none)_ | `getMetrics` |

---

### Internal Helper: `formatMemoryUsage(memObj)`

Converts the raw byte values from `process.memoryUsage()` into human-readable MB strings.

**Input:** the object returned by `process.memoryUsage()`.
**Output:**
```json
{
  "rss": "45.23 MB",
  "heapTotal": "30.00 MB",
  "heapUsed": "22.41 MB",
  "external": "1.12 MB",
  "arrayBuffers": "0.05 MB"
}
```

| Memory Field | Meaning |
|---|---|
| `rss` | Resident Set Size — total memory allocated for the Node.js process by the OS |
| `heapTotal` | Total size of the V8 heap allocated |
| `heapUsed` | Portion of the heap currently in use (actual JS objects) |
| `external` | Memory used by C++ objects bound to JS (e.g. Buffers) |
| `arrayBuffers` | Memory allocated for `ArrayBuffer` and `SharedArrayBuffer` |

---

### `exports.getHealth`

**Route:** `GET /api/system/health`  
**Auth:** None

**Purpose:** Returns a comprehensive snapshot of the server's health at the moment of the request. Useful for uptime monitoring and diagnosing issues.

**Response payload (full example):**
```json
{
  "status": "OK",
  "uptimeSeconds": 3600.42,
  "memory": {
    "rss": "51.20 MB",
    "heapTotal": "32.00 MB",
    "heapUsed": "25.13 MB",
    "external": "1.23 MB",
    "arrayBuffers": "0.07 MB"
  },
  "cpuLoad": {
    "1min": 0.12,
    "5min": 0.08,
    "15min": 0.05
  },
  "platform": "linux",
  "arch": "x64",
  "cpuCores": 4,
  "nodeVersion": "v20.11.0",
  "mongo": {
    "connectionState": 1,
    "host": "localhost",
    "name": "apilogs"
  },
  "activeSocketConnections": 12,
  "timestamp": "2026-09-15T15:48:00.000Z"
}
```

**Field-by-field breakdown:**

| Field | Source | Notes |
|---|---|---|
| `status` | Hardcoded `"OK"` | If the handler runs at all, status is OK |
| `uptimeSeconds` | `process.uptime()` | Seconds since npm start |
| `memory` | `process.memoryUsage()` | Formatted via `formatMemoryUsage()` |
| `cpuLoad["1min"]` | `os.loadavg()[0]` | 1-minute CPU load average (Linux/macOS) |
| `cpuLoad["5min"]` | `os.loadavg()[1]` | 5-minute CPU load average |
| `cpuLoad["15min"]` | `os.loadavg()[2]` | 15-minute CPU load average |
| `platform` | `os.platform()` | e.g. `"linux"`, `"win32"`, `"darwin"` |
| `arch` | `os.arch()` | e.g. `"x64"`, `"arm64"` |
| `cpuCores` | `os.cpus().length` | Logical CPU core count |
| `nodeVersion` | `process.version` | Node.js version string |
| `mongo.connectionState` | `mongoose.connection.readyState` | `0`=disconnected, `1`=connected, `2`=connecting, `3`=disconnecting |
| `mongo.host` | `mongoose.connection.host` | DB host being used |
| `mongo.name` | `mongoose.connection.name` | DB name |
| `activeSocketConnections` | `metrics.getActiveSocketConnections()` or `metrics.activeSocketConnections` | Live WebSocket count from in-memory metrics |
| `timestamp` | `new Date().toISOString()` | When this health check was generated |

**Error responses:**

| Status | Condition |
|---|---|
| `200` | Full health object returned |
| `500` | Unexpected error while building the health object |

---

### `exports.getMetrics`

**Route:** `GET /api/system/metrics`  
**Auth:** None

**Purpose:** Returns a snapshot of application-level performance counters from the in-memory `MetricsStore`. Useful for monitoring dashboards and alerting on ingest rates or security events.

**Step-by-step flow:**

1. Check that `metrics.getSnapshot` is a function (defensive guard against misconfiguration).
2. Call `metrics.getSnapshot()`.
3. Return the result directly.

**Response payload (full example):**
```json
{
  "totalEventsIngested": 4821,
  "eventsLastMinute": 12,
  "failedApiKeyAttempts": 3,
  "rateLimitHits": 7,
  "activeSocketConnections": 12
}
```

| Field | Type | Meaning |
|---|---|---|
| `totalEventsIngested` | Number | Total events ingested since server start |
| `eventsLastMinute` | Number | Events in the current 60-second rolling window. Resets every minute |
| `failedApiKeyAttempts` | Number | Total `apiKeyAuth` middleware failures (unexpected errors, not wrong keys) |
| `rateLimitHits` | Number | Number of requests rejected by rate limiters |
| `activeSocketConnections` | Number | Current live WebSocket connections |

> ⚠️ **All counters are in-memory only.** They reset to `0` every time the server restarts. They are not stored in the database.

**Error responses:**

| Status | Condition |
|---|---|
| `200` | Metrics snapshot returned |
| `500` | `metrics` is not defined or `getSnapshot` is not a function |

---

---

## Summary Reference Tables

### All API Routes With Full Context

| Method | Full URL | Auth Type | Middleware Chain | Controller | Function |
|---|---|---|---|---|---|
| POST | `/api/auth/register` | None | validateRegister | auth.Controller | `register` |
| POST | `/api/auth/verify/resend` | None | otpLimiter | auth.Controller | `resendVerificationOtp` |
| POST | `/api/auth/verify/confirm` | None | otpLimiter | auth.Controller | `verifyEmail` |
| POST | `/api/auth/login` | None | validateLogin | auth.Controller | `loginWithPassword` |
| POST | `/api/auth/login/otp/request` | None | otpLimiter | auth.Controller | `requestLoginOtp` |
| POST | `/api/auth/login/otp/verify` | None | otpLimiter | auth.Controller | `verifyLoginOtp` |
| POST | `/api/auth/token/refresh` | Refresh Token | None | auth.Controller | `refreshToken` |
| POST | `/api/auth/logout-everywhere` | JWT | authRequired | auth.Controller | `logoutEverywhere` |
| POST | `/api/events/ingest/:projectId` | API Key | apiKeyAuth → projectRateLimiter → validateEvent | event.controller | `ingestEvent` |
| GET | `/api/events/:projectId` | JWT | authRequired | event.controller | `getProjectEvents` |
| GET | `/api/incidents/:projectId` | JWT | authRequired | incident.controller | `getProjectIncidents` |
| PATCH | `/api/incidents/:incidentId/status` | JWT | authRequired | incident.controller | `updateIncidentStatus` |
| POST | `/api/project/create` | JWT | authRequired → ValidateProject | project.Controller | `createProject` |
| GET | `/api/project/list` | JWT | authRequired → ValidateProject | project.Controller | `listProject` |
| POST | `/api/project/:projectId/rotate-key` | JWT | authRequired | project.Controller | `rotateIngestKey` |
| GET | `/api/system/health` | None | None | system.controller | `getHealth` |
| GET | `/api/system/metrics` | None | None | system.controller | `getMetrics` |

---

### Models Touched by Each Controller

| Controller | Models Read | Models Written | Models Updated |
|---|---|---|---|
| `auth.Controller` | User, OtpToken | User, OtpToken | User (`isVerified`, `tokenVersion`), OtpToken (`consumed`, `attempts`) |
| `event.controller` | Project, Incident | Event, Incident | Incident (`eventCount`, `lastOccurredAt`) |
| `incident.controller` | Incident, Project | — | Incident (`status`) |
| `project.Controller` | Project | Project, AuditLog | Project (`ingestKeyHash`) |
| `system.controller` | — (uses `mongoose.connection`) | — | — |

---

### Real-time WebSocket Events Emitted by Controllers

| Controller Function | Socket.IO Event Name | Room | Payload |
|---|---|---|---|
| `ingestEvent` (ERROR/CRITICAL) | `"incident-updated"` | `project:<projectId>` | Full incident object |
| `ingestEvent` (always) | _(via emitEventToProject)_ | `project:<projectId>` | Full event object |
| `updateIncidentStatus` | `"incident-updated"` | `project:<projectId>` | Updated incident object |

---

### Security Design Summary

| Concern | How It's Handled |
|---|---|
| Password storage | bcrypt hashed via Mongoose pre-save hook. Never stored in plaintext |
| OTP storage | bcrypt hashed. Plain OTP is only in memory and in the email, never in DB |
| Ingest API key storage | SHA-256 hashed. Plain key returned only at creation/rotation time |
| JWT invalidation | `tokenVersion` in token payload + increment on `logoutEverywhere` |
| OTP brute force | Attempt counter on `OtpToken`. Locked at `OTP_MAX_ATTEMPTS` |
| OTP replay | `consumed` flag set to `true` after use. Single-use per token |
| API key failure logging | `AuditLog` record created on wrong key attempt with caller IP |
| Credential enumeration | `loginWithPassword` returns same `400` for "email not found" and "wrong password" |
| Unauthorized data access | All data-returning endpoints verify `project.ownerId === req.user.id` before responding |
