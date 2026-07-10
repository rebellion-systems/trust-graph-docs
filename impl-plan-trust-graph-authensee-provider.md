# Trust Graph as an Independent AuthenSee Provider — Implementation Plan

_Author: Rebby · 2026-07-10 · Status: proposal for Gonçalo review_

## 0. Guiding rule (seminal, Gonçalo 2026-07-10)
For now there are **no special relationships** between AuthenSee, The Real Network (TRN), and the trust graph. Each is an independent actor consuming the others over their **public APIs**. No dev-header bypasses, no localhost shortcuts, no app holding another's privileged write credential. Cross-app auth is real: standard AuthenSee provider login + JWKS-verified tokens; device-key-signed trust-graph actions.

---

## 1. Current state (grounded in code) — why "Register human node" 401s

- **`authnc-trust-graph/src/auth.ts`** — `resolveSubject` requires a `Bearer` JWT verified via `createRemoteJWKSet` against `AUTHENSEE_JWKS_URL`, checking `issuer=AUTHENSEE_JWT_ISSUER`, `audience=AUTHENSEE_JWT_AUDIENCE`. Dev-header (`x-authensee-subject`) only when `devHeaderAuth` (`AUTHENSEE_DEV_AUTH=true` OR `NODE_ENV!=production`). Staging/prod run `false` + `production` → **Bearer mandatory**.
- **`authnc-trust-graph/src/app.ts`** — `POST /v1/graph-keys` (node registration / key bind) calls `resolveSubject` unconditionally → JWT mandatory. `edges/request` is signature-only; renew/revoke/confirm use `optionalSubject` (device-key sig primary). So **the JWT gate is exactly one point: graph-key bind**.
- **`authnc-auth-server/src/auth-results/token.ts`** — the exchanged token sets `sub ← providerSubject`, **`aud ← providerId`**, `iss ← issuer`. There is no audience override.
- **`authnc-auth-server/src/auth-results/routes.ts`** — `POST /v1/auth-results/exchange` requires provider secret-key auth (`request.provider` via `x-api-key`) and returns `{ ..., token, providerSubject }`.
- **`the-real-network/server/provider-api.mjs`** — `trustGraphRequest()` attaches `Authorization: Bearer` only if `TRUST_GRAPH_AUTHENSEE_JWT` is set (it's unset), and `x-authensee-subject` only if `isLocalTrustGraph()` (false for the remote host). Against remote staging/prod it sends **no auth** → `401 Missing bearer token`.

**Root cause (two layers):**
1. No token with `aud=trust-graph-*` exists anywhere — the only token AuthenSee mints has `aud=providerId`.
2. The trust graph has **zero** code to obtain such a token; it only *verifies*. So the minting/exchange side of this integration does not exist yet.

---

## 2. Target architecture

1. **Trust graph = its own AuthenSee provider** — own `providerId` + own secret key (`sk_…`). Consumes AuthenSee's normal provider API exactly like TRN does for itself.
2. **Node registration = a real AuthenSee ceremony scoped to the trust-graph provider.** The trust graph's **own backend** creates the AuthenSee session (its provider key), the human authenticates, AuthenSee redirects to the trust graph's callback, the trust graph **exchanges the code with its own key** → gets `providerSubject` (and a token with `aud=trust-graph-*`) → binds the device graph key. One-time per device key.
3. **After bind → device-key signatures only.** No bearer, no provider involvement.
4. **TRN → trust graph is a plain client.** Device-key-signed writes; open reads. TRN never authenticates *as* the trust graph and never injects trust-graph auth.

**Identity note (important):** `deriveProviderSubject(secret, personaId, providerId, externalUserId)` includes `providerId`, so the **same human has a different subject under the trust-graph provider than under TRN**. That's correct for decoupling — but any "this TRN user ↔ this graph node" correlation must go through the shared AuthenSee **persona**, not a shared subject. TRN's "my node" UI must account for this.

---

## 3. Workstreams

### WS1 — AuthenSee: register the trust-graph provider (staging + prod)
- Provider creation is via the **admin backend** (`POST /api/providers` on `authnc-admin`; `POST /v1/providers` is retired — see `providers/routes.ts` comment). Create `Trust Graph (staging)` and `Trust Graph (production)`.
- **Audience contract (critical):** exchanged token `aud = providerId`; trust graph checks `audience === AUTHENSEE_JWT_AUDIENCE`. Either (a) create the provider so its **id literally equals** `trust-graph-staging` / `trust-graph-production`, or (b) accept the generated id and set `AUTHENSEE_JWT_AUDIENCE` to that id. Record the decision.
- Provider secret key: staging key already supplied by Gonçalo (stashed at `/home/openclaw/.openclaw/credentials/trust-graph-provider-key.env`, pending 1Password). Mint the prod key separately. Store both in 1Password (`Trust Graph Infra Staging` / `Production` → item `trust-graph-server` → field `AUTHENSEE_PROVIDER_API_KEY`).
- **Callback allowlist (fail-closed!):** `sessions/service.ts:assertCallbackUrlAllowed` rejects a supplied `callbackUrl` when the provider allowlist is empty. Add the trust-graph callback origins to the provider allowlist: `https://trust-staging.rebellion.systems`, `https://trust.rebellion.systems` (+ tailnet/localhost for dev). Without this, session create 400s.
- Confirm the provider's factor/scheme policy (which ceremony the human performs at bind).

### WS2 — Trust graph: own the AuthenSee exchange + bind flow (core work)
Mirror TRN's `provider-api.mjs` pattern inside `authnc-trust-graph`:
- **New config:** `AUTHENSEE_SERVER_URL`, `AUTHENSEE_PROVIDER_API_KEY`, `AUTHENSEE_CALLBACK_URL` (or derive from host), optional `AUTHENSEE_HOSTED_URL_BASE`.
- **New endpoints:**
  - `POST /v1/graph-keys/bind/start` — body `{ publicKeyJwk, label, nodeKind, metadata, ttlSeconds, returnUrl }`. Validate; persist a **pending bind** (`bindId`, short TTL); call AuthenSee `POST /v1/sessions` with the provider key (`callbackUrl = <trust-graph>/v1/authensee/callback?bind=<bindId>`). Return `{ bindId, hostedUrl }`.
  - `GET /v1/authensee/callback` — receive `authResultCode` + `bind`. Call AuthenSee `POST /v1/auth-results/exchange` with the provider key → `{ token, providerSubject }`. Complete the pending bind: `store.registerGraphKey({ authenseeSubject: providerSubject, publicKeyJwk, … })`. Render a relay page (`postMessage`/BroadcastChannel, like TRN's callback) or redirect to `returnUrl`.
  - `GET /v1/graph-keys/bind/:bindId` — poll status → `{ status, graphKey? }`.
- **Internal auth:** derive the subject **directly from the exchange response's `providerSubject`** (already returned) — no need to round-trip/verify our own JWT internally, since it's our own provider call over TLS with our key. This means the raw `Bearer /v1/graph-keys` path is no longer needed for humans.
  - **Decision:** keep `resolveSubject`/JWKS verify only if we still want to accept externally-minted `aud=trust-graph-*` tokens (future token-exchange, or agent/programmatic nodes). Otherwise retire it. Recommend: **keep for agent/programmatic nodes, route humans through `bind/start`.**
- **Pending-bind store:** use the existing **Neon** persistence (small table) rather than in-memory, so it survives multi-instance / restarts.
- Keep `AUTHENSEE_DEV_AUTH=false` and `NODE_ENV=production` everywhere (kills the dev-header path entirely).
- **Impl detail to verify:** confirm AuthenSee preserves the pre-existing `?bind=` query when appending its params to `callbackUrl`; if not, thread `bindId` via a path segment or an AuthenSee `state` param.

### WS3 — TRN: become a pure client (remove all special-relationship code)
`the-real-network/server/provider-api.mjs`:
- **Delete** `trustGraphAuthenSeeJwt` + `Authorization` injection.
- **Delete** `x-authensee-subject` injection + `isLocalTrustGraph()`.
- **Node registration** (`/api/trust-graph/graph-keys`): stop forwarding `authResult` as subject. Route registration through the trust graph's `bind/start` flow — TRN forwards `publicKeyJwk` etc., returns `{ bindId, hostedUrl }`; the app opens `hostedUrl`; trust-graph callback completes the bind; app polls `bind/:bindId`. TRN never handles trust-graph auth.
- **Writes** (`edges/request|confirm|revoke`, `graph-keys/:id/renew|revoke`) are device-key-signed → pass through unchanged; remove the `authResult`→subject fallback injection.
- **Reads** (`edges`, `graph/epoch/current`, `query/reachable`, `proofs/verify`) unchanged (open).
- **`graph-keys/me`** currently authorizes a subject-scoped list via injected subject — in the decoupled model this needs a real authorization decision (device-key-signed "list my keys", or a bind-session-scoped read). **Open decision.**
- Remove `TRUST_GRAPH_AUTHENSEE_JWT` from `.env.example` + deploy env; keep `TRUST_GRAPH_SERVER_URL`.
- `src/trust-graph/client.ts` + `browser-identity.ts`: adjust `registerGraphKey` to the `bind/start → open hostedUrl → poll` flow.

### WS4 — (follow-up, optional) generic public token-exchange (RFC-8693 style)
Only as a **standard, public** AuthenSee capability offered to any provider: allow `/v1/auth-results/exchange` (or a token endpoint) to accept an optional **target audience** and mint `aud=<target>` when the requesting provider is authorized. Enables single-login → trust-graph-audienced token (one tap instead of a separate bind ceremony). Must not be a bespoke AuthenSee↔trust-graph handshake. Deferred.

---

## 4. Config matrix

| Key | trust-graph staging | trust-graph prod |
|---|---|---|
| `NODE_ENV` | production | production |
| `AUTHENSEE_DEV_AUTH` | false | false |
| `AUTHENSEE_SERVER_URL` | https://api-staging.authensee.com | https://api.authensee.com |
| `AUTHENSEE_JWKS_URL` | https://api-staging.authensee.com/.well-known/jwks.json | https://api.authensee.com/.well-known/jwks.json |
| `AUTHENSEE_JWT_ISSUER` | https://api-staging.authensee.com | https://api.authensee.com |
| `AUTHENSEE_JWT_AUDIENCE` | trust-graph-staging (or assigned providerId) | trust-graph-production (or assigned providerId) |
| `AUTHENSEE_PROVIDER_API_KEY` | `sk_…` (supplied) | (mint) |
| `AUTHENSEE_CALLBACK_URL` | https://trust-staging.rebellion.systems/v1/authensee/callback | https://trust.rebellion.systems/v1/authensee/callback |
| `TRUST_GRAPH_DATABASE_URL` | Neon staging (pooled) | Neon prod (pooled) |

TRN: drop `TRUST_GRAPH_AUTHENSEE_JWT`; keep `TRUST_GRAPH_SERVER_URL` + `AUTHENSEE_PROVIDER_API_KEY` (TRN's own provider key, unchanged).

---

## 5. Rollout / sequencing
1. WS1 staging: register provider, set audience contract, callback allowlist, key → 1Password.
2. WS2 on `trust-graph-staging`: build bind/start + callback + exchange; deploy; keep old Bearer path temporarily behind a flag.
3. WS3 on TRN staging: wire bind flow; remove all auth injection.
4. E2E on `social.rebellion.systems`: create handle → register node → request/confirm edge → reachability.
5. Repeat WS1→WS3 for production.
6. WS4 later (UX single-tap).

## 6. Verification
- Unit: trust-graph `bind/start` + callback exchange (mock AuthenSee), subject derivation, pending-bind TTL/replay.
- Integration: real staging AuthenSee ceremony; assert graphKey bound to `providerSubject`; assert TRN sends no Bearer / no dev-header.
- Negative: expired auth-result code (10-min TTL), replayed `bindId`, wrong-audience token rejected, empty callback allowlist rejected.
- Guard: repo grep confirms no `TRUST_GRAPH_AUTHENSEE_JWT` / `x-authensee-subject` remain.

## 7. Open decisions (need Gonçalo)
1. Provider **id == `trust-graph-*`** (if admin allows custom id) vs generated id + set `AUTHENSEE_JWT_AUDIENCE`.
2. Is the supplied `sk_…` the **staging** key? (assumed yes)
3. `graph-keys/me` authorization model post-decoupling (device-key-signed list?).
4. Human bind UI: **TRN opens the trust-graph `hostedUrl` directly** (recommended) vs trust graph hosts its own pages.
5. Keep raw `Bearer /v1/graph-keys` for agent/programmatic nodes, or remove entirely.

## 8. Risks
- Multi-instance pending-bind store → must use Neon, not memory.
- Callback allowlist fail-closes → must be configured before first session.
- Per-provider subject divergence → TRN correlation via persona, not subject.
- Auth-result code TTL = 10 min (bind must complete promptly); graph-key TTL 5m–7d, 30d max lifetime.
