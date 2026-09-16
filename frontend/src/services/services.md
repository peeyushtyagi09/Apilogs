# APILogs Frontend — `services/` Documentation

> **Source of truth:** every claim below was verified by reading the actual source in
> `frontend/src/services/` and every file needed to understand its real behaviour — its only
> consumer (`context/RealTimeContext.jsx`), the hook and page that drive it
> (`hooks/useRealtime.jsx`, `pages/project/ProjectDetail.jsx`), `App.jsx` and
> `context/AuthContext.jsx` for the token that feeds it, and the **entire backend socket
> stack** it talks to (`realtime/socket.server.js`, `socket.auth.js`, `socket.manager.js`,
> `server.js`, `example.env.js`, `utils/metrics.js`, and the controllers that emit). Installed
> package versions were read from `node_modules`. Nothing is assumed. Where the client and
> server contracts disagree, the **actual** behaviour is documented and listed in
> [§13 Documentation Discrepancies and Bugs](#13-documentation-discrepancies-and-bugs).

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [File Index](#2-file-index)
3. [socket.js — Complete Documentation](#3-socketjs--complete-documentation)
4. [Configuration and Environment Variables](#4-configuration-and-environment-variables)
5. [The Socket Event Contract](#5-the-socket-event-contract)
6. [Connection Lifecycle — Client to Server](#6-connection-lifecycle--client-to-server)
7. [Authentication and Authorization Flow](#7-authentication-and-authorization-flow)
8. [Room Model and Multi-Tenancy](#8-room-model-and-multi-tenancy)
9. [Reconnection Behaviour](#9-reconnection-behaviour)
10. [Error Handling Conventions](#10-error-handling-conventions)
11. [Security Implementation Notes](#11-security-implementation-notes)
12. [Performance Notes](#12-performance-notes)
13. [Documentation Discrepancies and Bugs](#13-documentation-discrepancies-and-bugs)
14. [Unverified Assumptions](#14-unverified-assumptions)
15. [Interview Preparation Notes](#15-interview-preparation-notes)

---

## 1. Architecture Overview

`frontend/src/services/` is the **transport layer for the real-time channel**. It contains one
file and one exported function, whose entire job is to construct an authenticated Socket.IO
client. It is the WebSocket counterpart to `src/api/`, which owns HTTP.

```
services/
└── socket.js      ← createSocketConnection(token) → a configured socket.io-client instance
```

**Where it sits in the app**

```
AuthContext (accessToken)
   ↓ prop
App.jsx → RealtimeProvider token={accessToken}
   ↓ calls once per token change
services/socket.js → createSocketConnection(token)
   ↓ returns
socket.io-client Socket  ──── WebSocket ────►  backend realtime/socket.server.js
   ↑                                                     ↓
RealtimeContext (events[], incidents[])          socket.manager.js (rooms)
   ↑
useRealtime(projectId) → subscribe / unsubscribe / history
   ↑
ProjectDetail → ActivityFeed, IncidentSummary, IncidentList
```

**Design characteristics, all verified:**

- **Factory, not singleton.** The module exports a function that builds a *new* socket each
  call. It holds no module-level state and caches nothing — connection ownership belongs
  entirely to `RealtimeProvider`, which stores the instance in `socketRef` and disconnects it
  on teardown. That is the right split: the service knows *how* to connect, the context knows
  *when*.
- **The only file in `src/` that imports `socket.io-client`.** Verified by grep — no page,
  hook or context talks to the library directly, so the connection configuration has exactly
  one definition site.
- **One consumer.** `context/RealTimeContext.jsx` line 29 is the sole call site.
- **Fail-fast on a missing token.** It throws synchronously rather than attempting an
  anonymous connection.
- **No business logic.** It registers three diagnostic listeners (`connect`, `disconnect`,
  `connect_error`) that only log; every domain event (`new-event`, `incident-updated`) is
  wired up by the consumer, not here.

**Installed versions** (read from `node_modules`): `socket.io-client@4.8.3` on top of
`engine.io-client@6.6.6`. The backend uses the matching `socket.io` v4 server API
(`new Server(httpServer, { cors })`, `io.use(...)`, `socket.join(room)`).

---

## 2. File Index

| File | Lines | Exports | Imports | Imported by |
|---|---|---|---|---|
| `socket.js` | 27 | `createSocketConnection` (named) | `io` from `socket.io-client` | `context/RealTimeContext.jsx` (once) |

There is **no default export** — the consumer uses
`import { createSocketConnection } from "../services/socket"`.

---

## 3. socket.js — Complete Documentation

### 3.1 Purpose

Create and return an authenticated, pre-configured Socket.IO client connected to the APILogs
backend, with connection diagnostics already attached.

### 3.2 Dependencies

| Import | Package | Used for |
|---|---|---|
| `io` | `socket.io-client@4.8.3` | the client factory; `io(url, options)` returns a `Socket` |

No React, no contexts, no utilities. The module is framework-agnostic and could be lifted into
any JavaScript client unchanged.

### 3.3 The exported function, step by step

```js
export const createSocketConnection = (token) => {
    if (!token) {                                                    // 1
        throw new Error("Socket connection require authentication token");
    }

    const socket = io(import.meta.env.VITE_Backend_URL, {            // 2
        transports: ["websocket"],                                   // 3
        auth: { token: token },                                      // 4
        reconnection: true,                                          // 5
        reconnectionAttempts: 5,
        reconnectionDelay: 2000,
    });

    socket.on("connect",       () => console.log("Socket connected:", socket.id));        // 6
    socket.on("disconnect",    (reason) => console.log("Socket disconnected: ", reason)); // 7
    socket.on("connect_error", (err) => console.error("Socket connection error:", err.message)); // 8

    return socket;                                                   // 9
};
```

**1 — Token guard.** Throws synchronously on any falsy token (`null`, `undefined`, `""`).
This is a **precondition assertion**, not an error to be caught: it converts a silent
anonymous-connection bug into a loud failure at the call site. The consumer never triggers it,
because `RealtimeProvider`'s effect returns early when `!token` — so the guard exists purely to
protect future callers. Note the message has a grammatical slip ("require" → "requires") and,
if it ever did fire inside the provider's effect, it would propagate out of a React effect and
crash the tree, since nothing wraps the call in a `try/catch` or an error boundary.

**2 — Connection target.** `import.meta.env.VITE_Backend_URL` — Vite's build-time env
substitution, not a runtime lookup. See §4 for what happens when it is unset (it is not a
no-op; the behaviour differs from the axios layer's).

**3 — `transports: ["websocket"]`.** This **disables the HTTP long-polling fallback** and
skips Socket.IO's default polling→WebSocket upgrade handshake. Trade-offs, both real:

| Benefit | Cost |
|---|---|
| One connection attempt instead of a polling handshake plus an upgrade — lower connect latency | If any proxy, corporate firewall or CDN blocks WebSocket frames, the connection fails **completely**, with no degraded mode |
| No sticky-session requirement, since there is no multi-request handshake to pin to one instance | Rules out environments that only support long-polling |

For a developer-facing monitoring dashboard this is a defensible call; for a consumer app with
unknown networks, keeping `["websocket", "polling"]` would be safer.

**4 — `auth: { token }`.** Socket.IO v4's dedicated handshake auth channel. The payload is
delivered in the connection packet and read server-side as `socket.handshake.auth.token`.
Two things matter here:

- **Why `auth` and not a query parameter:** query strings land in server access logs, proxy
  logs and browser history. The `auth` payload travels in the Socket.IO packet body, so the
  JWT is not logged by ordinary HTTP infrastructure. This is the correct choice.
- **It is a static object, not a function.** Socket.IO v4 also accepts
  `auth: (cb) => cb({ token: getFreshToken() })`, which is re-invoked on every reconnection
  attempt. With a static object the socket re-presents the **same** token forever. In this app
  that is moot — there is no token refresh anywhere — but it means that once the access token
  expires, reconnection can never succeed.

**5 — Reconnection policy.** `reconnection: true` with `reconnectionAttempts: 5` and
`reconnectionDelay: 2000`. Detailed in §9, including the fact that exhausting the five
attempts emits `reconnect_failed`, an event **nothing in this app listens for**.

**6–8 — Diagnostic listeners.** All three only write to the console:

| Event | Emitted when | Handler |
|---|---|---|
| `connect` | handshake accepted, socket usable | logs `socket.id` |
| `disconnect` | connection lost or closed; `reason` says why (`"io server disconnect"`, `"io client disconnect"`, `"transport close"`, `"ping timeout"`, …) | logs the reason |
| `connect_error` | handshake rejected or transport failed; `err.message` carries the server's `next(new Error(...))` text | logs the message |

These are attached **inside the factory**, so every socket gets them and the consumer cannot
forget them. They are also never removed — but since the consumer calls `socket.disconnect()`
and drops the reference, the instance is garbage-collected with its listeners.

**9 — Return value.** The raw `Socket` instance. The factory returns full control to the
caller: `RealtimeProvider` then attaches `new-event` and `incident-updated`, emits `subscribe`
and `unsubscribe`, and eventually calls `socket.off(...)` and `socket.disconnect()`.

### 3.4 Inputs, outputs and failure modes

| | |
|---|---|
| **Input** | `token: string` — a JWT access token, read from `AuthContext.accessToken` (itself seeded from `localStorage["token"]`) |
| **Output** | a connected-or-connecting `socket.io-client` `Socket` |
| **Throws** | `Error("Socket connection require authentication token")` when `token` is falsy |
| **Side effects** | opens a network connection; registers three listeners; writes to `console` |
| **Idempotent?** | No — each call creates a new connection. The caller must not call it twice without disconnecting the first |

### 3.5 Interview explanation

"`socket.js` is a one-function transport factory. It guards that a token exists, then builds a
socket.io client with three deliberate options: `transports: ['websocket']` to skip the polling
upgrade, `auth: { token }` so the JWT rides in the handshake packet rather than a query string
where proxies would log it, and a bounded reconnection policy. It attaches connection
diagnostics and hands the socket back — the React context owns the lifecycle, the service just
knows how to build one. The two things I'd change: make `auth` a function so a refreshed token
is used on reconnect, and surface `reconnect_failed`, because today after five failed attempts
the socket gives up and the UI never finds out."

---

## 4. Configuration and Environment Variables

### 4.1 Client side

| Variable | Read in | Used for |
|---|---|---|
| `VITE_Backend_URL` | `services/socket.js` line 8, **and** `api/api.js` line 4 | the socket origin, and the axios `baseURL` (`${VITE_Backend_URL}/api`) |

One variable configures both transports. Vite requires the `VITE_` prefix for a variable to be
exposed to client code, and substitution happens **at build time** — the built bundle contains
the literal value, so changing it requires a rebuild, not just a restart.

**There is no `.env` file in `frontend/`** (verified). If the variable is unset, the two
transports fail *differently*, which is worth knowing when debugging:

| Layer | Expression | Result when unset |
|---|---|---|
| axios | `` `${import.meta.env.VITE_Backend_URL}/api` `` | the **string** `"undefined/api"` → requests resolve against the page origin, e.g. `http://localhost:5173/undefined/api/...` → 404 |
| socket | `io(import.meta.env.VITE_Backend_URL, opts)` | `io(undefined, opts)` → socket.io-client treats a non-string URL as "same origin" and connects to the **page origin** (e.g. the Vite dev server on :5173), where no Socket.IO server is listening → `connect_error` |

So a missing env var produces a 404 on HTTP and a handshake failure on the socket — two
different symptoms from one root cause.

Note also the unconventional casing: `VITE_Backend_URL` (mixed case) rather than the usual
`VITE_BACKEND_URL`. Vite does not care, but it is an easy typo when writing the `.env`.

### 4.2 Server side

The backend socket server reads configuration from two *different* sources, which is a latent
fragility worth stating precisely:

| Consumer | Source | Line |
|---|---|---|
| `realtime/socket.auth.js` | `const { JWT_ACCESS_SECRET } = process.env;` — destructured **at module load** | top of file |
| `middleware/auth.js` (HTTP) | `require("../../example.env")` — the config module, which calls `dotenv.config()` | top of file |
| `realtime/socket.server.js` | `const { CLIENT_URL } = require("../../example.env");` — **imported but never used** | top of file |

`example.env.js` is what runs `require("dotenv").config()`. So `socket.auth.js` only sees a
populated `process.env.JWT_ACCESS_SECRET` if some module has already loaded `example.env.js`.
In the current `server.js` it does: `require("./app")` (line 3) loads `example.env` before
`require("./src/realtime/socket.server")` (line 5) loads `socket.auth`. **It works today —
because of require order.** But inside `socket.server.js` itself, `./socket.auth` is required
*before* `../../example.env`, so reordering the requires in `server.js` would leave
`JWT_ACCESS_SECRET` undefined and make every handshake fail with "Unauthorized socket
connection". Having `socket.auth.js` import the same config module as everything else would
remove the hazard.

`example.env.js` provides `JWT_ACCESS_SECRET` with **no default** and enforces its presence
only when `NODE_ENV` is production (via the `REQUIRED_VARS` check). `CLIENT_URL` is likewise
required in production — yet the socket server imports it and then ignores it in favour of
`origin: "*"` (§11).

---

## 5. The Socket Event Contract

Every event that crosses this connection, verified on both ends.

### 5.1 Client → Server

| Event | Payload | Emitted by | Server handler |
|---|---|---|---|
| `subscribe` | `{ projectId }` | `RealtimeContext.subscribeToProject` | `socket.manager.js` — validates, authorises, `socket.join("project:<id>")` |
| `unsubscribe` | `{ projectId }` | `RealtimeContext.unsubscribeFromProject` | `socket.manager.js` — `socket.leave("project:<id>")`, no validation |

### 5.2 Server → Client

| Event | Payload | Emitted by | Client handler |
|---|---|---|---|
| `new-event` | the full `Event` document | `emitEventToProject(io, projectId, event)` in `socket.manager.js`, called from `event.controller.ingestEvent` | ✅ `RealtimeContext` — pushed to `eventQueueRef`, flushed every 100 ms |
| `incident-updated` | the full `Incident` document | `event.controller.ingestEvent` (on ERROR/CRITICAL) **and** `incident.controller.updateIncidentStatus` | ✅ `RealtimeContext` — dedupes by `_id`, unshifts to the front |
| `subscription-success` | `{ projectId, message }` | `socket.manager.js` after a successful join | ❌ **no listener anywhere in the client** |
| `subscription-error` | a string — `"Project ID is required"`, `"Invalid project Id format"`, `"Project not found"`, `"Unauthorized access to this project"`, `"Subscription failed"` | `socket.manager.js` on every rejection path | ❌ **no listener anywhere in the client** |

### 5.3 Built-in lifecycle events

| Event | Listened for? | Where |
|---|---|---|
| `connect` | ✅ | `socket.js` — console only |
| `disconnect` | ✅ | `socket.js` — console only |
| `connect_error` | ✅ | `socket.js` — console only |
| `reconnect`, `reconnect_attempt`, `reconnect_error` | ❌ | — |
| `reconnect_failed` | ❌ | — (fires after the 5th failed attempt; see §9) |

**The headline gap:** the server has a complete, well-designed error channel
(`subscription-error`) for every way a subscription can fail — bad ID, missing project,
unauthorised access — and the client listens to **none** of it. Combined with the fact that
`useRealtime` shows a green "Subscribed to project … (Live updates enabled)" banner driven by a
600 ms `setTimeout` rather than by the server's acknowledgement, a user whose subscription was
*rejected* still sees the success banner and an empty feed.

---

## 6. Connection Lifecycle — Client to Server

### 6.1 Who calls the factory, and when

`RealtimeProvider`'s effect keys on `token`:

```js
useEffect(() => {
  if (!token) {                                  // logged out
    if (socketRef.current) { socketRef.current.disconnect(); socketRef.current = null; }
    eventQueueRef.current = []; setEvents([]); setIncidents([]);
    return;
  }
  const socket = createSocketConnection(token);  // ← the only call site
  socketRef.current = socket;
  socket.on("new-event", handleNewEvent);
  socket.on("incident-updated", handleIncidentUpdate);
  flushIntervalRef.current = setInterval(flush, 100);
  return () => {
    socket.off("new-event", handleNewEvent);
    socket.off("incident-updated", handleIncidentUpdate);
    socket.disconnect();
    socketRef.current = null;
    clearInterval(flushIntervalRef.current);
    eventQueueRef.current = [];
  };
}, [token]);
```

So exactly one socket exists per token value. Login creates it; logout destroys it and clears
both the event and incident arrays. Because `App.jsx` passes `accessToken` from `AuthContext`,
**the socket's lifetime is the session's lifetime**.

Under React `StrictMode` (enabled in `main.jsx`) this effect mounts, cleans up and re-mounts in
development, so the connection is established twice on first load — visible as two `connect`
logs. The cleanup is correct, so this is noise rather than a leak.

### 6.2 Full handshake trace

```mermaid
sequenceDiagram
    participant AC as AuthContext
    participant RP as RealtimeProvider
    participant SVC as services/socket.js
    participant IO as socket.io-client
    participant SRV as socket.server.js
    participant AUTH as socket.auth.js
    participant MGR as socket.manager.js
    participant M as metrics

    AC->>RP: accessToken becomes non-null
    RP->>SVC: createSocketConnection(token)
    SVC->>SVC: guard — throw if falsy
    SVC->>IO: io(VITE_Backend_URL, { transports:['websocket'], auth:{token}, reconnection… })
    IO->>SRV: WebSocket handshake, auth.token in the connect packet
    SRV->>SRV: io.use middleware — read socket.handshake.auth.token
    alt token missing
        SRV-->>IO: next(new Error("Authentication token missing")) → connect_error
    else token present
        SRV->>AUTH: verifySocketToken(token)
        AUTH->>AUTH: jwt.verify(token, process.env.JWT_ACCESS_SECRET); require decoded.sub
        alt invalid / expired / no sub
            AUTH-->>SRV: throw Error("Unauthorized socket connection")
            SRV-->>IO: next(new Error("Authentication failed")) → connect_error
        else valid
            AUTH-->>SRV: decoded payload
            SRV->>SRV: socket.userId = user.sub
            SRV-->>IO: next() → connection accepted
        end
    end
    IO-->>SVC: "connect" → console.log(socket.id)
    SRV->>MGR: registerSocketHandlers(io, socket)
    MGR->>M: metrics.incrementSocketConnections()
    MGR->>MGR: register "subscribe" / "unsubscribe" / "disconnect" handlers
```

### 6.3 Subscribe trace (after connection)

```mermaid
sequenceDiagram
    participant PD as ProjectDetail
    participant H as useRealtime
    participant RC as RealtimeContext
    participant S as socket
    participant MGR as socket.manager.js
    participant DB as MongoDB

    PD->>H: useRealtime(projectId) on mount
    H->>RC: subscribeToProject(projectId)
    RC->>RC: setEvents([]) ; eventQueueRef = []
    RC->>S: emit "subscribe" { projectId }
    S->>MGR: handler
    MGR->>MGR: !projectId → emit "subscription-error" (client ignores)
    MGR->>MGR: !ObjectId.isValid → emit "subscription-error" (client ignores)
    MGR->>DB: Project.findById(projectId).select("ownerId")
    alt not found
        MGR-->>S: "subscription-error" — "Project not found" (client ignores)
    else ownerId !== socket.userId
        MGR->>DB: AuditLog.create(SOCKET_UNAUTHORIZED)  %% ReferenceError — AuditLog not imported
        MGR-->>S: "subscription-error" — "Unauthorized access to this project" (client ignores)
    else authorised
        MGR->>S: socket.join("project:<projectId>")
        MGR-->>S: "subscription-success" (client ignores)
    end
```

### 6.4 Teardown

| Trigger | Path |
|---|---|
| Logout | `accessToken` → `null` → provider effect's `!token` branch → `disconnect()`, state cleared |
| Provider unmount | cleanup → `off` both handlers → `disconnect()` → `clearInterval` |
| Leaving a project page | `useRealtime` cleanup → `unsubscribeFromProject` → `emit "unsubscribe"` → `socket.leave(room)`; **the socket itself stays open** |

That last row is the correct design: room membership is per-page, the connection is per-session.
On the server, `disconnect` is handled **twice** — `socket.server.js` logs it and
`socket.manager.js` calls `metrics.decrementSocketConnections()`. Both fire; neither conflicts.

---

## 7. Authentication and Authorization Flow

### 7.1 Two independent checks

| Stage | Where | What it proves |
|---|---|---|
| **Handshake authentication** | `socket.server.js` `io.use` → `socket.auth.js` | the JWT is signed by this server and carries a `sub` claim → `socket.userId` |
| **Per-room authorization** | `socket.manager.js` `subscribe` handler | this `socket.userId` owns *this* project → allowed to join `project:<id>` |

This separation is the most interview-relevant thing in the whole folder: **authenticating the
connection is not enough.** Without the second check, any logged-in user could emit
`subscribe` with another tenant's `projectId` and receive their live event stream. The backend
does implement it — `project.ownerId.toString() !== socket.userId.toString()` → reject — so
multi-tenancy holds on the WebSocket channel exactly as it does on REST.

### 7.2 What the token is, and its limits

`socket.auth.js`:

```js
const decoded = jwt.verify(token, JWT_ACCESS_SECRET);
if (!decoded || !decoded.sub) throw new Error("Invalid token payload");
return decoded;
```

- Verifies signature and expiry (`jwt.verify` throws on an expired token).
- Requires `sub`; **does not** check `tv` (token version) — so, exactly as with the HTTP
  `authRequired` middleware, "logout everywhere" does not invalidate a live or reconnecting
  socket.
- Every failure is collapsed into one generic message before being re-thrown, and
  `socket.server.js` collapses it again into `"Authentication failed"`. The client therefore
  cannot distinguish an expired token from a malformed one — deliberate for security, but it
  means the UI cannot offer "your session expired, please log in again".

### 7.3 Token freshness on reconnect

Because `auth` is a static object captured at construction time, a reconnection presents the
original token. Once it expires:

1. the socket drops (or a network blip triggers reconnection);
2. every reconnect attempt fails `jwt.verify` → `connect_error`;
3. after 5 attempts, `reconnect_failed` fires — unheard;
4. the UI shows no change: `ProtectedRoute` still passes (it only checks that a token string
   exists), the feed simply stops updating.

That chain is the clearest illustration of why this app needs both a refresh mechanism and a
`reconnect_failed` listener.

---

## 8. Room Model and Multi-Tenancy

### 8.1 Naming

Rooms are `project:${projectId}` — a namespaced key built in three places that must agree:

| Location | Code |
|---|---|
| join / leave | `socket.manager.js` — `const roomName = \`project:${projectId}\`` |
| event broadcast | `socket.manager.js` `emitEventToProject` — same template |
| incident broadcast | `event.controller.js` and `incident.controller.js` — `io.to(\`project:${projectId}\`)` inline |

The literal is duplicated across files rather than exported from one helper — a refactor hazard
(§13), though all four current spellings match.

### 8.2 Why rooms rather than client-side filtering

The server broadcasts only into the room, so a browser physically never receives another
tenant's events. The alternative — broadcasting widely and filtering in the client — would put
other customers' data on the wire, which is a data-leak whether or not the UI renders it. Using
rooms means authorization is enforced **once, at join time**, and every subsequent emit is
implicitly scoped.

### 8.3 One socket, many rooms

The connection is per-session; rooms are joined and left per page. A user with two tabs open on
two projects has two sockets, each in one room. Nothing prevents a single socket joining
several rooms — `useRealtime` just never does, because it unsubscribes on unmount.

**Client-side caveat:** `RealtimeContext.incidents` is **provider-global**, not keyed by
project. The `incident-updated` handler unshifts whatever arrives, regardless of project. Room
scoping is what keeps that array clean today, plus `ProjectDetail` calling
`initializeIncidents()` on mount. If a socket ever joined two rooms, the incident list would
interleave projects.

---

## 9. Reconnection Behaviour

### 9.1 Configured policy

| Option | Value | Meaning |
|---|---|---|
| `reconnection` | `true` | automatic retry after an unexpected drop |
| `reconnectionAttempts` | `5` | give up after 5 consecutive failures |
| `reconnectionDelay` | `2000` | base delay before the first retry |
| `reconnectionDelayMax` | *(default 5000)* | ceiling on the backoff |
| `randomizationFactor` | *(default 0.5)* | jitter applied to each delay |

Socket.IO applies exponential backoff with jitter between the base and the max, so retries land
roughly in the 2 s → 5 s range with randomisation — the jitter matters because it prevents a
thundering herd of reconnects when a server restarts and every client retries in lockstep.

### 9.2 What is and is not automatic

| Behaviour | Automatic? |
|---|---|
| Re-establishing the connection | ✅ up to 5 attempts |
| Re-sending the `auth` payload | ✅ — but the **same, possibly stale** token |
| **Re-joining rooms** | ❌ — rooms are server-side membership and are lost on disconnect |
| Replaying missed events | ❌ — no buffering, no sequence numbers, no catch-up query |
| Notifying the UI of permanent failure | ❌ — `reconnect_failed` has no listener |

### 9.3 The room-rejoin gap

This is the most consequential runtime issue in the realtime pipeline. After a successful
reconnect, the client is connected but **in no room**, because nothing re-emits `subscribe`.
`useRealtime` emits it once, in an effect keyed on `[projectId, …]`, and a reconnect does not
change `projectId`. The observable result: after a network blip, the page looks fine, the
console logs `Socket connected: <new id>`, and the feed **silently stops receiving events**
forever.

The fix is small and belongs in the consumer, not this service — re-emit `subscribe` from a
`connect` handler:

```js
socket.on("connect", () => { if (currentProjectId) socket.emit("subscribe", { projectId: currentProjectId }); });
```

### 9.4 Missed-event gap

Even with room rejoin, events emitted while disconnected are gone — the server does not buffer,
and the client does not re-fetch history on reconnect. A complete solution would re-run
`getProjectEvents(projectId, { before: <newest seen> })` after reconnecting. The HTTP endpoint
already supports the cursor; nothing calls it for this purpose.

---

## 10. Error Handling Conventions

| Failure | Handling | Visible to the user? |
|---|---|---|
| Missing token at call time | `throw new Error(...)` synchronously | would crash the effect — no boundary exists; unreachable in practice |
| Handshake rejected (no token / bad token) | `connect_error` → `console.error(err.message)` | ❌ console only |
| Connection dropped | `disconnect` → `console.log(reason)` | ❌ console only |
| Reconnect attempts exhausted | `reconnect_failed` — **no listener** | ❌ nothing at all |
| Subscription rejected by the server | `subscription-error` — **no listener** | ❌ nothing; the green "Live updates enabled" banner still shows |
| Unauthorised subscription attempt | server emits `subscription-error` + tries an `AuditLog` write (which throws — see §13) | ❌ client-side |
| Server-side handler throws | wrapped in `try/catch`; emits `"Subscription failed"`; increments a metric if the function exists | ❌ client-side |

**The convention, stated plainly: every realtime error is logged, none is rendered.** The
HTTP layer at least surfaces messages into page-level error banners; the socket layer has no
equivalent. A `connectionStatus` value exposed from `RealtimeContext` (`connecting` /
`connected` / `reconnecting` / `failed`) would close the gap, and every event needed to drive
it already exists on the client.

Server-side error handling is more defensive than the client's, and worth noting for contrast:
every `AuditLog.create` is wrapped in its own `try/catch` so audit-logging failure cannot break
a subscription, the `subscribe` handler validates ObjectId format before querying, and
`metrics.incrementSubscriptionError` is called only after a `typeof … === "function"` check.

---

## 11. Security Implementation Notes

### 11.1 Implemented and verified

- **JWT required on the handshake** — an anonymous connection is rejected before
  `io.on("connection")` runs, so unauthenticated sockets never reach any handler.
- **Token in the `auth` payload, not the query string** — keeps the JWT out of access logs,
  proxy logs and `Referer` headers. This is the single best security decision in the file.
- **Per-room ownership check** — `project.ownerId` vs `socket.userId` on every `subscribe`,
  so authentication alone does not grant access to a tenant's stream.
- **ObjectId validation before the database query** — `mongoose.Types.ObjectId.isValid` guards
  the lookup.
- **Audit logging intent** — an `AuditLog` entry with `purpose: "SOCKET_UNAUTHORIZED"` is
  attempted on rejection (see §13 for why it currently throws).
- **Generic client-facing auth errors** — the server never tells the client *why* the token
  failed.
- **Fail-fast client guard** — no accidental anonymous connections from this factory.

### 11.2 Weaknesses

| Weakness | Evidence | Impact |
|---|---|---|
| **Socket.IO CORS is `origin: "*"`** | `socket.server.js` — `cors: { origin: "*", methods: ["GET","POST"] }`, while `CLIENT_URL` is imported and unused | any origin may attempt a handshake. The JWT check is the only barrier, and because the token lives in `localStorage` (not a cookie), the browser's same-origin policy is *not* an additional layer here. The HTTP API is stricter — `app.js` restricts CORS to `CLIENT_PRO_URL` |
| Token from `localStorage` | `utils/token.js` | XSS-readable; an attacker who can run script can open their own authenticated socket |
| No `tv` (token version) check | `socket.auth.js` | "logout everywhere" does not close or block sockets |
| Token never refreshed on reconnect | static `auth` object | expiry permanently breaks reconnection |
| **Token logged to the console** | `socket.server.js` logs `"socket auth token: "` plus the token and the decoded payload; `App.jsx` logs `accessToken` | bearer tokens in browser and server logs |
| No per-socket rate limiting | `socket.manager.js` | a client may emit `subscribe` in a loop; each emission triggers a `Project.findById`, so the handler is a cheap DoS amplifier. HTTP has `otpLimiter` and `projectRateLimiter`; the socket channel has nothing |
| No payload validation beyond ObjectId shape | `socket.manager.js` | a malformed `{ projectId: { $ne: null } }` style payload is caught by `isValid`, so this is currently adequate — but it is the only validation present |

### 11.3 Not applicable / not implemented

- No message encryption beyond the transport's own TLS (`wss://` in production — depends on
  deployment, unverified).
- No CSRF concern: the token is presented explicitly, not sent as an ambient cookie.
- No signed or sequence-numbered messages — the client trusts everything the server emits,
  which is fine given the server is the only emitter into a room.

---

## 12. Performance Notes

| Technique | Where | Effect |
|---|---|---|
| WebSocket-only transport | `socket.js` | skips the polling handshake and upgrade; fewer round trips to first message |
| One connection per session | `RealtimeProvider` effect keyed on `token` | not one per page or per component; navigating between projects reuses the socket and only changes rooms |
| Server-side room filtering | `socket.manager.js` | a client receives only its own project's events — no wasted bandwidth, no client-side filtering cost |
| **Event batching in the consumer** | `RealtimeContext` — `eventQueueRef` + a 100 ms flush | incoming events are pushed to a ref, never to state; a burst of 200 events causes ~10 renders/second instead of 200 |
| Bounded memory | `MAX_EVENTS = 50`, trimmed on each flush | the array cannot grow without limit no matter the ingest rate |
| Virtualised rendering downstream | `ActivityFeed` + `react-window` | only visible rows mount |
| Jittered reconnect backoff | socket.io defaults | prevents a thundering herd when the server restarts |

**Costs and gaps:** there is no `disconnect` on page hide (a background tab keeps its socket and
keeps receiving events); no message compression is configured; and the 100 ms flush interval
runs unconditionally for the whole session even when no events arrive — negligible, but it is a
timer that could be started lazily.

---

## 13. Documentation Discrepancies and Bugs

Ordered by impact. All traced from source; none confirmed by running the app.

### 13.1 Client-side gaps

| # | Location | Problem | Effect |
|---|---|---|---|
| 1 | `RealtimeContext` / `useRealtime` | **No room re-join after reconnect.** `subscribe` is emitted once from an effect keyed on `projectId`; a reconnect does not re-emit it | after any network blip the socket reconnects into **no room** and the feed silently dies for the rest of the session |
| 2 | client-wide | **`subscription-error` has no listener.** The server emits it on five distinct failure paths | every subscription failure — bad ID, missing project, *unauthorised access* — is completely invisible |
| 3 | client-wide | **`reconnect_failed` has no listener** | after 5 failed attempts the socket gives up permanently with zero UI indication |
| 4 | `useRealtime` | The green "Subscribed to project … (Live updates enabled)" banner is driven by a **600 ms `setTimeout`**, not by `subscription-success` | the UI claims live updates are on even when the server rejected the subscription |
| 5 | `socket.js` | `auth` is a **static object**, not a function | reconnection always re-presents the original token; once it expires, reconnection can never succeed |
| 6 | client-wide | No reconnect **history catch-up** — events emitted while disconnected are lost, though the cursor-paginated `GET /api/events/:projectId?before=` endpoint exists and could backfill | permanent gaps in the feed after any drop |
| 7 | client-wide | No `connectionStatus` exposed from `RealtimeContext` | the UI cannot render "reconnecting…" or "connection lost", even though every needed event is available |
| 8 | `socket.js` line 5 | Error message grammar: `"Socket connection require authentication token"` | cosmetic |
| 9 | `socket.js` | If the throw ever fired inside `RealtimeProvider`'s effect it would propagate uncaught — there is no error boundary anywhere in the app | unreachable today (the provider guards `!token` first) |
| 10 | env | `VITE_Backend_URL` unset → `io(undefined, …)` silently targets the **page origin** rather than failing loudly | confusing `connect_error` that looks like a server problem |

### 13.2 Server-side defects reachable through this channel

| # | Location | Problem |
|---|---|---|
| 11 | `socket.manager.js` | **`AuditLog` is used but never imported** (the file imports only `mongoose`, `Project`, `metrics`). The unauthorised-subscribe branch therefore throws `ReferenceError`, caught by the local `try/catch` and merely logged — so **unauthorised subscription attempts are never actually audited**. The rejection itself still works |
| 12 | `socket.server.js` | **`CLIENT_URL` is imported and never used**; CORS is hardcoded to `origin: "*"` while the HTTP app restricts to `CLIENT_PRO_URL` — the two transports have inconsistent origin policies |
| 13 | `socket.auth.js` | Reads `process.env.JWT_ACCESS_SECRET` **destructured at module load**, while every other module reads the `example.env` config object. It works only because `server.js` requires `./app` (which loads `example.env` → `dotenv.config()`) before `socket.server.js`. Reordering those requires would silently break all socket auth |
| 14 | `socket.server.js` | **Logs the raw JWT and the decoded payload** (`console.log("socket auth token: ", token)`, `console.log("user", user)`) on every handshake |
| 15 | `socket.manager.js` | No rate limiting on `subscribe`; each emit costs a `Project.findById` |
| 16 | `socket.manager.js` | `unsubscribe` performs **no validation or ownership check** — harmless (leaving a room you are not in is a no-op) but inconsistent with `subscribe` |
| 17 | `socket.manager.js` + controllers | The room name `` `project:${projectId}` `` is re-derived as a string literal in **four** places instead of a shared helper |
| 18 | `socket.manager.js` | `registerSocketHandlers` is declared `async` but never awaits anything at the top level, and `socket.server.js` calls it without awaiting — harmless, but misleading |
| 19 | `socket.server.js` + `socket.manager.js` | `disconnect` is handled in both files (one logs, one decrements the metric). Correct, but the split means the connection-count metric and the log live in different modules |

### 13.3 Things that are correct and should be preserved

- The **factory pattern** — connection construction in one place, lifecycle ownership in the
  context.
- **Fail-fast token guard** — no silent anonymous connections.
- **`auth: { token }` rather than a query parameter** — keeps the JWT out of logs.
- **Two-stage security** — handshake authentication plus per-room authorization.
- **Diagnostic listeners attached inside the factory** — every socket gets them by
  construction.
- **Complete teardown in the consumer** — `off` both handlers, `disconnect`, clear the
  interval and the queue.
- **Per-session connection with per-page room membership** — the right granularity.
- Server-side: ObjectId validation before the query, `try/catch` around every audit write, and
  the `typeof` guard before calling an optional metrics method.

---

## 14. Unverified Assumptions

**Not verified from the available source code:**

1. **Runtime behaviour.** Nothing was confirmed by running the app or a server. Every finding —
   including the room-rejoin gap and the same-origin fallback when `VITE_Backend_URL` is unset —
   is derived by reading source and applying documented Socket.IO v4 semantics.
2. **`VITE_Backend_URL`'s actual value.** No `.env` exists in `frontend/`; the intended origin
   is unknown, as is whether production uses `wss://`.
3. **Socket.IO Manager multiplexing.** socket.io-client caches Managers per URL by default
   (`multiplex`), so the precise behaviour of calling `io()` again after a `disconnect()` —
   whether a fresh Manager is created or a cached one reused, and how the new `auth` payload is
   applied — was **not** verified against the library source or at runtime.
4. **Exact backoff timings.** `reconnectionDelayMax` (5000) and `randomizationFactor` (0.5) are
   quoted as socket.io-client v4 documented defaults; they are not set in this codebase and
   were not read from the package.
5. **`example.env.js` values.** `JWT_ACCESS_SECRET`, `CLIENT_URL` and `CLIENT_PRO_URL` are
   referenced by name; the configured values were not read (no `.env` is committed).
6. **Deployment topology.** Whether the backend runs behind a proxy that permits WebSocket
   upgrades, and whether it is single- or multi-instance, is unknown. This matters: with more
   than one instance and no Redis adapter, `io.to(room).emit(...)` reaches only clients
   connected to the emitting instance — no adapter configuration exists in the repo.
7. **`engine.io-client@6.6.6` internals.** Transport negotiation specifics were not read from
   the package.
8. **Whether missed events matter in practice** — that depends on ingest volume and typical
   disconnect duration, neither of which is measurable from source.

---

## 15. Interview Preparation Notes

### 15.1 Questions to expect

**Q: Walk me through how a real-time event reaches the browser.**
A producer POSTs to `/api/events/ingest/:projectId` with an API key. The controller writes the
`Event`, and if the severity is ERROR or CRITICAL it upserts an `Incident` grouped by a
normalised message signature. It then calls `emitEventToProject(io, projectId, event)`, which
does `io.to('project:<id>').emit('new-event', event)`. Every socket that joined that room
receives it. On the client, `RealtimeContext`'s handler pushes it into a ref-held queue, and a
100 ms interval flushes the queue into state with a 50-event cap. `ActivityFeed` renders it
through `react-window`.

**Q: How do you authenticate a WebSocket?**
You can't use headers the way you do with HTTP, so Socket.IO gives you the `auth` payload in
the handshake: `io(url, { auth: { token } })`, read server-side as
`socket.handshake.auth.token` inside an `io.use()` middleware. I deliberately use `auth`
rather than a query parameter, because query strings end up in access logs, proxy logs and
`Referer` headers — a bearer token should never be there. The middleware verifies the JWT and
attaches `socket.userId`; if it calls `next(new Error(...))` the connection is refused before
any handler runs.

**Q: Authentication isn't enough though — what about authorization?**
Right, and that's the key point. The handshake proves *who* you are; it says nothing about
*which* projects you may watch. So the `subscribe` handler re-checks: it validates the
ObjectId, loads the project, and compares `project.ownerId` to `socket.userId` before calling
`socket.join()`. Without that second check, any authenticated user could subscribe to another
tenant's project ID and receive their live event stream.

**Q: Why rooms instead of broadcasting and filtering client-side?**
Filtering in the browser means other customers' data is on the wire — that's a data leak
regardless of what the UI renders. With rooms, authorization is enforced once at join time and
every subsequent emit is implicitly scoped, so the browser physically never receives data it
isn't entitled to. It's also less bandwidth and less client CPU.

**Q: What happens when the connection drops?**
Socket.IO retries automatically — here, five attempts with exponential backoff and jitter from
a 2 s base. The jitter matters: without it, every client reconnects in lockstep after a server
restart and you get a thundering herd. But I'd be upfront that our reconnection is incomplete:
**rooms are server-side state and are lost on disconnect**, and nothing re-emits `subscribe` on
reconnect, so after a blip the socket is connected but in no room and the feed silently dies.
The fix is a `connect` handler that re-subscribes, ideally followed by a cursor-paginated
history fetch to backfill events missed while offline.

**Q: Why `transports: ['websocket']`?**
It skips the polling handshake and the upgrade, so you connect faster and you don't need
sticky sessions in a load-balanced deployment. The cost is that there's no fallback — if a
proxy blocks WebSocket frames, the connection fails outright instead of degrading to
long-polling. For a developer-facing dashboard that's an acceptable trade; for a consumer app
on unknown networks I'd keep polling in the list.

**Q: Why is this a factory function instead of a module-level singleton?**
Because the socket's lifetime is the session's lifetime, and the session is React state. A
module-level singleton would be created at import time, before a token exists, and couldn't be
rebuilt on login or torn down on logout. The factory takes the token as an argument and returns
a new instance; `RealtimeProvider` holds it in a ref and destroys it in the effect cleanup. The
service knows *how* to connect; the context knows *when*.

**Q: What would you fix first?**
Ranked: (1) re-emit `subscribe` on `connect` — the room-rejoin bug silently breaks the core
feature; (2) listen for `subscription-error` and `reconnect_failed`, and expose a
`connectionStatus` from the context so the UI can show "reconnecting" or "live updates
unavailable" — right now the green "Live updates enabled" banner is a `setTimeout`, so it lies
when a subscription was rejected; (3) make `auth` a function so reconnection picks up a
refreshed token, which requires implementing refresh at all; (4) restrict the socket CORS to
the client origin instead of `*`; (5) stop logging the JWT on both ends; (6) import `AuditLog`
in `socket.manager.js`, since unauthorised subscription attempts currently fail to audit.

### 15.2 Concepts to be able to define cold

WebSocket vs HTTP long-polling · the Socket.IO handshake and `io.use()` middleware · the `auth`
payload vs query parameters vs headers · rooms as a server-side broadcast primitive ·
authentication vs authorization on a persistent connection · exponential backoff with jitter
and the thundering-herd problem · why rooms don't survive reconnection · at-most-once delivery
and missed-message backfill · batching socket events through a ref to avoid render storms ·
sticky sessions and why WebSocket-only transport avoids them · the Redis adapter and why
`io.to(room)` breaks across multiple instances without one · JWT expiry on long-lived
connections · CORS on WebSocket vs HTTP.

### 15.3 The honest framing to use

This is a small, well-shaped transport module: the factory pattern is right, the `auth`
payload choice is the correct security decision, and the backend behind it does the hard part
properly by re-checking ownership at room-join time rather than trusting the handshake alone.
The weaknesses are all in the **lifecycle around it**, not in the file: no room re-join after
reconnect, no listener for the server's error channel, and no connection status surfaced to
the UI. Being able to describe the room-rejoin failure precisely — connected socket, no room,
silently dead feed — and give the three-line fix is a stronger answer than claiming the
realtime layer is complete.
