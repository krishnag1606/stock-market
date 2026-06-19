# Flux — Technical Interview Cheat Sheet

> A deep-dive reference for discussing the Flux architecture in advanced full-stack and system design interviews.

---

## 1. Elevator Pitch

**Flux** is a community/group chat platform — "a simpler Discord" — built as a **Microsoft Rush + pnpm monorepo** orchestrating **3 independently-started services**: a React (CRA) front end on :3000, a Node.js/Express REST API on :8000, and a dedicated Socket.IO real-time server on :9000, all sharing a `@common/shared` library. It delivers **sub-second, channel-scoped real-time messaging** with **AES-256-GCM encryption at rest**, **server-verified Google OAuth backed by custom session JWTs**, and a **granular 11-flag bitfield RBAC system** enforced authoritatively across *both* the HTTP and WebSocket boundaries. The core value proposition: every security guarantee — identity, permissions, encryption, rate limits — lives server-side, with the client treated as untrusted and the UI as a mirror, never the source of truth.

---

## 2. Deep-Dive Tech Stack & Architectural Benefits

### Microsoft Rush + pnpm Monorepo

**What it does here:** Manages four projects in one repo — `apps/client-ui` (React), `server` (Express), `socket` (Socket.IO), and `libs/shared` (the `@common/shared` Api base class + components) — with Rush orchestrating install/build ordering and pnpm providing the package store.

**Engineering problems it solved:**
- **Dependency isolation without duplication.** pnpm uses a content-addressable global store and symlinks packages into each project's `node_modules`. Identical dependency versions are stored once on disk and hard-linked, so the three services don't each carry a full physical copy — **lightning-fast installs and minimal disk footprint**.
- **Strict, non-flat `node_modules`.** pnpm's symlink layout means a package can only import what it *explicitly declares* in its own `package.json` — it kills "phantom dependencies" (accidentally relying on a transitive dep that happens to be hoisted). This is real **dependency hell prevention**: no silent version drift between services.
- **Build orchestration & shared-library wiring.** `libs/shared` is referenced as `workspace:*`; Rush sequences its `tsdx build` *before* `client-ui` so `@common/shared` resolves. One source of truth for the `Api` base class consumed by the client services.
- **Real-world constraint discussion point:** Node 24 blocks `rush update` (`rush.json` pins Node 18/20). I worked around this by **manually unpacking packages** (`react-router-dom`, `express-rate-limit`, etc.) into `node_modules` and installing server-side deps with plain `npm` — a pragmatic answer to "what do you do when your tooling fights your runtime?"

> **Note on the socket service's clever reuse:** `socket/` has *zero* model code of its own. It imports the REST server's Mongoose models + crypto/jwt/messageView helpers via **relative import** (`../server/...`), and those transitive deps resolve out of `server/node_modules`. So `socket/`'s own `package.json` only depends on `socket.io`. `socket/loadEnv.js` (dependency-free) loads `server/.env` so JWT_SECRET / MESSAGE_ENCRYPTION_KEY / DB creds live in exactly one place — **identical secrets across services is a correctness requirement**, since a JWT signed by REST must verify on the socket and a message encrypted by one must decrypt on the other.

### Zod — Runtime Type Safety at Every Trust Boundary

**What it does here:** A **single shared validation module** (`server/util/validation.js`, deliberately placed under `server/` so the socket resolves `zod` from `server/node_modules` like the other shared helpers) is the **single source of truth** for input schemas. Both the REST controllers (`message-controller`, `poll-controller`) and the socket handlers (`sendMessage` / `createPoll` / `votePoll` / `deleteMessage` / `pinMessage`) import from it.

**Why it matters / problems solved:**
- **One schema, two boundaries.** Before hardening, the socket had hand-rolled `if` checks while REST had its own — a recipe for divergent validation where a payload rejected by HTTP slips through WebSocket. Centralizing in Zod means **HTTP and WebSocket enforce byte-identical rules**.
- **Runtime type safety in a dynamically-typed boundary.** TypeScript only guards compile time; the moment untrusted JSON hits the server, types are a lie. Zod `safeParse` validates at runtime and surfaces the **first issue** (`firstIssue`) cleanly to the client.
- **Concrete schema enforcement:** trims text, **caps message length at 4000 chars**, enforces **poll options between 2 and 10**, and **rejects `expiresAt` timestamps in the past**. `allowMultiple` is enforced server-side and never trusted from the client.
- **Defense against malicious input:** oversized payloads, malformed poll structures, and injection-shaped strings are rejected *before* they touch Mongoose or the crypto layer.

