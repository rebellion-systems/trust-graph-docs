# Trust Graph architecture, threat model, and integration plan

Status: draft plan  
Date: 2026-07-10  
Scope: trust graph service, trust graph SDK, AuthenSee integration, and app/front-end integration patterns

## Executive answer

The trust graph should not have its own product UI.

It should be a product API plus SDK. The Real Network, recovery products, repo tools, agent dashboards, and future apps should render their own UX and consume the trust graph through the public SDK. The trust graph may need tiny machine endpoints for callbacks, polling, health, and operations, but not a separate user-facing app.

AuthenSee integration should also stay boring and public-contract-shaped: the trust graph is just another AuthenSee provider. It has its own provider id and provider secret. It starts an AuthenSee session when a user needs to bind or rebind a graph key, exchanges the one-time auth result code with its own provider key, receives a provider-scoped `providerSubject`, and stores that as the graph subject. After that, graph mutations are authorized by graph-key signatures, not by AuthenSee sessions and not by another app's credentials.

## Non-negotiable design rule

No special relationships between AuthenSee, The Real Network, and the trust graph.

That means:

- No `AUTHENSEE_DEV_AUTH=true` outside local tests.
- No `x-authensee-subject` bypass in deployed environments.
- No TRN-held static trust graph write credential.
- No TRN server proxy that authenticates as the trust graph or for the user.
- No raw AuthenSee global persona id as a cross-product public identity.
- No bespoke AuthenSee-to-trust-graph audience exchange unless it is a generic public AuthenSee feature available to any provider.

The acceptable relationship is:

- AuthenSee offers ordinary provider APIs.
- Trust graph consumes those APIs as a normal provider.
- Apps consume the trust graph public API/SDK.
- Users/devices sign graph actions with graph keys.

## Current state to preserve

The current trust graph already has the right skeleton:

- `POST /v1/graph-keys` binds a public graph key to an AuthenSee subject.
- Graph keys are public P-256 JWKs.
- Graph key id is `sha256(canonicalJson({ crv, kty, x, y }))`.
- Graph key TTL defaults to 7 days.
- Independent graph-key lifetime is capped at 30 days since `authBoundAt`.
- Signed graph actions have:
  - action type
  - `graphKeyId`
  - optional `edgeId`
  - nonce
  - `signedAt`
  - signature over canonical JSON with version tag `trust-graph-action-v1`
- Signed actions have a 5-minute freshness window.
- Signed-action nonces are single-use per graph key and retained for 15 minutes.
- Edge creation is signer-key signed.
- Edge confirmation can be subject-key signed.
- Edge revocation can be signer-key signed.
- Public reads and reachability queries do not require bearer auth.
- The SDK already intentionally avoids sending bearer tokens for public reads.
- The SDK exposes metadata fields for human/agent nodes and provider-safe derivatives.

What is not right yet:

- Some docs still describe bearer JWTs as the main authenticated path.
- Some code still has dev-header behavior for non-production/local use.
- TRN has its own local trust graph client instead of consuming the SDK as the primary integration contract.
- The graph-key bind flow is not a first-class trust graph flow yet.
- The current auth fields are too AuthenSee-internal-shaped and need sharper naming.
- `dev-reachability-v0` is not a production ZK proof.

## Target architecture

### Components

1. AuthenSee

AuthenSee remains the portable proof/auth product. It owns personas, proof verification, provider-scoped sessions, result-code exchange, provider-scoped `providerSubject`, JWKS, token signing, and factor policy.

2. Trust graph service

The trust graph owns graph keys, edges, reachability, proofs, graph roots, event roots, nonces, TTLs, and persistence. It is an AuthenSee provider only for bind/rebind ceremonies.

3. Trust graph SDK

The SDK is the integration surface for all apps. It should expose both server-safe and browser-safe entry points:

- HTTP client
- browser graph-key generation/storage helpers
- canonical signing helpers
- bind/rebind ceremony helpers
- edge mutation helpers
- reachability/proof helpers
- strict mappers and errors

4. Product apps/frontends

