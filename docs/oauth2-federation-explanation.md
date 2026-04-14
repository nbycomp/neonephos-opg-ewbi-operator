# OAuth2-Authenticated Federation: Architecture Explanation

## Overview

This document describes how two instances of the OPG EWBI operator communicate with each other, and how OAuth2 authentication is introduced into every inter-operator interaction — both the forward path (Guest → Host) and the callback path (Host → Guest).

The design keeps changes within existing controllers and CRDs. No new controllers or custom resources are introduced.

---

## Current State (as-is)

Today, the operator instances rely on:

- **`X-Client-ID` header** — added to every outbound EWBI API request (in `internal/opg/opg.go`), but not cryptographically verified.
- **`GuestPartnerCredentials`** — carries a `clientId` and `tokenUrl` in the Federation spec, but no actual OAuth2 token exchange is performed; `tokenUrl` is used as the EWBI API base URL.
- **`Partner.CallbackCredentials`** — carries a `clientId` and optionally a `tokenUrl` for the callback direction, also not performing real token exchange.
- **No reverse-proxy** — the EWBI API is called directly with no enforcement point.
- **Credentials in plain text** — `clientId` values sit in the Federation CR spec, visible to anyone with read access to the namespace.

---

## Proposed State (to-be)

### Key Principles

1. **OAuth2 Client Credentials flow** for machine-to-machine authentication in both directions.
2. **Identity Provider per operator** — each operator platform deploys its own IdP (Keycloak, ORY Hydra, or any standards-compliant OIDC provider). The IdP issues and verifies tokens for its own domain.
3. **Reverse-proxy as enforcement point** — each operator places an API-aware reverse proxy (ORY Oathkeeper, Envoy with ext_authz, or similar) in front of the EWBI API. The proxy validates tokens by introspecting/verifying against the local IdP before forwarding requests.
4. **Secrets in K8s Secrets** — `client_secret` values are stored exclusively in Kubernetes Secrets. Controllers reference them by secret name/key. The Federation CR spec carries only the `clientId` and IdP token endpoint URL — never the secret itself.
5. **No new controllers or CRDs** — all changes happen inside the existing federation, file, artefact, application and application-instance reconcilers, plus the OPG client factory.

---

## Architecture Components

### Per Operator Platform

| Component | Role |
|---|---|
| **Identity Provider** (Keycloak / ORY Hydra / …) | Issues OAuth2 tokens (`/token` endpoint), introspects/verifies tokens, manages client registrations. |
| **Reverse-Proxy** (Oathkeeper / Envoy / …) | Sits in front of the EWBI API. Intercepts every inbound request, validates the `Authorization: Bearer` token against the local IdP, rejects unauthenticated/unauthorized calls. |
| **EWBI API** (`opg-ewbi-api`) | The GSMA EWBI REST API implementation. Receives only pre-authenticated traffic from the proxy. |
| **OPG EWBI Operator** | The Kubernetes operator. Controllers perform outbound EWBI calls (guest path) and outbound callbacks (host path). |
| **K8s Secrets** | Store `client_secret` values. One secret per federation relationship. |

---

## Interaction Phases

### Phase 0 — Client Registration (pre-federation, out-of-band)

Before any federation is established, each side must register an OAuth2 client with the _other_ side's IdP:

1. **Guest registers with Host's IdP** → receives `client_id` + `client_secret` for forward-path calls.  
2. **Host registers with Guest's IdP** → receives `callback_client_id` + `callback_client_secret` for callback calls.

These credentials are stored in Kubernetes Secrets on the respective operator's cluster. The Federation CR spec references them:

- `spec.guestPartnerCredentials.clientId` + `spec.guestPartnerCredentials.tokenUrl` (Host's IdP token endpoint)
- `spec.partner.callbackCredentials.clientId` + `spec.partner.callbackCredentials.tokenUrl` (Guest's IdP token endpoint)

> **Requirement (new):** The federation controller (or an out-of-band provisioning step) must ensure client registration is complete and secrets are populated before reconciliation begins. This can be manual initially and automated later (e.g., via OIDC Dynamic Client Registration RFC 7591).

> **Requirement (new):** Add a `secretRef` field to `FederationCredentials` so that `client_secret` is read from a K8s Secret instead of being embedded in the CR. The current `clientId` field stays in the spec; only the secret portion moves.

---

### Phase 1 — Federation Creation (Guest → Host)

**Controller:** `FederationReconciler` (guest)

1. Reconcile loop picks up a new Federation CR with `relation=guest`.
2. Controller reads `client_secret` from the referenced K8s Secret.
3. Controller calls **Host's IdP** `POST /token` with `grant_type=client_credentials`, `client_id`, `client_secret`, and the appropriate `scope` (e.g., `ewbi:federation`).
4. IdP returns an `access_token` (JWT or opaque).
5. Controller calls **Host's Reverse-Proxy** `POST /partner` (CreateFederation) with `Authorization: Bearer <access_token>` and `X-Client-ID`.
6. The proxy validates the token against Host's IdP (introspection or JWT signature verification).
7. On success, the request reaches the EWBI API, which creates the host-side Federation CR and returns `federationContextId` + `offeredAvailabilityZones`.
8. Guest controller updates its Federation CR status.

### Phase 2 — Zone Subscription (Guest → Host)

Same OAuth2 flow. In the next reconcile cycle, the guest controller acquires a token and calls `PUT /partner/{fedCtxId}/zone` (ZoneSubscribe) through the proxy.

### Phase 3 — File Upload (Guest → Host)

**Controller:** `FileReconciler` (guest)

Identical token-acquisition pattern. The guest controller resolves the parent Federation by `federation-context-id` index, reads credentials from the Secret, gets a token from Host's IdP, and calls `POST /partner/{fedCtxId}/file` (UploadFile, multipart) through the proxy.

### Phase 3b — File Status Callback (Host → Guest)

**Controller:** `FileReconciler` (host)

1. A host-side File CR changes state (via external PATCH — this is the WIP status-setting mechanism).
2. Host controller reads **callback** credentials from its own Secret (the secret for Guest's IdP).
3. Calls **Guest's IdP** `POST /token` with `callback_client_id`, `callback_client_secret`, `scope=ewbi:callback`.
4. Calls **Guest's Reverse-Proxy** at `statusLink` URL: `POST /cb/{clientId}/file/status` (FileStatusCallbackLink) with `Authorization: Bearer <callback_access_token>`.
5. Guest's proxy validates the token against Guest's IdP.
6. Callback payload reaches Guest's EWBI API, which patches the guest-side File CR status.

### Phases 4, 4b, 5, 5b, 6, 6b

Artefact, Application, and ApplicationInstance follow the exact same two-leg pattern:

- **Forward path (guest → host):** Token from Host's IdP → call through Host's proxy.
- **Callback path (host → guest):** Token from Guest's IdP → call through Guest's proxy.

### Phase 7 — Deletion

On `DeletionTimestamp`, guest controllers acquire a token and call the corresponding `DELETE` endpoint through Host's proxy, same auth flow.

---

## Changes Required in This Codebase

### 1. `FederationCredentials` — add `SecretRef`

```go
type FederationCredentials struct {
    ClientId  string `json:"clientId"`
    TokenUrl  string `json:"tokenUrl,omitempty"`
    // NEW: reference to a K8s Secret holding the client_secret
    SecretRef *SecretKeyRef `json:"secretRef,omitempty"`
}

type SecretKeyRef struct {
    Name string `json:"name"`
    Key  string `json:"key"`
}
```

This ensures `client_secret` is never stored in the Federation CR itself.

### 2. OPG Client Factory (`internal/opg/opg.go`) — add token exchange

The `GetOPGClient` method currently uses `tokenUrl` as the EWBI API base URL. It needs to:

1. Accept a separate `apiUrl` (EWBI endpoint) and `tokenUrl` (IdP token endpoint).
2. Before creating the HTTP client, perform `POST /token` (client credentials grant) against `tokenUrl`.
3. Attach the resulting `access_token` as `Authorization: Bearer` header via a request editor.
4. Cache tokens and refresh them before expiry (standard `oauth2.TokenSource` pattern from `golang.org/x/oauth2/clientcredentials`).
5. Continue adding `X-Client-ID` as today.

```go
import "golang.org/x/oauth2/clientcredentials"

// Token source per federation key, reuses tokens until expiry
tokenConfig := &clientcredentials.Config{
    ClientID:     clientId,
    ClientSecret: clientSecret,     // read from K8s Secret
    TokenURL:     tokenUrl,         // IdP endpoint
    Scopes:       []string{scope},
}
tokenSource := tokenConfig.TokenSource(ctx)

opts := []opgc.ClientOption{
    opgc.WithRequestEditorFn(func(ctx context.Context, req *http.Request) error {
        token, err := tokenSource.Token()
        if err != nil {
            return err
        }
        req.Header.Set("Authorization", "Bearer "+token.AccessToken)
        req.Header.Set("X-Client-ID", clientId)
        return nil
    }),
}
```

### 3. Controllers — read secrets, pass separate URLs

Each controller that calls `GetOPGClient` needs to:

1. Read the `client_secret` from the K8s Secret referenced by the Federation's `secretRef`.
2. Pass the **IdP token URL** and the **EWBI API URL** (or callback `statusLink`) as separate parameters.
3. This applies to both the forward path (using `guestPartnerCredentials`) and the callback path (using `partner.callbackCredentials`).

The reconciler structs already have `client.Client` embedded, so reading Secrets requires no new dependencies.

### 4. Federation spec — separate `apiUrl` from `tokenUrl`

Currently `guestPartnerCredentials.tokenUrl` is used as the EWBI API base URL. These should be split:

- `guestPartnerCredentials.tokenUrl` → IdP token endpoint (e.g., `https://host-idp.example.com/oauth2/token`)
- A new field or label for the actual EWBI API URL (e.g., `spec.partner.apiUrl` or reuse the existing label `FederationGuestUrlLabel`)

Similarly for callbacks:

- `partner.callbackCredentials.tokenUrl` → Guest's IdP token endpoint
- `partner.statusLink` → Guest's EWBI callback URL (already exists)

### 5. Reverse-Proxy Deployment (infrastructure, not code)

Each operator platform must deploy a reverse-proxy in front of its EWBI API service. The proxy:

- Intercepts all inbound HTTP requests to the EWBI API.
- Validates `Authorization: Bearer` tokens by calling the local IdP's introspection endpoint or verifying JWT signatures.
- Rejects requests with missing, expired, or invalid tokens (401/403).
- Forwards authenticated requests to the EWBI API backend.

This is a deployment concern (Helm chart / Kustomize overlay) and does not require operator code changes.

---

## Credential Flow Summary

| Direction | Who authenticates | Against which IdP | What is called | Through which proxy |
|---|---|---|---|---|
| Guest → Host (forward) | Guest controller | **Host's** IdP | Host EWBI API | **Host's** reverse-proxy |
| Host → Guest (callback) | Host controller | **Guest's** IdP | Guest callback endpoint | **Guest's** reverse-proxy |

---

## Security Properties

1. **No plain-text secrets in CRs** — `client_secret` lives only in K8s Secrets, referenced by name.
2. **Short-lived tokens** — OAuth2 tokens have configurable TTL; the `clientcredentials.TokenSource` handles refresh automatically.
3. **Scope-based authorization** — different EWBI operations can require different scopes, enforced by the IdP and proxy.
4. **Mutual distrust** — each side validates tokens using its own IdP. A compromised guest cannot forge host tokens or vice versa.
5. **Proxy as choke point** — even if the EWBI API has no auth logic, the proxy blocks unauthenticated traffic.
6. **Callback authentication** — host-to-guest callbacks are authenticated with the same rigor as forward calls, preventing spoofed status updates.

---

## Open Items / WIP

| Item | Status | Notes |
|---|---|---|
| Status set externally via PATCH | WIP (dev-callback branch) | Host-side CRs have their status changed by an external actor. Callbacks fire on the next reconcile. The PATCH endpoint itself should also be behind the reverse-proxy and require a valid token. |
| Client registration automation | TODO | Currently assumed out-of-band. Can be automated via OIDC Dynamic Client Registration (RFC 7591) in the federation controller. |
| Token caching in OPG client | TODO | Replace current simple client map with `oauth2.TokenSource`-backed clients. |
| `SecretRef` in `FederationCredentials` | TODO | Add field to CRD, update deepcopy, update all controllers. |
| Split `tokenUrl` / `apiUrl` | TODO | Separate IdP token URL from EWBI API URL in the Federation spec. |
| Reverse-proxy deployment manifests | TODO | Add Kustomize overlays or Helm values for Oathkeeper/Envoy sidecar. |
| Scope definitions | TODO | Define which scopes map to which EWBI operations. |
