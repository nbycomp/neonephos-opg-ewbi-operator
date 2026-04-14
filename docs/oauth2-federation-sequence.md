sequenceDiagram
    autonumber

    %% ──────────────────────────────────────────────────
    %% Participants - Guest (left) then Host (right)
    %% All cross-platform traffic goes through the proxy.
    %% IdP is internal, never directly exposed.
    %% ──────────────────────────────────────────────────
    box rgb(220, 235, 255) Guest Operator Platform
        participant GuestK8s as Guest K8s API
        participant GuestCtrl as Guest Controllers<br/>(Federation, AppInstance)
        participant GuestSecret as Guest Secret Store<br/>(K8s Secret / Vault / SecretStore...)
        participant GuestProxy as Guest Reverse-Proxy<br/>(Oathkeeper / Envoy / ...)
        participant GuestIdP as Guest Identity Provider<br/>(Keycloak / ORY Hydra / ...)
        participant GuestEWBI as Guest EWBI API<br/>(callback endpoint)
    end

    box rgb(255, 235, 220) Host Operator Platform
        participant HostProxy as Host Reverse-Proxy<br/>(Oathkeeper / Envoy / ...)
        participant HostIdP as Host Identity Provider<br/>(Keycloak / ORY Hydra / ...)
        participant HostEWBI as Host EWBI API<br/>(opg-ewbi-api)
        participant HostK8s as Host K8s API
        participant HostCtrl as Host Controllers<br/>(Federation, AppInstance)
        participant HostSecret as Host Secret Store<br/>(K8s Secret)
    end

    rect rgb(255, 250, 210)
        Note over GuestCtrl, HostCtrl: Only Federation and ApplicationInstance CRs are shown.<br/>File, Artefact, and Application CRs follow the same<br/>forward (guest to host) + callback (host to guest) pattern<br/>through the same proxy / IdP / EWBI chain.<br/>Zone subscription (PUT), deletion (DELETE), and other CRUD<br/>operations follow the same forward path as Phase 1.<br/>Tokens are cached in-memory (oauth2.TokenSource) and reused until near expiry.
    end

    %% ══════════════════════════════════════════════════
    %% PHASE 0 - Pre-federation: Client registration
    %% ══════════════════════════════════════════════════
    rect rgb(245, 245, 255)
        Note over GuestSecret, HostSecret: PHASE 0 - Pre-federation setup (out-of-band / manual)
        Note over GuestSecret: PREREQUISITE (Guest admin):<br/>1. Register OAuth2 client on Host IdP<br/>2. Receive client_id + client_secret<br/>3. Create K8s Secret with credentials<br/>4. Reference Secret in Federation CR<br/>spec.guestPartnerCredentials.secretRef
        Note over HostSecret: PREREQUISITE (Host admin):<br/>1. Register OAuth2 client on Guest IdP<br/>for callbacks<br/>2. Receive callback_client_id + secret<br/>3. Create K8s Secret with credentials<br/>4. Reference Secret in Federation CR<br/>spec.partner.callbackCredentials.secretRef
    end

    %% ══════════════════════════════════════════════════
    %% PHASE 1 - Federation creation (Guest -> Host)
    %% ══════════════════════════════════════════════════
    rect rgb(230, 255, 230)
        Note over GuestCtrl, HostEWBI: PHASE 1 - Federation creation (FederationReconciler - guest)

        GuestK8s ->> GuestCtrl: Reconcile Federation CR (relation=guest)
        GuestCtrl ->> GuestSecret: Read client_id + client_secret from Secret
        GuestSecret -->> GuestCtrl: client_id + client_secret

        Note over GuestCtrl, HostProxy: Token acquisition via proxy (noop authenticator on /oauth2/token)
        GuestCtrl ->> HostProxy: POST /oauth2/token (grant_type=client_credentials,<br/>client_id, client_secret, scope=ewbi:federation)
        HostProxy ->> HostIdP: Forward POST /oauth2/token (pass-through)
        HostIdP -->> HostProxy: access_token (JWT / opaque)
        HostProxy -->> GuestCtrl: access_token
        Note right of GuestCtrl: Token cached in-memory (oauth2.TokenSource).<br/>Reused for subsequent calls until near expiry.<br/>Not persisted to Secret or disk.

        Note over GuestCtrl, HostProxy: API call via proxy (jwt/introspection authenticator on /partner/*)
        GuestCtrl ->> HostProxy: POST /partner (CreateFederation)<br/>Authorization: Bearer access_token<br/>X-Client-ID: client_id
        HostProxy ->> HostIdP: Introspect / verify JWT (JWKS)
        HostIdP -->> HostProxy: Token valid, scopes OK
        HostProxy ->> HostEWBI: Forward POST /partner
        HostEWBI ->> HostK8s: Create Federation CR (relation=host)
        HostEWBI -->> HostProxy: 200 federationContextId, offeredAvailabilityZones
        HostProxy -->> GuestCtrl: 200 federationContextId, offeredAvailabilityZones

        GuestCtrl ->> GuestK8s: Status().Update() Federation CR (federationContextId, offeredAZs, state=AVAILABLE)
    end

    %% ══════════════════════════════════════════════════
    %% PHASE 2 - Application Instance install (Guest -> Host)
    %% ══════════════════════════════════════════════════
    rect rgb(255, 250, 230)
        Note over GuestCtrl, HostEWBI: PHASE 2 - App instance install (AppInstanceReconciler - guest)

        GuestK8s ->> GuestCtrl: Reconcile ApplicationInstance CR (relation=guest, state empty)
        GuestCtrl ->> GuestK8s: Resolve parent Federation by context-id index
        Note right of GuestCtrl: Reuses cached access_token if not expired

        GuestCtrl ->> HostProxy: POST /partner/fedCtxId/application/appId/instance (InstallApp)<br/>Authorization: Bearer access_token
        HostProxy ->> HostIdP: Verify token
        HostIdP -->> HostProxy: OK
        HostProxy ->> HostEWBI: Forward request
        HostEWBI ->> HostK8s: Create ApplicationInstance CR (relation=host, state=PENDING)
        HostEWBI -->> HostProxy: 200 OK
        HostProxy -->> GuestCtrl: 200 OK

        GuestCtrl ->> GuestK8s: Status().Update() ApplicationInstance CR (state=PENDING, appInstanceId)
    end

    %% ══════════════════════════════════════════════════
    %% PHASE 2b - AppInstance callback (Host -> Guest)
    %% ══════════════════════════════════════════════════
    rect rgb(255, 240, 240)
        Note over GuestProxy, HostCtrl: PHASE 2b - AppInstance callback (AppInstanceReconciler - host)

        HostK8s ->> HostCtrl: Reconcile ApplicationInstance CR (relation=host, state changed)
        HostCtrl ->> HostK8s: Resolve parent Federation (host)
        HostCtrl ->> HostSecret: Read callback_client_id + callback_client_secret from Secret
        HostSecret -->> HostCtrl: callback_client_id + callback_client_secret

        Note over HostCtrl, GuestProxy: Token acquisition via Guest proxy (noop on /oauth2/token)
        HostCtrl ->> GuestProxy: POST /oauth2/token (client_credentials,<br/>callback_client_id, callback_client_secret,<br/>scope=ewbi:callback)
        GuestProxy ->> GuestIdP: Forward (pass-through)
        GuestIdP -->> GuestProxy: callback_access_token
        GuestProxy -->> HostCtrl: callback_access_token
        Note left of HostCtrl: Token cached in-memory.<br/>Reused for subsequent callbacks.

        Note over HostCtrl, GuestProxy: Callback via Guest proxy (jwt/introspection on /cb/*)
        HostCtrl ->> GuestProxy: POST /cb/clientId/application/appId/instance/status<br/>(AppInstCallbackLink)<br/>Authorization: Bearer callback_access_token
        GuestProxy ->> GuestIdP: Verify callback token
        GuestIdP -->> GuestProxy: OK
        GuestProxy ->> GuestEWBI: Forward callback
        GuestEWBI ->> GuestK8s: Status().Update() ApplicationInstance CR (state + accesspoints)
        GuestK8s -->> GuestEWBI: 200 OK (status updated)
        GuestEWBI -->> GuestProxy: 200 OK
        GuestProxy -->> HostCtrl: 200 OK
    end