Apps render UX. Examples: TRN, recovery UI, repo-maintenance UI, agent-control UI. They should call the SDK and hold only their own app credentials. They do not get the trust graph provider secret and do not hold privileged graph write auth.

### Boundaries

AuthenSee boundary:

- AuthenSee knows global personas.
- Providers see provider-scoped subjects.
- Auth result callbacks return one-time result codes, not JWTs and not raw persona ids.
- Provider backends exchange result codes with their own `sk_` keys.

Trust graph boundary:

- Trust graph sees only the trust-graph provider-scoped `providerSubject` from AuthenSee.
- Trust graph stores graph-visible metadata.
- Trust graph does not need TRN account ids to authorize graph writes.
- Trust graph does not call TRN private APIs.
- Trust graph public reads must stay usable by any independent app.

App boundary:

- Apps use the SDK.
- Apps may generate and hold local graph private keys for their users/devices.
- Apps may display graph state.
- Apps may request graph actions from users.
- Apps must not inject a user's graph identity using app-controlled headers.

## How the trust graph consumes AuthenSee personas

The trust graph consumes AuthenSee personas exactly as a normal AuthenSee provider.

### Provider setup

Create two AuthenSee providers:

- `trust-graph-staging`
- `trust-graph-production`

Each provider has:

- own provider id
- own `sk_` provider key
- own callback allowlist
- own branding/policy if needed
- own AuthenSee provider-scoped `providerSubject` namespace

The trust graph runtime stores only its own provider key, outside git, in the trust graph infra vault for that environment.

### Bind ceremony

The bind ceremony is the one-time act of attaching a device or agent graph key to a proven AuthenSee persona.

The target flow:

1. Frontend generates or loads a local graph keypair.
2. Frontend calls trust graph SDK `startGraphKeyBind({ publicKeyJwk, nodeKind, label, metadata })`.
3. Trust graph service creates a pending bind record:
   - `bindId`
   - graph key id
   - public key JWK
   - requested node kind
   - metadata
   - expiration
   - callback URL
   - nonce/state
4. Trust graph service calls AuthenSee `POST /v1/sessions` using its own provider key.
5. AuthenSee returns a `hostedUrl` or SDK flow bootstrap.
6. Frontend completes AuthenSee proof using AuthenSee's normal hosted/SDK experience.
7. AuthenSee redirects/calls back with a one-time `authResultCode`.
8. Trust graph callback exchanges that code using the trust graph provider key.
9. AuthenSee returns:
   - `providerSubject`
   - `externalUserId`, if any
   - `personaType`
   - `schemeId`
   - `authTime`
   - `factorsVerified`
   - `token`, if retained for verification/audit during request handling
10. Trust graph verifies/accepts the result for the pending bind.
11. Trust graph stores `authenseeSubject = providerSubject`.
12. Trust graph registers or rebinds the graph key.
13. Frontend polls or receives completion and stores the graph key id.

The raw AuthenSee global `persona_id` should not be required in this flow. In AuthenSee's own docs, provider-visible boundaries should use `providerSubject` and result codes, not raw persona ids.

### What to store on a graph key

Current field names exist, but the target model should make the data model clearer:

- `graphKeyId`: public graph-key id.
- `authenseeSubject`: the trust-graph-provider-scoped `providerSubject`. This is the authorization subject.
- `nodeKind`: `human` or `agent`.
- `publicKeyJwk`: public P-256 JWK only.
- `label`: non-secret local display label.
- `authBoundAt`: when AuthenSee last proved/reproved this binding.
- `expiresAt`: short graph-key TTL.
- `revokedAt`: optional.
- `authResultId`: optional audit pointer to the exchanged auth result, if safe and useful.
- `authSchemeId`: optional scheme id, if safe and useful.
- `authPersonaType`: human/agent result classification.
- `metadata`: non-secret graph-visible metadata.
- `providerPersonaIdDerivative`: optional app-specific derivative supplied by an app only when product value justifies it.

Avoid:

- raw AuthenSee `persona_id`
- recovery answers
- factor commitments
- face/liveness data
- passkey credential ids
- private graph keys
- bearer tokens
- app access tokens
- email/phone unless explicitly product-reviewed

## Does the trust graph need its own UI?

No, not as a product.

It needs:

- API endpoints.
- SDK docs.
- API reference.
- health/status endpoints.
- maybe an internal/admin ops view later for metrics, event replay, and incident response.

It does not need:

- a consumer dashboard
- profile pages
- a social UI
- graph browsing UI as a separate product
- AuthenSee-branded login pages owned by the trust graph

TRN should be one frontend for the trust graph. Other apps can be other frontends. The trust graph remains the shared substrate.

The bind flow can still require trust-graph backend callback endpoints. That is not a UI. It is an API/control-plane callback so the trust graph can exchange its own AuthenSee result codes with its own provider key.

## Integration contract for other apps

All apps integrate through the SDK.

### SDK initialization

```ts
import { createTrustGraphSdk } from "@rebellion-systems/trust-graph-sdk";

const trustGraph = createTrustGraphSdk({
  serverUrl: "https://trust.rebellion.systems",
});
```

Most app-facing operations should not configure a bearer token. The browser/app should use graph-key signatures after binding.

### Browser/frontend responsibilities

Frontends should:

- generate or retrieve a local graph keypair
- keep the private key in the strongest available local boundary
- call SDK bind/rebind helpers
- sign edge certificates
- sign graph actions
- display public graph reads
- ask the user before confirming or revoking meaningful edges

Frontends should not:

- hold trust graph provider secrets
- hold AuthenSee provider secrets
- inject identity headers
- store AuthenSee JWTs in local storage
- treat metadata as private
- render `dev-reachability-v0` as a production ZK guarantee

### Backend responsibilities

App backends may:

- host their own app data
- call public trust graph reads
- verify trust graph proof outputs
- generate agent graph keys for their own agents
- store agent private keys in server/HSM boundaries

App backends should not:

- authenticate to the trust graph as the user
- hold a trust graph provider key
- exchange trust graph AuthenSee result codes
- mint trust graph sessions

### Trust graph SDK additions needed

Add a browser-safe entry point:

```ts
import {
  createTrustGraphSdk,
  createBrowserGraphKey,
  loadBrowserGraphKey,
  startGraphKeyBind,
  signEdgeWithBrowserKey,
  signGraphActionWithBrowserKey,
} from "@rebellion-systems/trust-graph-sdk/browser";
```

Add first-class methods:

- `startGraphKeyBind(input)`
- `getGraphKeyBind(bindId)`
- `completeGraphKeyBind(input)` only if a popup/mobile callback returns to the app instead of directly to the trust graph
- `listGraphKeysBySignedAction(input)` or `listGraphKeysBySubjectProof(input)`, replacing `listMyGraphKeys` bearer expectations
- `renewGraphKey(graphKeyId, signedAction, { rebind?: boolean })`
- `rebindGraphKey(graphKeyId)` as a bind ceremony wrapper

Keep public methods credential-free:

- `listEdges`
- `getCurrentEpoch`
- `queryReachability`
- `verifyProof`

## Core flows

### Flow A: First device bind

Actor: human using TRN or another frontend.

1. App loads SDK.
2. App creates local graph keypair.
3. App calls `startGraphKeyBind`.
4. Trust graph creates pending bind.
5. Trust graph starts AuthenSee session as provider `trust-graph-*`.
6. User completes AuthenSee proof.
7. Trust graph exchanges result code.
8. Trust graph binds public key to `providerSubject`.
9. App receives graph key id.
10. App can now sign graph actions.

Security result: AuthenSee proves the persona only at bind time; graph private key controls routine graph actions after bind.

### Flow B: Edge request

Actor: signer node.

1. App chooses `subjectGraphKeyId`, scope, trust score, expiry, nonce.
2. App canonicalizes edge payload.
3. App signs payload with signer graph private key.
4. App calls `requestEdge`.
5. Trust graph verifies signer key exists, is not revoked, is not expired, and signature is valid.
6. Trust graph creates pending edge.

