# Direct K8s Federation: Architecture Explanation

## Overview

This document describes an alternative architecture where two operator instances federate **directly via their Kubernetes API servers**, authenticated with OIDC (JWT from OAuth2 client credentials). This eliminates the EWBI REST API, the reverse-proxy, and the callback mechanism entirely.

---

## What changes vs. the EWBI API version

| Concern | EWBI API version | Direct K8s version |
|---|---|---|
| **Forward path** (Guest creates resources on Host) | Guest controller calls EWBI REST API through reverse-proxy | Guest controller calls Host K8s API directly using `client-go` |
| **Callback path** (Host notifies Guest of status) | Host controller calls Guest EWBI callback endpoint | Guest controller watches Host K8s resources for status changes |
| **Authentication** | OAuth2 token verified by reverse-proxy | OAuth2 JWT verified by K8s API server (native OIDC support) |
| **Authorization** | Scopes in token, proxy rules | K8s RBAC (ClusterRole/RoleBinding) |
| **Infrastructure** | IdP + reverse-proxy + EWBI API per platform | IdP per platform (K8s API server replaces proxy + API) |
| **GSMA EWBI compliance** | Full compliance | **Not compliant** — standard REST interface is removed |

---

## Architecture Components

### Per Operator Platform

| Component | Role |
|---|---|
| **K8s API Server** (with `--oidc-*` flags) | Serves as both data store and authenticated entry point. Validates JWTs from the remote party's IdP using JWKS. |
| **Identity Provider** (Keycloak / ORY Hydra / ...) | Issues JWTs via `client_credentials` grant. Publishes JWKS for API server to verify tokens. |
| **OPG EWBI Operator** | Controllers create CRs on the remote K8s and watch for status changes. Uses `client-go` with `oauth2.Transport`. |
| **K8s Secrets** | Store `client_secret` for the remote IdP. |
| **RBAC** | ClusterRole scoped to `opg.ewbi.nby.one` API group only. Binds to the OIDC `sub` claim of the remote party. |

**Eliminated components:** Reverse-proxy (Oathkeeper/Envoy), EWBI REST API service.

---

## How OIDC authentication works with K8s

### K8s API server configuration

The host K8s API server is configured to trust the host's IdP as an OIDC issuer:

```
# K8s 1.30+ supports multiple issuers via --authentication-config
--oidc-issuer-url=https://host-idp.example.com
--oidc-client-id=opg-ewbi-federation
--oidc-username-claim=sub          # maps JWT sub -> K8s username
--oidc-groups-claim=scopes         # maps JWT scopes -> K8s groups
```

### Request flow

1. Guest controller gets a JWT from Host's IdP (`POST /oauth2/token`, client credentials grant)
2. Guest controller calls Host's K8s API with `Authorization: Bearer <JWT>`
3. K8s API server validates the JWT signature using the IdP's JWKS endpoint (cached)
4. K8s API server extracts the `sub` claim as the username
5. K8s RBAC evaluates whether that username/group can perform the requested action

