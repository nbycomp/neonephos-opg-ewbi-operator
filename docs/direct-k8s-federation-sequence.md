sequenceDiagram
    autonumber

    %% ──────────────────────────────────────────────────
    %% Participants - Guest (left) then Host (right)
    %% No EWBI API, no callbacks.
    %% Gateway acts as network filter (path allowlist),
    %% NOT as authentication layer (K8s API does OIDC).
    %% Host status changes are picked up via cross-cluster watch.
    %% ──────────────────────────────────────────────────
    box rgb(220, 235, 255) Guest Operator Platform
        participant GuestK8s as Guest K8s API
        participant GuestCtrl as Guest Controllers<br/>(Federation, AppInstance)
        participant GuestSecret as Guest Secret Store<br/>(K8s Secret / Vault / SecretStore...)
    end

    box rgb(255, 235, 220) Host Operator Platform
        participant HostGW as Host Network Gateway<br/>(Envoy / nginx / Cloud LB)<br/>Path filter only, no auth
        participant HostIdP as Host Identity Provider<br/>(Keycloak / ORY Hydra / ...)
        participant HostK8sAPI as Host K8s API Server<br/>(OIDC auth enabled)
        participant HostK8s as Host K8s Resources<br/>(CRDs + RBAC)
        participant HostCtrl as Host Controllers<br/>(Federation, AppInstance)
    end

    rect rgb(255, 250, 210)
        Note over GuestCtrl, HostCtrl: Only Federation and ApplicationInstance CRs are shown.<br/>File, Artefact, and Application CRs follow the same<br/>create + watch pattern. No EWBI API needed.<br/>Gateway is a path filter (defense in depth), not an auth layer.<br/>Zone subscription (PATCH), deletion (DELETE), and other CRUD<br/>operations follow the same forward path as Phase 1.<br/>Callbacks are replaced by cross-cluster watches.<br/>Tokens are cached in-memory (oauth2.TokenSource).
    end

    %% ══════════════════════════════════════════════════
    %% PHASE 0 - Pre-federation setup
    %% ══════════════════════════════════════════════════
    rect rgb(245, 245, 255)
        Note over GuestSecret, HostCtrl: PHASE 0 - Pre-federation setup (out-of-band / manual)
        Note over GuestSecret: PREREQUISITE (Guest admin):<br/>1. Register OAuth2 client on Host IdP<br/>2. Receive client_id + client_secret<br/>3. Create K8s Secret with credentials<br/>4. Reference Secret in Federation CR<br/>spec.guestPartnerCredentials.secretRef<br/>5. Host admin creates RBAC: ClusterRole<br/>scoped to opg.ewbi.nby.one CRDs only
        Note over HostCtrl: PREREQUISITE (Host admin):<br/>1. Configure K8s API server with<br/>--oidc-issuer-url pointing to Host IdP<br/>2. Create RBAC: ClusterRole + Binding<br/>mapping Guest OIDC sub to opg-ewbi role<br/>3. Ensure IdP JWKS endpoint is reachable<br/>from K8s API server<br/>4. Configure gateway path allowlist:<br/>/oauth2/token -> IdP (passthrough)<br/>/apis/opg.ewbi.nby.one/* -> K8s API<br/>All other paths -> 403 Forbidden
    end

    %% ══════════════════════════════════════════════════
    %% PHASE 1 - Federation creation (Guest -> Host K8s)
    %% ══════════════════════════════════════════════════
    rect rgb(230, 255, 230)
        Note over GuestCtrl, HostK8s: PHASE 1 - Federation creation (FederationReconciler - guest)

        GuestK8s ->> GuestCtrl: Reconcile Federation CR (relation=guest)
        GuestCtrl ->> GuestSecret: Read client_id + client_secret from Secret
        GuestSecret -->> GuestCtrl: client_id + client_secret

        Note over GuestCtrl, HostGW: Token acquisition via gateway (passthrough to IdP)
        GuestCtrl ->> HostGW: POST /oauth2/token (grant_type=client_credentials,<br/>client_id, client_secret, scope=opg.ewbi)
        HostGW ->> HostIdP: Forward /oauth2/token (passthrough)
        HostIdP -->> HostGW: access_token (JWT)
        HostGW -->> GuestCtrl: access_token (JWT)
        Note right of GuestCtrl: Token cached in-memory (oauth2.TokenSource).<br/>Used as Bearer token on rest.Config via<br/>WrapTransport with oauth2.Transport.

        Note over GuestCtrl, HostK8sAPI: K8s API call via gateway (path allowed, forwarded to K8s API)
        GuestCtrl ->> HostGW: POST /apis/opg.ewbi.nby.one/v1beta1/namespaces/ns/federations<br/>Authorization: Bearer access_token
        HostGW ->> HostK8sAPI: Forward (path in allowlist)
        HostK8sAPI ->> HostIdP: Verify JWT signature (JWKS, cached)
        HostIdP -->> HostK8sAPI: Valid (sub=client_id, scopes OK)
        Note right of HostK8sAPI: RBAC check: ClusterRoleBinding maps<br/>OIDC sub claim to opg-ewbi-guest role
        HostK8sAPI ->> HostK8s: Create Federation CR (relation=host)
        HostK8s -->> HostK8sAPI: 201 Created
        HostK8sAPI -->> HostGW: 201 Created
        HostGW -->> GuestCtrl: 201 Created (Federation with offeredAZs)

        GuestCtrl ->> GuestK8s: Status().Update() Federation CR (federationContextId, offeredAZs, state=AVAILABLE)
    end

    %% ══════════════════════════════════════════════════
    %% PHASE 2 - Application Instance install (Guest -> Host K8s)
    %% ══════════════════════════════════════════════════
    rect rgb(255, 250, 230)
        Note over GuestCtrl, HostK8s: PHASE 2 - App instance install (AppInstanceReconciler - guest)

        GuestK8s ->> GuestCtrl: Reconcile ApplicationInstance CR (relation=guest, state empty)
        GuestCtrl ->> GuestK8s: Resolve parent Federation by context-id index
        Note right of GuestCtrl: Reuses cached access_token if not expired

        GuestCtrl ->> HostGW: POST /apis/.../applicationinstances<br/>Authorization: Bearer access_token
        HostGW ->> HostK8sAPI: Forward (path in allowlist)
        HostK8sAPI ->> HostK8s: Create ApplicationInstance CR (relation=host, state=PENDING)
        HostK8s -->> HostK8sAPI: 201 Created
        HostK8sAPI -->> HostGW: 201 Created
        HostGW -->> GuestCtrl: 201 Created

        GuestCtrl ->> GuestK8s: Status().Update() ApplicationInstance CR (state=PENDING, appInstanceId)
    end

    %% ══════════════════════════════════════════════════
    %% PHASE 2b - Status sync via cross-cluster watch
    %% (replaces callbacks entirely)
    %% ══════════════════════════════════════════════════
    rect rgb(255, 240, 240)
        Note over GuestCtrl, HostK8s: PHASE 2b - Status sync via watch (replaces callbacks)

        Note right of GuestCtrl: Guest controller maintains a long-lived<br/>WATCH on Host K8s via gateway for ApplicationInstance CRs.<br/>Uses same OIDC token (auto-refreshed).

        HostCtrl ->> HostK8s: (internal) Process AppInstance, update status
        HostK8s ->> HostK8sAPI: Status subresource updated (state + accesspoints)

        HostK8sAPI -->> HostGW: WATCH event: MODIFIED
        HostGW -->> GuestCtrl: WATCH event: MODIFIED (ApplicationInstance status changed)

        GuestCtrl ->> GuestK8s: Status().Update() local ApplicationInstance CR (state + accesspoints)
        GuestK8s -->> GuestCtrl: 200 OK (status updated)
    end