Security result: only holder of signer graph private key can request an outgoing edge.

### Flow C: Edge confirmation

Actor: subject node.

1. Subject app sees pending edge.
2. Subject signs `confirm-edge` action with subject graph key.
3. App calls `confirmEdge(edgeId, { signedAction })`.
4. Trust graph verifies action freshness, nonce, signature, edge id, and subject key id.
5. Trust graph marks edge active.

Security result: a trust edge requires signer intent and subject consent.

### Flow D: Edge revocation

Actor: signer node.

1. Signer signs `revoke-edge`.
2. Trust graph verifies signer key controls the edge.
3. Trust graph marks edge revoked.

Optional future: subject-side "reject/remove incoming edge" can exist as a separate action type, but it should not mean the signer revokes trust; it means the subject hides or refuses association.

### Flow E: Key renewal

Actor: bound device.

1. Device signs `renew-graph-key`.
2. Trust graph verifies key signature.
3. If still inside independent lifetime, trust graph renews short TTL.
4. If beyond independent lifetime or already expired, trust graph requires rebind via AuthenSee.

Security result: stolen keys expire quickly; long-lived continuity requires periodic AuthenSee proof.

### Flow F: Reachability query

Actor: any app.

1. App asks if source reaches target under policy.
2. Trust graph evaluates active edges only.
3. Trust graph returns `reachable`, path length, public inputs, and proof envelope/proof.
4. App may call `verifyProof`.

Security result: reads are public, but proof semantics must be honest. Today the proof is only a development envelope.

## Threat model

### Assets

- Trust graph provider secret key in AuthenSee.
- AuthenSee signing keys/JWKS.
- AuthenSee provider-scoped `providerSubject`.
- Graph private keys on devices/servers.
- Public graph keys.
- Graph-key binding records.
- Edges and edge status.
- Event log root.
- Graph epoch root.
- Action nonces.
- Persistence database.
- Reachability proof/verifier keys.
- SDK package integrity.
- Frontend bundle integrity.

### Actors

- Honest human user.
- Honest app frontend.
- Honest app backend.
- Honest agent.
- Malicious user.
- Malicious app integrator.
- Compromised frontend bundle.
- Compromised browser/device.
- Compromised app backend.
- Compromised trust graph server.
- Compromised AuthenSee provider key.
- Network attacker.
- Graph spammer/sybil.
- Curious third party scraping public graph reads.

### Trust boundaries

1. Browser/device boundary

Graph private keys live here for human devices. This boundary is vulnerable to XSS, malicious extensions, local malware, and insecure storage.

2. App backend boundary

App backends own app credentials and app data. They do not own trust graph identity.

3. Trust graph service boundary

Trust graph owns binding, graph state, graph auth rules, and proof generation.

4. AuthenSee boundary

AuthenSee owns proof verification and provider-scoped identity assertions.

5. Public network/API boundary

Public reads and signed writes cross this boundary. Anything in metadata should be treated as public.

### Threats and mitigations

