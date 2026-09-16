# APILogs Backend — Validation Layer Documentation

> **Source of truth:** All claims in this document are verified directly from the source code in `backend/src/validations/`. Do not assume any behavior; read the code.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Validation Library — Joi](#2-validation-library--joi)
3. [File Index](#3-file-index)
4. [auth.Validation.js — Authentication Schemas](#4-authvalidationjs--authentication-schemas)
5. [Event.validation.js — Event Ingestion Schemas](#5-eventvalidationjs--event-ingestion-schemas)
6. [project.validation.js — Project Schemas](#6-projectvalidationjs--project-schemas)
7. [Route ↔ Validator Wiring Map](#7-route--validator-wiring-map)
8. [Validation Middleware Execution Model](#8-validation-middleware-execution-model)
9. [Error Response Format](#9-error-response-format)
10. [Cross-File Rules and Consistency Analysis](#10-cross-file-rules-and-consistency-analysis)
11. [Model vs. Validation Schema Comparison](#11-model-vs-validation-schema-comparison)
12. [Security Notes](#12-security-notes)
13. [Documentation Discrepancies](#13-documentation-discrepancies)
14. [Unverified Assumptions](#14-unverified-assumptions)
15. [Interview Preparation Notes](#15-interview-preparation-notes)

---

## 1. Architecture Overview

The `validations/` folder contains **request-level input validation** using the [Joi](https://joi.dev/) schema-description library. Every validator in this folder is a plain Express middleware function that:

1. Receives `req` (and optional `req.params`).
2. Runs a Joi `.validate()` call with `{ abortEarly: false }` so **all** field errors are collected before responding.
3. If invalid → responds immediately with `HTTP 400` and an `errors` array; the controller is **never reached**.
4. If valid → calls `next()` to pass control to the next middleware or controller.

```
Client Request
     │
     ▼
Express Router
     │
     ▼
[ Optional Rate-Limiter ]       ─── otpLimiter / projectRateLimiter / apiKeyAuth
     │
     ▼
[ Joi Validation Middleware ]   ─── validateRegister / validateLogin / validateEvent / ValidateProject
     │── 400 + errors[] if invalid
     │
     ▼
[ Controller ]                  ─── auth.Controller, event.controller, project.Controller
     │
     ▼
[ Database / Side Effects ]
     │
     ▼
HTTP Response
```

Validations serve as the **gate before the database layer**. They prevent malformed data from ever reaching controllers or Mongoose models.

---

## 2. Validation Library — Joi

| Property | Value |
|---|---|
| Library | `joi` (CommonJS `require("joi")`) |
| Strategy | Schema-first: define schema once, derive middleware from it |
| `abortEarly` | `false` in all three files — all field errors are collected and returned together |
| Custom messages | All schemas define `.messages({})` per rule to produce user-friendly, fieldspecific errors |
| Strip unknown | Not configured — unknown fields are **not stripped**, they pass through to `req.body` as-is |
| Conversion | Joi's default coerce-on-validate is active. Dates passed as strings are coerced to `Date` objects (relevant for `eventTimestamp`) |

---

## 3. File Index

| File | Exports | Route file that imports it |
|---|---|---|
| `auth.Validation.js` | `validateRegister`, `validateLogin`, `registerSchema`, `loginSchema` | `routes/auth.Routes.js` |
| `Event.validation.js` | `validateEvent`, `eventBodySchema`, `eventParamsSchema` | `routes/event.Routes.js` |
| `project.validation.js` | `ValidateProject`, `ProjectSchema` | `routes/project.Routes.js` |

---

## 4. `auth.Validation.js` — Authentication Schemas

### 4.1 Purpose

Validate the request bodies for user **registration** and **password login** before the auth controller runs.

### 4.2 Dependencies

```js
const Joi = require("joi");
```

Only Joi. No project-internal imports.

### 4.3 Exported Schemas

#### `registerSchema`

Validates `req.body` for `POST /auth/register`.

| Field | Type | Required | Constraints | Custom Error Key |
|---|---|---|---|---|
| `username` | string | ✅ Yes | `alphanum()`, `trim()`, min 3, max 100 | `string.base`, `string.empty`, `string.min`, `string.max`, `any.required` |
| `email` | string | ✅ Yes | Valid email format, `tlds: { allow: false }` | `string.base`, `string.email`, `string.empty`, `any.required` |
| `password` | string | ✅ Yes | min 8, max 500, regex complexity rule | `string.min`, `string.max`, `string.pattern.base`, `string.empty`, `any.required` |

**Password regex (line 31):**
```
^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[!@#$%^&*()_+\-=\[\]{};':"\\|,.<>\/?]).+$
```
Requires at minimum:
- One lowercase letter `[a-z]`
- One uppercase letter `[A-Z]`
- One digit `\d`
- One special character from: `!@#$%^&*()_+-=[]{}; ':"\\|,.<>/?`

> **Note on error message vs. constraint (important discrepancy):** The schema defines `max(500)` on `password`, but the custom error message reads `"Password must be at most 50 characters"`. The **enforced limit is 500 characters**; the error message is misleading. See [Section 13](#13-documentation-discrepancies).

**`tlds: { allow: false }` explained:** Joi's email validator normally rejects TLDs it does not recognise. Setting `allow: false` disables TLD list checking, which means synthetically valid TLDs like `.test` are accepted. This is intentional for development/testing flexibility.

#### `loginSchema`

Validates `req.body` for `POST /auth/login`.

| Field | Type | Required | Constraints |
|---|---|---|---|
| `email` | string | ✅ Yes | Valid email, `tlds: { allow: false }` |
| `password` | string | ✅ Yes | Non-empty only — no length or regex constraint |

> **Key difference from registerSchema:** The login password field has **no `min`, `max`, or pattern rules**. This is intentional — the schema just checks that a password was provided; the actual credential check happens in the controller via `bcrypt.compare`.

### 4.4 Exported Middleware

#### `validateRegister(req, res, next)`

```js
const validateRegister = (req, res, next) => {
    const { error } = registerSchema.validate(req.body, { abortEarly: false });
    if (error) {
        return res.status(400).json({ errors: error.details.map(e => e.message) });
    }
    next();
};
```

- Validates `req.body` against `registerSchema`.
- On failure: `HTTP 400`, body `{ errors: string[] }`.
- On success: calls `next()`.

#### `validateLogin(req, res, next)`

```js
const validateLogin = (req, res, next) => {
    const { error } = loginSchema.validate(req.body, { abortEarly: false });
    if (error) {
        return res.status(400).json({ errors: error.details.map(e => e.message) });
    }
    next();
};
```

Identical execution model to `validateRegister`, using `loginSchema`.

### 4.5 Middleware Placement in Routes

```
POST /auth/register   → [validateRegister] → ctrl.register
POST /auth/login      → [validateLogin]    → ctrl.loginWithPassword
```

All other auth routes (`/verify/*`, `/login/otp/*`, `/token/refresh`, `/logout-everywhere`) do **not** pass through any Joi validation middleware. They rely on controller-level checks or other middleware (e.g., `authRequired`, `otpLimiter`).

---

## 5. `Event.validation.js` — Event Ingestion Schemas

### 5.1 Purpose

Validate **both** the URL parameter (`projectId`) and the request body for the event ingestion endpoint. This is the only validator in the project that validates `req.params`.

### 5.2 Dependencies

```js
const Joi = require("joi");
const SEVERITY_LEVELS = ["INFO", "WARN", "ERROR", "CRITICAL"];
```

`SEVERITY_LEVELS` is a local constant (not imported from the model). It must stay in sync manually with the `Event` model's enum definition.

### 5.3 Exported Schemas

#### `eventParamsSchema`

Validates `req.params` for the `ingest` route.

| Field | Type | Required | Constraints |
|---|---|---|---|
| `projectId` | string | ✅ Yes | `hex()` (only hex chars), exact `length(24)` |

The `length(24)` + `hex()` combination enforces a structurally valid MongoDB ObjectId string. This prevents the event controller from receiving a malformed ObjectId that would cause a Mongoose `CastError`.

#### `eventBodySchema`

Validates `req.body` for `POST /events/ingest/:projectId`.

| Field | Type | Required | Default | Constraints |
|---|---|---|---|---|
| `service` | string | ✅ Yes | — | `trim()`, min 2, max 100 |
| `severity` | string | ✅ Yes | — | `valid("INFO","WARN","ERROR","CRITICAL")` — exact enum match, case-sensitive |
| `message` | string | ✅ Yes | — | `trim()`, min 1, max 2000 |
| `metadata` | object | ❌ No | `{}` (set by model) | `unknown(true)` — any keys accepted |
| `environment` | string | ❌ No | `"production"` (set by model) | `valid("development","staging","production")` |
| `eventTimestamp` | date | ❌ No | `Date.now` (set by model) | Coerced from ISO string to JS `Date` |

**`metadata: Joi.object().unknown(true)`** — This means any JSON object is accepted as metadata, with any nested keys. There is no depth or size limit enforced at the Joi layer.

**`severity` is case-sensitive:** Sending `"error"` instead of `"ERROR"` will fail validation with the message: `"Severity must be one of INFO, WARN, ERROR, CRITICAL"`.

**`eventTimestamp` coercion:** Joi parses the string into a `Date` object before `next()` is called. The controller then receives a JS `Date` in `req.body.eventTimestamp`.

### 5.4 Exported Middleware

#### `validateEvent(req, res, next)`

```js
const validateEvent = (req, res, next) => {
    // Step 1: Validate URL params
    const { error: paramsError } = eventParamsSchema.validate(req.params, { abortEarly: false });
    if (paramsError) {
        return res.status(400).json({ errors: paramsError.details.map(e => e.message) });
    }

    // Step 2: Validate request body
    const { error: bodyError } = eventBodySchema.validate(req.body, { abortEarly: false });
    if (bodyError) {
        return res.status(400).json({ errors: bodyError.details.map(e => e.message) });
    }

    next();
};
```

**Two-phase validation:** Params are validated first. If params fail, the body is never validated (early `return`). This is a minor design detail — in practice `projectId` format errors are always caught before body errors.

### 5.5 Middleware Placement in Routes

```
POST /events/ingest/:projectId
  → [apiKeyAuth]           (verify API key)
  → [projectRateLimiter]   (per-project rate limiting)
  → [validateEvent]        (validate params + body)  ← this file
  → ctrl.ingestEvent

GET  /events/:projectId
  → [authRequired]         (JWT session auth)
  → ctrl.getProjectEvents  (no validation middleware — projectId validated by Mongoose CastError handling in controller)
```

> **Note:** `GET /:projectId` does **not** use `validateEvent`. The `projectId` in the read path is not validated via Joi. The controller or Mongoose handles the invalid ObjectId case.

---

## 6. `project.validation.js` — Project Schemas

### 6.1 Purpose

Validate the request body when creating a new project.

### 6.2 Dependencies

```js
const Joi = require("joi");
```

No project-internal imports.

### 6.3 Exported Schemas

#### `ProjectSchema`

Validates `req.body` for `POST /projects/create`.

| Field | Type | Required | Default | Constraints |
|---|---|---|---|---|
| `projectName` | string | ✅ Yes | — | `trim()`, min 3, max 200 |
| `description` | string | ❌ No | `""` (set by model) | `allow("")`, `trim()`, max 500 |

**`description: Joi.string().allow("")`** — Allows the empty string, which means omitting the description or sending `""` are both valid. Without `.allow("")`, Joi would reject an empty string as `string.empty`.

### 6.4 Exported Middleware

#### `ValidateProject(req, res, next)`

```js
const ValidateProject = (req, res, next) => {
    const { error } = ProjectSchema.validate(req.body, { abortEarly: false });
    if (error) {
        return res.status(400).json({ errors: error.details.map(e => e.message) });
    }
    next();
};
```

**Bug found — middleware on GET route:**

From `routes/project.Routes.js`:

```js
router.post("/create",  authRequired, ValidateProject, ctrl.createProject);
router.get("/list",     authRequired, ValidateProject, ctrl.listProject);   // ← BUG
router.post("/:projectId/rotate-key", authRequired, ctrl.rotateIngestKey);
```

`ValidateProject` is applied to `GET /projects/list`. This is almost certainly a mistake: `GET` requests have no body, so `req.body` will be `{}` or `undefined`. Since both `projectName` is `required` and `description` is optional, this means `GET /projects/list` will **always fail validation** with a 400 error unless `projectName` happens to be in the request body (which is not standard for GET). This effectively makes the list endpoint broken/unreachable. See [Section 13](#13-documentation-discrepancies).

---

## 7. Route ↔ Validator Wiring Map

| HTTP Method | Path | Validation Middleware | Validates |
|---|---|---|---|
| `POST` | `/auth/register` | `validateRegister` | `req.body` |
| `POST` | `/auth/login` | `validateLogin` | `req.body` |
| `POST` | `/auth/verify/resend` | *(none — rate-limited only)* | — |
| `POST` | `/auth/verify/confirm` | *(none — rate-limited only)* | — |
| `POST` | `/auth/login/otp/request` | *(none — rate-limited only)* | — |
| `POST` | `/auth/login/otp/verify` | *(none — rate-limited only)* | — |
| `POST` | `/auth/token/refresh` | *(none)* | — |
| `POST` | `/auth/logout-everywhere` | *(none — `authRequired` only)* | — |
| `POST` | `/events/ingest/:projectId` | `validateEvent` | `req.params` + `req.body` |
| `GET` | `/events/:projectId` | *(none)* | — |
| `POST` | `/projects/create` | `ValidateProject` | `req.body` |
| `GET` | `/projects/list` | `ValidateProject` ⚠️ **BUG** | `req.body` (always empty for GET) |
| `POST` | `/projects/:projectId/rotate-key` | *(none)* | — |

---

## 8. Validation Middleware Execution Model

### Execution Order

All three middleware functions follow the exact same pattern:

```
1. Call Joi schema.validate(input, { abortEarly: false })
2. Destructure `error` from result
3. if (error):
     → map error.details to message strings
     → return res.status(400).json({ errors: [...] })
     (stops the middleware chain — controller never runs)
4. else:
     → call next()
     (passes control to the next middleware or controller)
```

### `abortEarly: false` — What It Means

By default, Joi stops after the first failing field. Setting `abortEarly: false` forces Joi to validate **all** fields before returning. The result is that the client receives all validation errors in a single response instead of one at a time. This is the correct production behaviour.

### No Input Sanitisation

The validators do **not** strip unknown keys (no `stripUnknown: true`). Any extra fields sent by the client will pass through to `req.body` and reach the controller. The controllers and Mongoose models are responsible for ignoring those fields.

The only sanitisation that does occur:
- `.trim()` — Joi trims whitespace before validation, so `"  hello  "` passes as `"hello"`. The trimmed value is what Joi returns in `value`, but by default the validators do **not** assign the Joi `value` back to `req.body`. This means the original untrimmed value remains in `req.body` unless the controller re-reads from `value`. **This should be verified per controller.**

---

## 9. Error Response Format

All three validation middleware files return the same error shape on failure:

```json
HTTP 400 Bad Request
Content-Type: application/json

{
  "errors": [
    "Username must be at least 3 characters",
    "Password must contain at least one uppercase letter, one lowercase letter, one digit, and one special character"
  ]
}
```

- Key: `"errors"` (array of strings).
- Each string is the custom `.messages()` value for the failing rule.
- Multiple errors can appear in a single response (because `abortEarly: false`).
- No HTTP 422 (Unprocessable Entity) is used — all validation failures use 400.

---

## 10. Cross-File Rules and Consistency Analysis

### Severity Enum

`Event.validation.js` defines:
```js
const SEVERITY_LEVELS = ["INFO", "WARN", "ERROR", "CRITICAL"];
```

The `Event` Mongoose model (`models/Event.js`) defines:
```js
enum: ["INFO", "WARN", "ERROR", "CRITICAL"]
```

These are **in sync** as of the current codebase. However, the severity levels are **not shared from a single source of truth**. If the model enum changes, the validation file must be updated manually.

### Max Field Lengths

| Field | Joi Validation | Mongoose Model | In Sync? |
|---|---|---|---|
| `username` min | 3 | 3 | ✅ |
| `username` max | 100 | 100 | ✅ |
| `password` max (validation) | 500 | *(no max in model — stored as hash)* | N/A |
| `password` max (error message) | Says "50" | — | ❌ Bug |
| `projectName` min | 3 | 3 | ✅ |
| `projectName` max | 200 | 200 | ✅ |
| `description` max | 500 | 500 | ✅ |
| `service` min | 2 | *(no min in model)* | Diverge |
| `service` max | 100 | 100 | ✅ |
| `message` max | 2000 | 2000 | ✅ |

### Environment Enum

`Event.validation.js`: `valid("development","staging","production")` — all lowercase.  
`Event` model: `enum: ["development","staging","production"]` — ✅ in sync.

---

## 11. Model vs. Validation Schema Comparison

### User Registration

| Field | Joi (auth.Validation.js) | Mongoose (Users.js) |
|---|---|---|
| `username` | alphanum, 3–100, trim | String, unique, trim, 3–100 |
| `email` | email format, required | String, unique, trim, lowercase |
| `password` | min 8, max 500, complexity regex | String, minlength 3 (stored as hash) |

> The model has `password minlength: 3`, which refers to the hash string. The actual plaintext password minimum is enforced by Joi at 8 characters. There is no conflict.

### Project Creation

| Field | Joi (project.validation.js) | Mongoose (Project.js) |
|---|---|---|
| `projectName` | required, trim, 3–200 | required, trim, 3–200 |
| `description` | optional, allow(""), trim, max 500 | optional, trim, max 500, default `""` |
| `ownerId` | *(not in Joi schema)* | required ObjectId — set by controller from `req.user._id` |
| `ingestKeyHash` | *(not in Joi schema)* | required — generated by `generateIngestKey()` |

`ownerId` and `ingestKeyHash` are server-generated values and are correctly **not exposed** in the validation schema.

### Event Ingestion

| Field | Joi (Event.validation.js) | Mongoose (Event.js) |
|---|---|---|
| `service` | required, trim, 2–100 | required, trim, max 100 *(no min)* |
| `severity` | required, valid enum | required, enum |
| `message` | required, trim, 1–2000 | required, trim, max 2000 |
| `metadata` | optional, object, unknown | Mixed, default `{}` |
| `environment` | optional, valid enum | optional, enum, default `"production"` |
| `eventTimestamp` | optional, date | Date, default `Date.now` |
| `projectId` | in `eventParamsSchema` (req.params) | required ObjectId — set from `req.params.projectId` |

---

## 12. Security Notes

### What Validations Protect Against

| Threat | Protection |
|---|---|
| Oversized payloads per field | `max()` constraints on all string fields |
| Missing required fields | `required()` on critical fields |
| Invalid enum injection | `valid(...)` on `severity` and `environment` |
| Malformed ObjectId in URL | `hex().length(24)` in `eventParamsSchema` |
| Weak passwords | Regex complexity rule in `registerSchema` |
| Invalid email format | `.email()` on registration and login |

### What Validations Do NOT Protect Against

| Gap | Notes |
|---|---|
| Body size limit | No `express.json({ limit: ... })` configured in this layer. Body size limits must be set in the Express app config. |
| Unknown field stripping | `stripUnknown` is not set — extra fields pass through |
| XSS / HTML injection | No sanitisation of string content (e.g., `<script>` in `message` or `service` would pass Joi validation) |
| SQL/NoSQL injection | Joi validates structure, not content. Mongoose parameterized queries provide protection at the DB layer |
| Metadata depth/size | `metadata` is `Joi.object().unknown(true)` — a very large or deeply nested object would pass |
| Duplicate email/username | Not checked at validation layer — only MongoDB unique index catches this |
| Rate limiting on register/login | No `express-rate-limit` applied to `POST /auth/register` or `POST /auth/login` endpoints directly (only OTP routes use `otpLimiter`) |

---

## 13. Documentation Discrepancies

| # | Location | Issue |
|---|---|---|
| D1 | `auth.Validation.js` line 35 | `password` custom error message says **"at most 50 characters"** but `max(500)` is the actual enforced limit. The error message is wrong — clients are told 50 when the real limit is 500. |
| D2 | `project.Routes.js` line 9 | `ValidateProject` is applied to `GET /projects/list`. A GET request has no body. `projectName` is `required()` in the schema, so this middleware will **always return HTTP 400** for the list endpoint, making it unreachable unless the client sends `projectName` in the body of a GET request (non-standard). |
| D3 | `Event.validation.js` | `SEVERITY_LEVELS` is a local constant not imported from the model. Any future model change (e.g., adding `"DEBUG"`) will not be reflected in the validation automatically. |
| D4 | `event.Routes.js` | `GET /events/:projectId` does not use `validateEvent`. The `:projectId` in the read path is not Joi-validated. Invalid ObjectId strings reaching the controller may cause Mongoose `CastError` exceptions instead of clean 400 responses. |

---

## 14. Unverified Assumptions

| Assumption | Reason not verifiable from validation files alone |
|---|---|
| Joi `value` (trimmed output) is used by controllers | The validators call `next()` without assigning `value` back to `req.body`. Whether controllers use the raw `req.body` or Joi's trimmed `value` cannot be determined from this folder alone. |
| `metadata` max depth/size is bounded elsewhere | No evidence of a middleware enforcing a size cap on nested objects. |
| Global body size limit exists | Not set in the validation files. Must be verified in `app.js` / `server.js`. |
| Rate limiting on login/register | Only `otpLimiter` is clearly applied. Whether a separate rate limiter is applied to `/auth/login` or `/auth/register` at the app or reverse-proxy level is not verifiable from this folder. |

---

## 15. Interview Preparation Notes

### Q: What library is used for validation and why?

**Joi** is used because it provides declarative, schema-first validation with chainable rules, built-in type coercion, custom error messages per rule, and `abortEarly: false` support for collecting all errors at once. It decouples validation logic from controller business logic.

### Q: How does `abortEarly: false` affect the API consumer experience?

Without it, the client would receive one error at a time and need to submit the form repeatedly to discover all problems. With `abortEarly: false`, all validation errors are returned in a single `errors` array in one response, allowing the UI to display all field errors simultaneously.

### Q: What is the middleware execution order on `POST /events/ingest/:projectId`?

```
apiKeyAuth → projectRateLimiter → validateEvent → ctrl.ingestEvent
```

`validateEvent` runs after the API key is verified and the rate limiter passes. This is intentional: there is no reason to run expensive Joi validation if the request is unauthorized or rate-limited.

### Q: How does `validateEvent` validate both params and body?

It calls two separate Joi schema validations sequentially: `eventParamsSchema.validate(req.params)` first, then `eventBodySchema.validate(req.body)`. If params fail, it returns early without validating the body. Both use `abortEarly: false` within their own schema.

### Q: What does `hex().length(24)` accomplish for `projectId`?

It validates that the URL parameter is exactly 24 hexadecimal characters, which is the string representation of a MongoDB `ObjectId`. This prevents Mongoose from receiving a string that would cause a `CastError` when trying to cast it to `ObjectId`.

### Q: Why is `password` not validated strictly during login?

At login, the goal is simply to attempt authentication. The only check needed is that a non-empty password was provided. Complex rules (length, regex) at login would cause false rejections for users who registered before current rules were tightened.

### Q: Describe the known bug with `ValidateProject` on `GET /projects/list`.

`routes/project.Routes.js` applies `ValidateProject` to the list route. Since GET requests normally have no body, `req.body` is empty `{}`. Because `projectName` is `required()`, Joi returns a 400 error for every GET request to `/projects/list`. The endpoint is effectively broken unless a request body is sent, which is non-standard for a GET request.

### Q: What security gaps exist in the validation layer?

1. **No XSS sanitisation** — strings are validated for length/format but not stripped of HTML tags.
2. **`metadata` is unbounded** — any deeply nested object passes.
3. **No body size limit** in this layer.
4. **No rate limiting** on register or password login endpoints at the application layer.
5. **Unknown fields not stripped** — `stripUnknown` not set.
6. **GET /events/:projectId** does not validate `:projectId` via Joi.

### Q: How do the Joi schemas relate to the Mongoose models?

They enforce **complementary but distinct rules**:
- **Joi** validates the *incoming request* before it reaches the controller.
- **Mongoose** validates the *document structure* before it is written to MongoDB.

Most constraints (lengths, enums) are duplicated intentionally. If the Joi layer is bypassed somehow, Mongoose provides a second line of defence. However, they are maintained independently — a change to one does not automatically propagate to the other.

### Q: Is the password maximum correctly enforced?

The Joi schema enforces `max(500)`. However, the custom error message says "at most 50 characters". A client receiving this error message would be misled. The actual behaviour is that passwords up to 500 characters are accepted. This is a documentation/message bug, not a functional one.

---

## Files Inspected

| File | Purpose |
|---|---|
| `validations/auth.Validation.js` | Auth validation schemas and middleware |
| `validations/Event.validation.js` | Event ingestion validation schemas and middleware |
| `validations/project.validation.js` | Project creation validation schema and middleware |
| `routes/auth.Routes.js` | Verified which auth endpoints use validation middleware |
| `routes/event.Routes.js` | Verified event endpoint middleware chain |
| `routes/project.Routes.js` | Verified project route middleware chain, found GET bug |
| `models/Users.js` | Compared User schema constraints against `registerSchema` |
| `models/Event.js` | Compared Event schema constraints and enums against `eventBodySchema` |
| `models/Project.js` | Compared Project schema constraints against `ProjectSchema` |
| `middleware/ratelimiter.js` | Understood `otpLimiter` which runs on some auth routes |

---

*Last updated: 2026-09-16. All documentation is based on direct source code inspection. No behavior is assumed or invented.*