### RBAC example

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: opg-ewbi-guest
rules:
- apiGroups: ["opg.ewbi.nby.one"]
  resources:
  - federations
  - federations/status
  - files
  - files/status
  - artefacts
  - artefacts/status
  - applications
  - applications/status
  - applicationinstances
  - applicationinstances/status
  verbs: ["create", "get", "list", "watch", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: opg-ewbi-guest-binding
subjects:
- kind: User
  name: "3acde22c-d245-480d-b01e-24e38e01806d"  # OIDC sub claim = client_id
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: opg-ewbi-guest
  apiGroup: rbac.authorization.k8s.io
```

---

## Interaction Phases

### Phase 0 — Setup (out-of-band)

1. **Guest admin** registers an OAuth2 client on Host's IdP, receives `client_id` + `client_secret`, stores them in a K8s Secret, and references it in the Federation CR.
2. **Host admin** configures the K8s API server with `--oidc-issuer-url` pointing to Host's IdP, creates RBAC for the guest's OIDC identity (scoped to `opg.ewbi.nby.one` CRDs only).
3. **Host admin** registers an OAuth2 client on Guest's IdP for watching guest CRs (optional reverse direction).
4. **Guest admin** configures OIDC + RBAC for host to watch guest-side CRs (optional).

### Phase 1 — Federation Creation

The guest controller:
1. Reads credentials from K8s Secret
2. Gets a JWT from Host's IdP (`client_credentials` grant)
3. Calls Host K8s API: `POST /apis/opg.ewbi.nby.one/v1beta1/namespaces/ns/federations`
4. K8s API server validates the JWT and checks RBAC
5. Federation CR is created directly on Host's cluster
6. Guest updates its local Federation CR status

### Phase 2 — Zone Subscription

Guest controller patches the remote Federation's status to set `acceptedAvailabilityZones`. Same auth, same `client-go` call.

### Phase 3 — ApplicationInstance Install

Guest controller creates an ApplicationInstance CR directly on Host K8s via `POST /apis/.../applicationinstances`.

### Phase 3b — Status Sync (replaces callbacks)

Instead of Host sending callbacks to Guest:
1. Guest controller opens a long-lived **WATCH** on Host K8s for ApplicationInstance CRs (using the same OIDC-authenticated `client-go` client)
2. When Host's internal controller processes the ApplicationInstance and updates its status, the K8s watch delivers a `MODIFIED` event to Guest
3. Guest controller patches its local ApplicationInstance CR with the new status + access points

This completely eliminates the callback mechanism. No inbound traffic to Guest is needed.

### Phase 3c — (Optional) Host watches Guest

If Host needs to react to new Guest CRs (instead of Guest pushing them), Host can watch Guest's K8s API using the same OIDC pattern in the reverse direction.

### Phase 4 — Deletion

Guest controller calls `DELETE` on the Host K8s API to remove the remote CR. Same OIDC auth.

---

## How to build the cross-cluster K8s client in Go

```go
import (
    "golang.org/x/oauth2/clientcredentials"
    "k8s.io/client-go/rest"
    "net/http"
)

tokenConfig := &clientcredentials.Config{
    ClientID:     clientID,         // from K8s Secret
    ClientSecret: clientSecret,     // from K8s Secret
    TokenURL:     idpTokenURL,      // Host IdP endpoint
    Scopes:       []string{"opg.ewbi"},
}
tokenSource := tokenConfig.TokenSource(ctx)

remoteConfig := &rest.Config{
    Host: "https://host-k8s-api.example.com:6443",
    TLSClientConfig: rest.TLSClientConfig{
        CAData: hostCACert,  // Host K8s CA certificate
    },
    WrapTransport: func(rt http.RoundTripper) http.RoundTripper {
        return &oauth2.Transport{
            Source: tokenSource, // auto-refreshes tokens
            Base:   rt,
        }
    },
}

// Build a controller-runtime client for the remote cluster
remoteClient, err := client.New(remoteConfig, client.Options{Scheme: scheme})

// Or start a watch
remoteCache, err := cache.New(remoteConfig, cache.Options{Scheme: scheme})
```

This replaces the entire `internal/opg/opg.go` client factory with standard `client-go` mechanics.

---

## Trade-offs

### What you gain

| Benefit | Detail |
|---|---|
| Eliminate EWBI API service | No separate REST service to deploy, test, or version |
| Eliminate reverse-proxy | K8s API server handles auth natively |
| Eliminate callbacks | Cross-cluster watches are real-time, bidirectional, and reliable (with re-list on disconnect) |
| Native RBAC | Fine-grained, auditable, standard tooling (`kubectl auth can-i`) |
| Simpler operator code | Standard `client-go` / controller-runtime instead of generated EWBI client |
| Fewer network hops | Controller talks directly to K8s API — no proxy → IdP → proxy → API chain |

### What you lose / risk

| Concern | Severity | Detail |
|---|---|---|
| **GSMA EWBI compliance** | **High** | The standardized telco federation REST interface is gone. Other operators expecting EWBI can't interoperate. |
| **K8s API exposed externally** | **High** | The remote party can reach your K8s API server. Even with tight RBAC, the attack surface is larger than a purpose-built API. An RBAC misconfiguration could expose cluster internals. |
| **Business logic displacement** | **Medium** | Validation/transformation currently in the EWBI API must move to admission webhooks or controller logic. |
| **Single OIDC issuer limit** | **Medium** | Pre-K8s 1.30 supports only one OIDC issuer. Multiple partners need K8s 1.30+ `StructuredAuthenticationConfiguration` or a shared IdP. |
| **Long-lived watch connections** | **Low** | Cross-cluster watches are persistent TCP connections. Network partitions trigger re-list storms if not handled carefully. |
| **CA certificate distribution** | **Low** | Guest needs Host's K8s CA cert (and vice versa). Must be distributed out-of-band and rotated. |

---

## Comparison with EWBI API version

```
EWBI version:
Guest Ctrl → Host Proxy → Host IdP (verify) → Host EWBI API → Host K8s
Host Ctrl  → Guest Proxy → Guest IdP (verify) → Guest EWBI API → Guest K8s  (callback)

Direct K8s version:
Guest Ctrl → Host IdP (token) → Host K8s API (OIDC verify + RBAC)
Guest Ctrl ← Host K8s API (watch events)                                     (replaces callback)
```

The direct version has fewer moving parts but trades external interoperability (GSMA compliance) for simplicity.