| Threat | Impact | Mitigation | Status |
| --- | --- | --- | --- |
| App injects `x-authensee-subject` to impersonate a user | Full graph-key bind impersonation | Remove deployed dev header path; enforce JWT/provider bind flow only | Required |
| TRN holds trust graph privileged credential | TRN can write as users or TG | No static trust graph credential in app env; use SDK + graph-key signatures | Required |
| AuthenSee token audience confusion | Token for one provider accepted by another | Trust graph verifies `iss`, `aud=trust-graph-*`, `sub`; AuthenSee mints `aud=providerId` | Partly present |
| Trust graph provider key leaks | Attacker can start/exchange TG AuthenSee sessions | 1Password/secret manager, scoped provider, rotation runbook, audit logs, rate limits | Required |
| Auth result code replay | Bind without fresh proof | AuthenSee result codes are single-use and TTL-bound | Present in AuthenSee |
| Bind CSRF/state mixup | Attacker binds victim proof to attacker's key | Pending bind id, state nonce, key id, callback origin allowlist, short bind TTL | Required |
| Graph private key theft | Attacker signs edges/actions | Short graph-key TTL, 30-day independent cap, revoke/rebind, secure storage guidance | Partly present |
| XSS steals browser key | Same as private key theft | CSP, no inline scripts where possible, key extractability review, WebAuthn/secure enclave path later | Required per app |
| Signed action replay | Duplicate confirm/revoke/renew | Nonce per key, freshness window, persisted recent nonces | Present |
| Edge signature tampering | Fake trust edge | Canonical payload, P-256 signature verification | Present |
| Duplicate active edge spam | Ambiguous trust state | Unique active/pending edge per signer/subject/scope | Present |
| Self-trust edge | Inflated graph path | Reject signer == subject | Present |
| Expired/revoked key still mutates | Stale authority | `assertKeyUsable` for mutations; signed action verification checks revoked | Partly present; audit all paths |
| Metadata leaks private info | Privacy breach | Metadata schema guidance, lint/tests for forbidden fields, docs | Required |
| Raw persona id leaks across provider boundary | Cross-provider correlation | Store `providerSubject`; avoid raw `persona_id`; rename target fields | Required |
| Graph scraping | Public relationship enumeration | Treat graph as public; consider pagination, rate limits, privacy modes later | Accepted/residual |
| Sybil graph poisoning | Low-quality trust graph | Trust policy consumers choose roots/scopes/min trust; edge confirmation; rate limits | Required |
| Proof envelope misrepresented as ZK | False security claim | Label `dev-reachability-v0`; ship Noir/Barretenberg before production proof claims | Required |
| DB tampering | Forged graph state | Event roots, append-only log direction, backups, DB access controls, future signed checkpoints | Required |
| SDK supply-chain compromise | Apps sign/send malicious payloads | GitHub Packages controls, provenance, lockfiles, CI, signed releases later | Required |
| CORS abuse | Browser cross-origin write attempts | Signatures still required; bind endpoints need state/CSRF; narrow CORS for bind | Required |
| JWKS outage | Bind/rebind unavailable | JWKS cache behavior, retries, operational alerting | Required |
| Clock skew | Signed actions rejected | 5-minute freshness, app clock error UX, server time endpoint optional | Required |
| Agent key misuse | Automated high-volume graph mutation | Agent node kind, per-key/app rate limits, server/HSM storage, audit | Required |

## API changes recommended

### New bind endpoints

```http
POST /v1/graph-keys/bind/start
{
  "publicKeyJwk": {},
  "nodeKind": "human",
  "label": "Work laptop",
  "metadata": {}
}
→ {
  "bindId": "gkb_...",
  "graphKeyId": "...",
  "hostedUrl": "https://auth.authensee.com/flow/...",
  "expiresAt": "..."
}
```

```http
GET /v1/graph-keys/bind/:bindId
→ {
  "status": "pending" | "complete" | "failed" | "expired",
  "graphKey": {}
}
```

```http
GET /v1/authensee/callback?bindId=...&authResultCode=...
```

The callback may return tiny relay HTML for browser popup closure if needed, but it is not a trust graph UI.

### Replace bearer-centered methods

Deprecate direct `registerGraphKey` bearer usage for normal human flows. Keep it only if there is a strong agent/server use case and document it as advanced.

Replace `listMyGraphKeys` with one of:

- `listGraphKeys({ graphKeyId, signedAction })` for a device proving possession.
- `listGraphKeysForBindSubject` only during a fresh bind/rebind ceremony.
- public `listEdges({ graphKeyId })` remains enough for most app views.

### Keep signed action methods

Keep:

- `requestEdge`
- `confirmEdge` with signed action
- `revokeEdge` with signed action
- `renewGraphKey` with signed action
- `revokeGraphKey` with signed action

Make bearer fallback optional and eventually remove it from app-facing SDK examples.

## App integration examples

### The Real Network

TRN should be a frontend for the trust graph.

Target TRN shape:

- TRN imports `@rebellion-systems/trust-graph-sdk`.
- TRN stores local graph keypair in browser storage/secure storage.
- TRN starts bind through the SDK.
- TRN displays graph state through public reads.
- TRN signs edge actions locally.
- TRN does not proxy authenticated graph writes as a privileged server.
- TRN account handles remain TRN-local. They may be displayed in graph metadata only as public non-secret labels or derivatives.

### Recovery app

Recovery app uses the trust graph to model recovery relationships:

- scope: `recovery`
- trust score: policy-dependent
- edge expiry: recommended
- confirmation required from recovery contact
- reachability query checks enough active trusted path under recovery policy

AuthenSee proves the person at bind/rebind; recovery policy consumes graph proofs.

### Repo-maintenance app

Repo tools use the graph for maintainer/agent delegation:

- scope: `repo-maintenance` or `github`
- human maintainer nodes sign trust edges to agent nodes
- agent nodes use server-held graph keys
- CI/deploy tools query reachability or verify signed authority path

### Agent apps

Agents are graph nodes with `nodeKind=agent`.

Agent bind options:

- agent proves through AuthenSee agent-persona flow, if available
- or a human owner binds/attests the agent key under an explicit agent policy

Agent private keys should live in HSM/server secret stores, not browser local storage.

## Data model plan

### GraphKey

Target fields:

```ts
type GraphKey = {
  graphKeyId: string;
  authenseeSubject: string; // trust-graph providerSubject
  publicKeyJwk: JsonWebKey;
  nodeKind: "human" | "agent";
  label?: string;
  metadata?: Record<string, unknown>;
  authBoundAt: string;
  expiresAt: string;
  revokedAt?: string;
  authResultId?: string;
  authSchemeId?: string;
  authPersonaType?: "human" | "agent";
  providerPersonaIdDerivative?: string;
};
```

### TrustEdge

Current shape is reasonable:

```ts
type TrustEdge = {
  edgeId: string;
  signerGraphKeyId: string;
  subjectGraphKeyId: string;
  trust: number;
  scope: TrustScope;
  status: "pending" | "active" | "revoked";
  createdAt: string;
  expiresAt?: string;
  nonce: string;
  signature: string;
  confirmedAt?: string;
  revokedAt?: string;
};
```

Future addition:

- `updatedAt`
- optional `rejectedAt`
- optional `revocationReasonCode`
- optional `policyHints`, if consumers need structured hints

### Event log

Current event log stores event type, time, and payload hash. Future production hardening should move toward append-only event storage with replay and signed checkpoints.

## Proof plan

Current `dev-reachability-v0` is an integration envelope only.

Production plan:

1. Keep the API shape stable:
   - `queryReachability`
   - `verifyProof`
   - public inputs
   - proof system id
2. Add proof system version:
   - `wot-path-noir-v1`
3. Align edge certificate format with circuit witness requirements.
4. Build witness generation behind an internal proof provider interface.
5. Add verifier keys to deployment.
6. Keep dev proof behind explicit local/test flag.
7. Update SDK proof types to support multiple proof systems.
8. Update docs so apps cannot mistake dev envelopes for ZK proofs.

## Rate limits and abuse controls

Needed before broader use:

- bind start per IP/device
- bind completion per AuthenSee subject
- graph key count per subject
- edge request per graph key
- edge confirmation/revocation per graph key
- reachability query per IP/app
- public edge list pagination and caps
- metadata size caps
- label length caps
- pending bind expiry sweeper
- pending edge expiry/cleanup

## Observability and audit

Trust graph should emit structured events for:

- bind start
- bind complete
- bind failed
- graph key registered
- graph key rebound
- graph key renewed
- graph key revoked
- edge requested
- edge confirmed
- edge revoked
- proof generated
- proof verified
- auth failure
- nonce replay rejection
- stale signed action rejection

Never log:

- provider secrets
- bearer tokens
- private keys
- raw AuthenSee proof material
- raw recovery answers
- full unfiltered metadata if metadata can contain app-provided fields

## Implementation plan

### Phase 0: Lock the target contract