### Zustand — Decoupled Client State with Hard Synchronization Guarantees

**What it does here:** A single typed store (`IStore` + `store.ts`) split into **decoupled slices**: `currentUser` / `token`, `communities`, `channels`, `messages`, `roles`, `members`, `myPermissions`, `polls`, and the live `socket` reference.

**Hard runtime-synchronization problems it solved:**
- **Message deduplication by `_id`.** The socket broadcasts `newMessage` to the room *including the sender*, and a REST fallback path also exists. Without dedup, an optimistic send + the authoritative broadcast would render the same message twice. `addMessage` **dedups by `_id`**, making the two delivery paths idempotent. `addPoll` does the same for polls.
- **Out-of-order / authoritative-overwrite for live votes.** Poll voting is **optimistic**: `updatePoll` fires immediately on click, then the authoritative `pollUpdated` socket broadcast (or REST response) **overwrites** with the canonical tally — so a slow network or a racing vote can't leave the bar in a wrong state.
- **Safe multi-account reset (cross-account leak prevention).** `store.reset()` clears *everything including the `socket`*, and `AuthGate` keys the shell on the user id (`<AppShell key={currentUser._id} />`). Switching accounts **force-remounts** the tree → communities refetch and the socket reconnects under the new `userId`. This closes a real **cross-account data-leak** vector where stale community/message state from account A could bleed into account B.
- **The `socket` ref as a coordination primitive.** Storing the live connection in Zustand (rather than React context/refs) is what makes the real-time race-condition fix possible — see §4.

---

## 3. Monorepo vs. Alternative Architectures (Trade-Offs)

Interviewers want to hear you reason about *what you gave up* and *what you'd lose under a different model*.

### If it were a Multi-Repo system

| Friction point | Consequence |
|---|---|
| **Separate git repos per service** | No atomic cross-cutting change. A feature touching API + UI + socket needs 3 PRs across 3 repos, coordinated by hand — no single commit that's all-or-nothing. |
| **Publishing `@common/shared`** | Every shared-lib change requires **version-bump → publish to a private npm registry → bump the consumer's dependency → reinstall** in *each* service. A one-line type change becomes a multi-repo release dance. |
| **Breaking API contracts silently** | With independent versioning, the front end can pin an old `@common/shared` while the backend ships a breaking change. Contract drift is only caught at runtime, in prod. The monorepo gives a **single, always-consistent version** of the shared contract. |
| **Disjointed CI/CD** | Three pipelines, three sets of caches, no shared dependency graph. Rush's incremental, dependency-aware builds (only rebuild what changed downstream) are impossible. |

**Talking point:** the monorepo's killer feature here is the **atomic, single-version shared contract** — the `Api` base class and shared helpers can never be out of sync between consumers because there's only ever one copy on the branch.

### If it were a Monolith (single deployable)

| Downside | Consequence |
|---|---|
| **Can't scale Socket.IO independently** | Real-time and REST have *opposite* scaling profiles. WebSocket connections are **stateful and long-lived** (memory-bound, sticky); REST is **stateless and request-bound** (CPU/throughput). Bundled together you must over-provision the whole thing to absorb a connection spike. Split, I scale the socket fleet on concurrent connections and the REST tier on request rate — independently. |
| **Single point of failure** | A crash or a bad deploy takes down messaging *and* auth *and* REST at once. Separated, a socket-server restart doesn't drop the ability to log in or load history. |
| **Coupled build/deploy cadence** | UI builds get chained to backend deploys. A CSS-only change would force a full backend redeploy. The 3-service split lets the React bundle ship on its own cadence. |

**Honest counter-point (shows maturity):** the 3-service split costs you **operational complexity** (3 processes to run, shared-secret synchronization, a DNS/SRV gotcha for Atlas) and **the socket's relative-import coupling to `server/`** is a deliberate trade — it avoids duplicating models but means the socket can't be deployed from a directory that doesn't also contain `server/node_modules`. For a portfolio-scale project that's the right call; at true scale I'd extract the shared models into a published package.

---

## 4. Hardest Engineering Challenges (Deep Technical Breakdowns)

### A. Cryptographic Layer — AES-256-GCM Encryption at Rest

**The envelope.** `Message.content` is **not a string** — it's an object `{ ciphertext, iv, tag }`. The crypto helper exposes `encrypt(plaintext) -> { ciphertext, iv, tag }` / `decrypt(...) -> plaintext` over Node's built-in `crypto`, keyed by `MESSAGE_ENCRYPTION_KEY`.