- Update trust graph docs to remove deployed dev-header guidance.
- State explicitly that deployed environments use ordinary AuthenSee provider auth for bind/rebind only.
- Mark bearer `registerGraphKey` as transitional.
- Document that apps must consume the SDK.
- Add a public "integration model" doc to `docs.trust.rebellion.systems`.

### Phase 1: SDK-first integration surface

- Move TRN from local trust graph client to SDK.
- Add browser-safe signing helpers.
- Add browser graph-key persistence helpers.
- Add DER encoding for browser WebCrypto signatures.
- Add bind/rebind method stubs/types, even before server support, to pin the API.
- Split SDK docs into:
  - frontend integration
  - backend integration
  - agent integration
  - security boundaries

### Phase 2: Trust graph bind service

- Add pending bind persistence table.
- Add `/v1/graph-keys/bind/start`.
- Add `/v1/graph-keys/bind/:bindId`.
- Add AuthenSee callback endpoint.
- Add server-side AuthenSee provider client.
- Exchange result code with trust graph provider key.
- Store `providerSubject` as `authenseeSubject`.
- Stop requiring apps to forward AuthenSee auth results into graph-key registration.
- Add tests for:
  - successful bind
  - expired bind
  - bind state mismatch
  - wrong provider result rejected
  - reused code rejected by AuthenSee
  - same key same subject idempotent
  - same key different subject rejected

### Phase 3: Remove privileged app auth paths

- Remove `TRUST_GRAPH_AUTHENSEE_JWT` from TRN.
- Remove TRN auth injection for graph writes.
- Remove non-local `x-authensee-subject` paths.
- Keep local test utilities isolated to test env.
- Update SDK examples to prefer bind + signed actions.

### Phase 4: Production hardening

- Add rate limits.
- Add pagination.
- Add metadata schema limits.
- Add provider key rotation runbook.
- Add audit dashboards.
- Add signed checkpoint/export job.
- Add backup restore tests.
- Add alerting for auth failures, nonce replays, bind spikes, proof errors.

### Phase 5: Real proof system

- Wire Noir/Barretenberg proof provider.
- Replace `dev-reachability-v0` in non-local environments.
- Add verifier endpoint support for proof versions.
- Add proof latency metrics.
- Add integration tests for 1-hop, 2-hop, 3-hop, expired edge, low-trust path.

## Open decisions

These are product/security decisions, not coupling questions:

1. Should public graph reads expose all edges globally, or should `listEdges` require a graph key filter once scale/privacy matters?
2. Should incoming edge confirmation be mandatory for all scopes, or can some scopes allow unilateral attestations?
3. Should subjects be able to reject/hide incoming pending edges without treating it as signer revocation?
4. Should human graph private keys be exportable browser keys initially, or should we move quickly to WebAuthn/secure-enclave-backed signing?
5. What exact metadata fields are allowed for TRN profile display?
6. What scopes ship first: `identity`, `recovery`, `real-network`, `github`, `repo-maintenance`?
7. What trust policy consumes reachability for recovery: min trust, max hops, expiry windows, required roots?

## Direct answers

### How does the trust graph integrate with AuthenSee?

As an ordinary AuthenSee provider. It starts an AuthenSee session for graph-key bind/rebind, receives a one-time result code, exchanges it server-to-server with its own provider key, and stores the returned trust-graph-provider-scoped `providerSubject` as the graph subject.

### How does the trust graph consume personas?

It does not consume raw global personas directly. It consumes provider-scoped AuthenSee results. The durable graph identity is `authenseeSubject = providerSubject` for the trust graph provider. That preserves AuthenSee's provider boundary and avoids cross-app special identity sharing.

### How do other apps integrate?

They use the trust graph SDK. They render their own UX, generate/hold graph keys for their own user/device/agent context, start bind through the SDK, sign graph actions locally or server-side depending on node type, and use public graph reads/proofs.

### Does the trust graph need its own UI?

No. TRN can be a frontend for the trust graph. Other apps can also be frontends. The trust graph itself should stay API/SDK/docs plus operational tooling.