- **`iv` (Initialization Vector):** a **fresh 96-bit (12-byte) random IV per `encrypt()` call**. 96 bits is the GCM-recommended IV size. *Why fresh-per-message matters:* GCM is a counter mode — **reusing an (key, IV) pair is catastrophic**: it leaks the XOR of two plaintexts and can collapse the authentication guarantee entirely. A new random IV per message keeps every ciphertext independent.
- **`tag` (GCM auth tag):** a 128-bit authentication tag produced via `setAuthTag` + `final()`. GCM is **AEAD** — it gives integrity, not just confidentiality. Decryption *fails* if the ciphertext or IV was tampered with. The hardened `decrypt` validates the **envelope shape + IV(12)/tag(16) byte-lengths + non-empty ciphertext** *before* touching OpenSSL, so truncated or non-GCM input fails clearly (and is caught upstream by `toMessageView` → `text: null`).

**Why server-side decryption (not E2E):** this is **at-rest protection against DB theft**, *not* end-to-end encryption. The server holds the key, encrypts on write, decrypts on read. That's a deliberate design choice: it keeps **moderation, polls, and search working** — a moderator deleting a message or an audit log referencing it needs the server to read plaintext. E2E would make authoritative moderation impossible. The threat model is "attacker dumps the MongoDB collection," and against that the ciphertext+tag is useless without the env-held key.

### B. Real-Time Race Condition — Child Effects Before Parent Init

**Situation:** Real-time was fully broken — messages only appeared after a manual refresh.

**Root cause (the subtle part):** **React runs child effects before parent effects** (mount order: children's `useEffect` fire before the parent's). `ChannelView` (child) ran its `joinChannel` / `onNewMessage` listener-attachment effect *before* `AppShell` (parent) ran `connectSocket()`. So the child hit a **still-`null` module-level socket** — `joinChannel` and the `on*` listeners **silently no-op'd**. The room was never joined, no listeners ever attached, and the connection that eventually opened broadcast into silence.

**Action:** Move the socket from a module-level variable into the **Zustand store** (set by `AppShell` once connected). `ChannelView` now **reads `socket` from the store, gates its effect on it, and lists it in the dependency array** — so the effect *re-runs* the moment the connection exists. Added `onConnect(handler)` and **re-join the room on every reconnect**, because room membership is **per-connection** (a dropped/re-established socket loses its rooms).

**Result:** Deterministic real-time with no refresh, resilient across reconnects. The backend emitters/auth were correct the whole time — this was purely a client-side lifecycle ordering bug. **Lesson: never assume a parent has initialized shared infrastructure by the time a child's effect runs; make the dependency explicit and reactive.**

### C. Bitfield RBAC & Security Boundaries

**Design.** A single integer **bitfield** on `Role.permissions`, with a small, intentional flag set:
`VIEW_CHANNELS, SEND_MESSAGES, CREATE_POLLS, POST_ANNOUNCEMENTS, MANAGE_MESSAGES, KICK, BAN, MUTE, MANAGE_CHANNELS, MANAGE_ROLES, MANAGE_COMMUNITY` (11 flags).

**Computation.** A member can hold multiple roles, so effective permissions = **bitwise OR overlay** of every role's bitfield (`computePermissions(membership, community)`). The community `ownerId` **implicitly gets `ALL_PERMISSIONS`**. Channel-type awareness: `sendPermissionFor(channel)` returns `POST_ANNOUNCEMENTS` for announcement channels, else `SEND_MESSAGES`.

**Dual-boundary enforcement (the key point).** Permissions are checked in **two authoritative places**:
- **Stateless REST:** `requirePermission(flag)` middleware runs after `requireMembership`, loads the community, attaches `req.permissions`, and 403s on a missing flag.
- **Stateful WebSocket:** the socket's `sendMessage` / `deleteMessage` / `pinMessage` / `createPoll` / `votePoll` handlers re-run the *same* permission + mute + slowmode checks. **A muted user's socket send is rejected server-side** — you cannot bypass REST guards by going through the socket.
- UI gating (hiding controls you can't use) is a **third, non-authoritative** layer — convenience only.

**Privilege-escalation guards (what interviewers probe):**
- You **can't grant, assign, or delete permission bits you don't hold yourself** (`grantsAllowed`).
- The **owner role (`ALL_PERMISSIONS`) can't be deleted**, and **only the owner can change the owner's roles**.
- **Role hierarchy via `canModerate(actorPerms, targetPerms, actorIsOwner)`** — an actor can only moderate a target whose permission set is a **subset** of the actor's (owner unrestricted).
- **Documented design trade-off (be honest about it):** `canModerate` is **permission-subset based, not role-position based** — so two members with *identical* permission sets can moderate each other. Left intentionally for simplicity; a position-ranked hierarchy (Discord-style) would be the production fix.

### D. Dual-Layer Rate Limiting

Two completely different mechanisms because HTTP and WebSocket have different shapes:

- **HTTP (REST) — `express-rate-limit@8`:** middleware in `server/middleware/rateLimit.js`:
  - `authLimiter`: **20 / 15 min per IP** on `/api/auth/*` (brute-force defense; keyed by IP since the user isn't authenticated yet).
  - `writeLimiter`: **30 / min per user** on message/poll create + vote.
  - `mutationLimiter`: **60 / min per user** on community/channel create + join.
  - Authenticated limiters key on the **verified `userId`** (via `ipKeyGenerator` fallback) and are mounted **after `requireAuth`** so the key is trustworthy. JSON 429 with `retryAfter`. `trust proxy` is opt-in via `TRUST_PROXY` env (off in dev) so the client IP isn't spoofable behind a proxy.
- **WebSocket — dependency-free, in-memory, fixed-window counters** (`socket/rateLimit.js`, `allowEvent(socket, event)`): `sendMessage` 10/10s, `createPoll` 5/min, `votePoll` 30/min. **Counters are stored *on the socket object*** so they **GC automatically on disconnect** — no external store, no leak, and per-connection isolation. This stops a single socket from flooding the messaging thread (a cheap DoS on the room broadcast).

**Scaling caveat (shows you know the limits):** both stores are **in-memory**, so they're per-process. Horizontally scaled, I'd swap to a **Redis-backed store** so limits are global across the fleet — explicitly flagged as a Phase 9 deploy task.

### E. Design System & UI Architecture — "Cozy 16-Bit RPG"

**The synchronization problem.** Theme tokens live in **two synced places**: `src/styles/_tokens.scss` (SCSS variables — `$color-primary`, `$shadow-chunky`, `$bevel-recessed`…) for module styles, and `src/index.css` `:root` (the same palette as CSS custom properties — `--color-primary`…) for inline styles / plain CSS. Keeping them in lockstep is the discipline that prevents drift.

**Avoiding style bleed.** Every component has a **co-located `.module.scss`** — CSS Modules scope class names locally (imported as a `styles` object), so no global selector leaks across components. The whole frontend was re-skinned from a neon "Y2K/Cyberpunk" theme to the warm SNES/Stardew look **purely by reworking the token files + the `src/ui/` library** — no layout/logic churn — which is only possible because styling is centralized in tokens, not hand-rolled per element.

**Semantic, layout-agnostic abstractions.** The `src/ui/` library is barrel-exported and **reused everywhere instead of raw styled elements**:
- `PixelButton` — wooden inventory-slot buttons, variants (`primary` / `cyan` / `lime` / `yellow` / `ghost` / `danger` / `icon`), `forwardRef`.
- `PixelCard` — RPG dialog box (cream inner frame + espresso outer outline).
- `PixelModal` — carved-wood title bar, Escape-to-close, optional footer.
- `PixelInput` — naming-screen text box with gold focus frame; `PixelIcon` — typed name map over `pixelarticons` SVGs (recolored via `currentColor`).

The framing language is **"RPG double borders, never glows"** — composed from reusable helpers (`$frame-outer-dark`, `$frame-inner-light`, `$bevel-recessed`, `$shadow-chunky`) so a new feature reaches for a token, never a one-off shadow. **Talking point:** this is a *design-token-driven* system — the components are semantic ("a dialog box," "an inventory button"), the *theme* is data, and a full re-skin is a token swap, not a rewrite.

---

## 5. High-Probability Interview Questions (with STAR talking points)

### Q1. "You split into 3 services — how do you keep identity and encryption consistent across them, and what breaks if they drift?"
- **Situation:** REST issues JWTs and encrypts messages; the socket must verify those same JWTs and decrypt those same messages.
- **Task:** Guarantee one set of secrets across two processes without duplicating logic.
- **Action:** The socket **imports the REST server's jwt/crypto/model helpers via relative import** and loads `server/.env` through a dependency-free `loadEnv.js`, so `JWT_SECRET` and `MESSAGE_ENCRYPTION_KEY` exist in exactly one file. Identity on the socket comes **only** from the verified JWT in `handshake.auth.token` → `socket.data.userId`, never from the client.
- **Result:** A token signed by REST verifies on the socket; a message encrypted on one decrypts on the other. If the secrets drifted, **JWT verify and at-rest decrypt would both fail across services** — which is exactly why secret synchronization is treated as a correctness invariant, not config.

### Q2. "Walk me through a real concurrency/ordering bug you fixed."
- **S:** Real-time messaging silently required a page refresh.
- **T:** Find why listeners never fired despite correct backend broadcasts.
- **A:** Diagnosed that **React fires child effects before parent effects**, so `ChannelView` attached socket listeners against a `null` socket before `AppShell` connected. Moved the socket into the **Zustand store**, **gated the child effect on it and added it to the deps**, and **re-joined the room on every reconnect** since rooms are per-connection.
- **R:** Deterministic, refresh-free real-time, resilient to reconnects. **Lesson:** make cross-component infrastructure dependencies explicit and reactive — don't assume parent init order.

### Q3. "How do you stop a user from escalating their own privileges or moderating someone above them?"
- **S:** A bitfield RBAC where members assign roles and moderate each other.
- **T:** Prevent privilege escalation and unauthorized moderation — authoritatively.
- **A:** Effective perms = **bitwise OR** of role bitfields; owner gets `ALL_PERMISSIONS` implicitly. Guards: you **can't grant/assign/delete bits you don't hold**, the owner role is **undeletable**, **only the owner edits owner roles**, and `canModerate` requires the **target's perms to be a subset of the actor's**. Enforced in **both** REST middleware and socket handlers; UI gating is cosmetic only.
- **R:** No client can escalate by going through the socket instead of REST. I'd also call out the **honest limitation**: `canModerate` is subset- not position-based, so equal-permission members can moderate each other — a documented trade-off I'd fix with ranked role positions at scale.

### Q4. "Your rate limiters are in-memory. What happens when you scale to multiple instances, and how would you redesign them?"
- **S:** `express-rate-limit` (HTTP) and a custom per-socket fixed-window counter (WebSocket), both in-process.
- **T:** Reason about correctness under horizontal scaling.
- **A:** In-memory counters are **per-process**, so behind N instances a user effectively gets N× the limit, and a reconnecting socket lands on a different node with a fresh counter. The fix is a **shared Redis store** for the HTTP limiter (atomic INCR + TTL) and a Redis or sticky-session strategy for socket counts; also set `TRUST_PROXY` so the real client IP keys the auth limiter behind a load balancer.
- **R:** Today (single process, portfolio scope) the in-memory design is correct and zero-dependency; the Redis swap is an **explicitly documented Phase 9 task** — I knew the boundary before being asked.

### Q5. "Why encrypt at rest server-side instead of end-to-end, and what's the cost of that decision?"
- **S:** Threat model is **DB theft**, and the product needs moderation, polls, and search.
- **T:** Pick an encryption strategy that protects stored data without breaking server features.
- **A:** Chose **AES-256-GCM at rest** with a `{ciphertext, iv, tag}` envelope, **fresh 96-bit IV per message**, GCM auth tag validated on read, and `decrypt` hardened to reject malformed envelopes before hitting OpenSSL. The server holds the key, so it can read plaintext to moderate, tally, and serialize messages.
- **R:** Strong protection against a dumped database, with full server-side functionality intact. **Cost (state it proactively):** it is **not** E2E — a server compromise *with* the env key exposes plaintext. E2E would defeat moderation entirely, so for this product at-rest is the correct trade; a real deployment would move the key into a **KMS** rather than env.

### Q6 (Behavioral). "Tell me about a time your tooling fought you."
- **S:** Node 24 in my environment violates `rush.json`'s Node 18/20 pin, so `rush update` refused to add dependencies.
- **T:** Ship features (`react-router-dom`, `express-rate-limit`, `zod`) without downgrading the whole environment mid-build.
- **A:** **Manually unpacked** the needed packages into the correct `node_modules` and installed server-side deps with **plain `npm`**, while keeping the lockfile-managed projects intact. Also fixed an Atlas **`querySrv ECONNREFUSED`** by setting an explicit public DNS resolver (`8.8.8.8`/`1.1.1.1`) and supporting a non-SRV `MONGODB_URI` override.
- **R:** Unblocked every phase without a disruptive toolchain migration. **Lesson:** know your toolchain well enough to route around it deliberately — and document the workaround so the next engineer doesn't rediscover the wall.

---

*Generated as an interview-prep reference for the Flux project. Pairs with `CLAUDE.md` (authoritative architecture) and `RESUME.md` (bullet summary).*
