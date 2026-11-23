# OAuth 1.0a and OAuth 2.0 — A Builder's Guide

## What OAuth is actually for

OAuth exists to solve one problem: letting a third-party application act on a user's behalf against some API, without that application ever holding the user's password. Before OAuth, the only way for a budgeting app to read your bank transactions was for you to hand it your bank password, and for it to log in as you. That arrangement fails on every axis that matters — the app gets unlimited scope when it only needed read access to transactions, it gets unlimited duration because passwords do not expire on a schedule, you cannot revoke it without changing your password and breaking everything else, and the credential itself is now replicated across every vendor you ever tried. OAuth replaces that with a scoped, revocable, independently-expiring credential issued to a specific application, derived from an explicit user consent event.

```mermaid

flowchart LR
    subgraph before["Before OAuth: credential sharing"]
        direction LR
        U1["User"] -->|"hands over<br/>bank password"| A1["Budgeting App"]
        A1 -->|"logs in AS the user<br/>full scope, forever"| B1["Bank"]
    end
    subgraph after["With OAuth: delegated authorization"]
        direction LR
        U2["User"] -->|"consents once<br/>in the bank's own UI"| B2["Bank"]
        B2 -->|"scoped, revocable,<br/>expiring token"| A2["Budgeting App"]
        A2 -->|"read:transactions<br/>and nothing else"| B2
    end
    before ~~~ after
    style A1 fill:#f8d7da,color:#000
    style A2 fill:#d4edda,color:#000
```

*The password never reaches the third party. That single removed arrow is OAuth's entire contribution; everything else in this document is machinery to make it practical.*

The single most important thing to internalize before reading further is that OAuth is an **authorization** protocol, not an authentication one. It answers "may this application call this API on behalf of this user?" It does not, by itself, answer "who is this user, and are they signed in right now?" The industry conflated those two questions for years — the "Login with Facebook" era was largely people misusing OAuth 2.0 access tokens as proof of identity, which is unsound because an access token is a bearer credential that says nothing verifiable about who obtained it. OpenID Connect was created specifically to layer a real authentication answer on top of OAuth 2.0, and if you need login, that is what you use. I come back to this distinction later, because getting it wrong is the most common and most consequential integration mistake.

Four parties recur throughout, and it is worth fixing the vocabulary now because both versions of the protocol use it. The **resource owner** is the human who owns the data. The **client** is your application, the thing that wants access. The **authorization server** is the system that authenticates the human, shows them a consent screen, and mints credentials. The **resource server** is the API that holds the data and accepts those credentials. In practice the authorization server and resource server are usually operated by the same company and people say "the provider" to mean both, but they are separate roles and at scale they are often separate systems with separate failure modes, so keeping them distinct in your head pays off during incidents.

```mermaid

flowchart TB
    RO["Resource Owner<br/>the human who owns the data"]
    C["Client<br/>your application"]
    subgraph prov["The provider (usually one vendor, two roles)"]
        AS["Authorization Server<br/>authenticates the human,<br/>shows consent, mints tokens"]
        RS["Resource Server<br/>holds the data,<br/>accepts tokens"]
    end
    RO -->|"1 - uses"| C
    C -->|"2 - sends the user to consent"| AS
    RO -->|"3 - authenticates and approves"| AS
    AS -->|"4 - issues tokens"| C
    C -->|"5 - calls with access token"| RS
    RS -.->|"6 - validates token<br/>(introspection or local JWT check)"| AS
    style RO fill:#e1f5ff,color:#000
    style C fill:#fff3cd,color:#000
```

*Steps 2 and 4 are the only places your code talks to the authorization server, and step 5 the only place it talks to the resource server. Everything else is the user's browser and the provider's own UI.*

## OAuth 1.0a: signatures instead of trust in the transport

OAuth 1.0a is a product of its time. It was designed when TLS was expensive enough that it was not assumed on every API call, so the protocol could not rely on the transport to keep a credential secret. Its answer was to never transmit the secret at all. Instead, the client and server each hold a shared secret, and every single request carries a cryptographic signature computed over the request itself — the HTTP method, the URL, and the full set of sorted, canonicalized parameters. The server recomputes that signature with its copy of the secret and compares. A credential is therefore never on the wire, and an intercepted request cannot be replayed against a different URL or with different parameters, because the signature would no longer match.

That design decision is the source of everything people remember about OAuth 1.0a, good and bad. It is genuinely secure over plain HTTP, which OAuth 2.0 is not. It is also brutally unforgiving to implement, because the signature depends on byte-exact agreement between client and server about how to canonicalize the request, and there are half a dozen places where reasonable implementations disagree. Percent-encoding must follow RFC 3986 with only `A-Z a-z 0-9 - . _ ~` left unescaped, which means the common language-library "URL encode" function is usually wrong — it encodes spaces as `+` rather than `%20`, or leaves `!` and `*` alone. Parameters must be sorted by encoded key and then by encoded value. Body parameters are included in the signature only when the content type is `application/x-www-form-urlencoded`, which means a JSON POST signs an empty body and a form POST does not. When you debug an OAuth 1.0a integration, you are almost always debugging the signature base string, and the only productive technique is to print your base string and diff it character-by-character against the provider's documented example. Appendix A walks through the construction and shows the canonical example from the spec.

The credential model has three layers rather than two. Your application registers once and receives a **consumer key and consumer secret**, which identify the application itself. To start a user flow, the client makes a signed server-to-server call to a request-token endpoint and receives **temporary credentials** — a request token and its own secret. The user is redirected to the provider with that request token, authenticates, approves, and is sent back with the request token plus an `oauth_verifier`. The client then exchanges the verifier for **token credentials** — the long-lived access token and, critically, an access token *secret*. From then on every API call is signed with a key built by concatenating the consumer secret and the token secret. That two-part signing key is why OAuth 1.0a credentials cannot be copy-pasted the way a bearer token can: possession of the token alone is useless without the token secret and the consumer secret.

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant App as Client / your app
    participant SP as Service Provider

    Note over App,SP: Leg 1 - temporary credentials, signed, server to server
    App->>SP: POST /oauth/request_token<br/>signed with consumer secret only<br/>oauth_callback set
    SP-->>App: request token + request token secret

    Note over User,SP: Leg 2 - user authorization, front channel
    App-->>User: redirect to /oauth/authorize?oauth_token=temp
    User->>SP: authenticate and approve
    SP-->>User: redirect to callback<br/>oauth_token + oauth_verifier
    User->>App: callback delivers the verifier

    Note over App,SP: Leg 3 - token credentials, signed, server to server
    App->>SP: POST /oauth/access_token<br/>signed with consumer + temp token secret<br/>oauth_verifier proves the same user approved
    SP-->>App: access token + access token SECRET

    Note over App,SP: There is no refresh. These usually live until revoked.
    App->>SP: GET /api/resource<br/>Authorization: OAuth ... oauth_signature=...
    SP-->>App: 200 OK
```

*Legs 1 and 3 are signed server-to-server calls; leg 2 is the only part the user sees. The verifier returned in leg 2 is what proves to leg 3 that the user who approved is the user you started the flow for. Note the absence of any refresh step.*

The `oauth_verifier` deserves a note because it is the entire difference between OAuth 1.0 and 1.0a. The original 1.0 spec let the user return from the provider without any value binding that return to the specific request token the client had issued, which allowed a session fixation attack: an attacker starts a flow, gets a request token, tricks a victim into authorizing it, and then completes the exchange themselves, ending up with a token bound to the victim's account. The 1.0a revision added the verifier, delivered to the user's browser on the callback and required on the exchange, so the client can only complete a flow that the same user actually approved. The lesson generalizes — every version of OAuth has needed some value that binds the authorization step to the token-exchange step, and in OAuth 2.0 that job is split between `state` and PKCE.

Two operational properties of OAuth 1.0a matter more than the cryptography when you are running it. First, tokens usually do not expire. There is no refresh flow in the spec, so a token lives until the user or the provider revokes it. That makes the runtime simple — no refresh scheduler, no expiry tracking — but it also means a leaked token is leaked indefinitely, and your only lifecycle event is revocation, which providers signal inconsistently. Second, clock skew is a real failure mode: the signature includes a timestamp and providers reject requests outside a tolerance window, typically five minutes, so a host with drifting time fails every call with an opaque 401.

You will still meet OAuth 1.0a in the wild, which is the only reason to learn it. It persists in long-lived enterprise and financial APIs where migration is expensive — NetSuite, various Garmin and Trello endpoints, Twitter's v1.1 surface, and a long tail of ERP and accounting systems. If you are building a connector platform, you need it as a supported auth type, not as a curiosity. If you are integrating a single modern SaaS product, you will almost certainly never touch it.

## OAuth 2.0: a framework, not a protocol

OAuth 2.0 made the opposite bet. It assumes TLS everywhere and therefore stops signing requests. A token becomes a **bearer** credential — a string that grants access to whoever presents it, exactly like a cash banknote. All the cryptographic machinery of 1.0a collapses into `Authorization: Bearer <token>`. That is why OAuth 2.0 won: an integration that took days of signature debugging became an afternoon, and mobile apps and single-page apps, which cannot safely hold a signing secret anyway, finally had a story.

The cost of that bet is that every security property now rests on the transport and on correct handling of the token. A bearer token in a URL ends up in access logs, browser history, and `Referer` headers. A bearer token in `localStorage` is readable by any XSS payload on the page. A bearer token leaked from a log aggregator is immediately usable by anyone. OAuth 2.0 is easier to implement and much easier to implement insecurely, and essentially all subsequent specification work — PKCE, the security best-practice RFC, DPoP, mTLS-bound tokens, OAuth 2.1 — is the community walking back the most dangerous parts of that original simplification.

The second thing to understand about OAuth 2.0 is that RFC 6749 is deliberately a *framework*. It defines roles, endpoints, and a small set of grant types, and then leaves a great deal genuinely unspecified: the format and lifetime of an access token, whether refresh tokens are issued at all, what scope strings mean and how they are delimited, what error bodies look like, how clients authenticate to the token endpoint, and whether any discovery mechanism exists. This is why "we support OAuth 2.0" tells you almost nothing about what you will actually have to build, and it is the root cause of every difficulty in the multi-provider section below. The specification's flexibility was a feature for adoption and is a tax on integrators forever.

Structurally, OAuth 2.0 gives you two endpoints you must know and three more you will want. The **authorization endpoint** is a URL you send the user's browser to; it is the only part of the flow the user sees. The **token endpoint** is a back-channel API your server calls to trade one credential for another; the user's browser never touches it. Beyond those, a **revocation endpoint** (RFC 7009) lets you proactively invalidate a token when a user disconnects, an **introspection endpoint** (RFC 7662) lets a resource server ask whether a token is still valid and what it covers, and a **metadata document** at a well-known URL (RFC 8414, or the OpenID Connect equivalent) lets you discover all of the above programmatically instead of hardcoding it. Discovery matters enormously at scale and I return to it later.

```mermaid
flowchart TB
    subgraph front["Front channel - the browser goes here"]
        AUTH["/authorize<br/>the only endpoint a user ever sees"]
    end
    subgraph back["Back channel - your server only"]
        TOK["/token<br/>exchange code, refresh, client credentials"]
        REV["/revoke - RFC 7009<br/>call this on disconnect"]
        INTRO["/introspect - RFC 7662<br/>is this token still good?"]
        META["/.well-known/... - RFC 8414 / OIDC<br/>discover all of the above<br/>instead of hardcoding it"]
    end
    subgraph undef["What RFC 6749 deliberately does NOT specify"]
        direction TB
        U1["access token format<br/>opaque? JWT? how long?"]
        U2["whether refresh tokens<br/>are issued at all, or rotate"]
        U3["what scope strings mean<br/>and how they are delimited"]
        U4["error response shapes"]
        U5["how clients authenticate<br/>Basic header? request body?"]
    end
    NOTE["This is why 'we support OAuth 2.0' tells you almost nothing<br/>about what you will have to build - and it is the root cause<br/>of every difficulty in the multi-provider section."]
    undef --> NOTE
    style undef fill:#3a2226,color:#f0dcdf,stroke:#8a5560
    style NOTE fill:#fff3cd,color:#000
    style AUTH fill:#e1f5ff,color:#000
    style META fill:#d4edda,color:#000
```

*Two endpoints you must build against, three that make your life easier, and a red box that explains why every provider integration feels slightly different from the last one.*

Access tokens and refresh tokens do different jobs and should be reasoned about separately. The access token is short-lived by design — typically minutes to an hour — and is presented to the resource server on every call. Its short life is what limits the blast radius of a leak. The refresh token is long-lived, is presented only to the authorization server, and exists solely to mint new access tokens without dragging the user back through a consent screen. The asymmetry is deliberate: the high-exposure credential is cheap to replace and the high-value credential is rarely transmitted. An important practical consequence is that your refresh token is the thing that actually represents the user's connection. If you lose it, the connection is dead and the user must re-consent; if it leaks, the attacker has durable access. Treat access tokens as cache and refresh tokens as durable state, and encrypt the latter accordingly.

```mermaid
flowchart LR
    subgraph at["Access token"]
        direction TB
        A1["presented on EVERY API call"]
        A2["high exposure"]
        A3["short life - minutes to an hour"]
        A4["cheap to replace"]
        A5["treat as CACHE"]
    end
    subgraph rt["Refresh token"]
        direction TB
        B1["presented ONLY to the authorization server"]
        B2["low exposure"]
        B3["long life - months, or until revoked"]
        B4["losing it kills the connection;<br/>the user must consent again"]
        B5["treat as DURABLE STATE - encrypt at rest"]
    end
    at -->|"the asymmetry is deliberate:<br/>the high-exposure credential is cheap,<br/>the high-value one is rarely transmitted"| rt
    style at fill:#3d3520,color:#f2e8cc,stroke:#8a7940
    style rt fill:#1e3350,color:#dce7f5,stroke:#4f7aa8
    style B5 fill:#d4edda,color:#000
```

*Reasoning about these two separately is the habit that prevents most token-handling mistakes.*

Scopes are the unit of authorization, and they are the part of OAuth 2.0 most thoroughly undefined by the spec. A scope is just a string that the provider interprets. The spec suggests space-delimiting them in a single parameter; GitHub uses commas, some providers use plus signs, and a few take repeated parameters. Their granularity varies from the excellent (`https://www.googleapis.com/auth/drive.file`, which grants access only to files your app created) to the useless (a single scope that means "everything"). The one universal rule is to request the minimum set that makes your feature work, and to request scopes incrementally as features are used rather than demanding everything at install time — both because users abandon consent screens that ask for too much, and because some providers subject broad scopes to a manual security review that can add weeks to your launch.

## The flows, and which ones you should actually use

**Authorization Code with PKCE** is the flow. If you take one thing from this document, take that: for any client acting on behalf of a human, this is the correct answer, and the others are either special cases or mistakes. It works in two legs. In the front channel, you redirect the user's browser to the provider's authorization endpoint with your client ID, your redirect URI, the scopes you want, a random `state` value, and a `code_challenge`. The user authenticates and consents, and the provider redirects their browser back to your registered redirect URI with a short-lived, single-use `code` and your `state` echoed back. In the back channel, your server POSTs that code to the token endpoint along with your client secret and the `code_verifier`, and receives the access token and refresh token in the response body. Appendix B traces the actual HTTP.

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant B as Browser
    participant App as Your Server
    participant AS as Authorization Server
    participant RS as Resource Server

    App->>App: generate state + code_verifier<br/>code_challenge = SHA256(code_verifier)
    App->>App: store state to verifier, session, return_to

    rect rgb(28, 52, 84)
    Note over User,AS: FRONT CHANNEL - passes through the browser
    App-->>B: 302 to /authorize with client_id, redirect_uri,<br/>scope, state, code_challenge, S256
    B->>AS: GET /authorize
    User->>AS: authenticate and consent
    AS-->>B: 302 to redirect_uri with code and state
    B->>App: GET /oauth/callback
    end

    App->>App: state exists? bound to session? unused?<br/>then consume it

    rect rgb(26, 60, 38)
    Note over App,AS: BACK CHANNEL - direct TLS, never touches the browser
    App->>AS: POST /token<br/>code + client_secret + code_verifier
    AS-->>App: access_token, refresh_token, expires_in, scope
    end

    App->>RS: identity probe - which account is this?
    RS-->>App: provider account id
    App->>App: encrypt refresh token, persist connection,<br/>store GRANTED scopes and absolute expiry

    App->>RS: Authorization: Bearer access_token
    RS-->>App: 200 OK
```

*The shading is the security boundary. Everything in the blue band passes through the browser and must be assumed visible to an attacker who controls it; the credentials that actually matter are only ever issued inside the green band.*

The split between front and back channel is the whole point. The authorization code that travels through the browser is useless on its own — it is single-use, expires in under a minute, and cannot be redeemed without the client secret or the PKCE verifier, neither of which ever touched the browser. The valuable credentials are only ever delivered over a direct server-to-server TLS connection.

```mermaid
flowchart TB
    subgraph browser["Passes through the browser - assume it is attacker-visible"]
        direction LR
        p1["client_id"]
        p2["redirect_uri"]
        p3["scope"]
        p4["state"]
        p5["code_challenge<br/>a SHA-256 hash"]
        p6["authorization code<br/>single-use, seconds-long TTL"]
    end
    subgraph server["Never leaves the server-to-server channel"]
        direction LR
        s1["client_secret"]
        s2["code_verifier"]
        s3["access_token"]
        s4["refresh_token"]
    end
    browser -->|"an attacker who captures every value above<br/>still cannot redeem the code, because redemption<br/>needs a secret that was never in the browser"| server
    style browser fill:#3a2226,color:#f0dcdf,stroke:#8a5560
    style server fill:#1f3a28,color:#daeede,stroke:#4f8a62
```

*The same boundary stated as an inventory. This is the table to check a design against when you are unsure whether something is safe to put in a redirect.*

`state` and PKCE look similar and solve different problems, and you need both. `state` is a random value you generate, store against the user's session, and compare on return; it defends against CSRF, where an attacker tricks a logged-in user's browser into completing *the attacker's* authorization flow, silently connecting the attacker's account to the victim's. PKCE (RFC 7636) defends against code interception: you generate a random `code_verifier`, send its SHA-256 hash as the `code_challenge` on the authorization request, and send the raw verifier on the token exchange. If an attacker steals the authorization code — historically achievable on mobile via a malicious app registering the same custom URL scheme, and more mundanely via referrer leakage or a redirect-URI flaw — they still cannot redeem it, because they never saw the verifier. PKCE was originally specified for public clients that have no client secret, but current guidance, including OAuth 2.1, is that every client uses it, confidential ones included. It is cheap and it closes real attacks.

**Client Credentials** is the flow for machine-to-machine access where there is no user at all — your service calling another service with its own identity. Your server POSTs its client ID and secret to the token endpoint and gets back an access token. There is no browser, no consent screen, and normally no refresh token, because you can simply re-authenticate whenever you need a new access token. Use it for your own backend jobs and for provider APIs that expose app-level rather than user-level data. Do not use it to impersonate users.

```mermaid
sequenceDiagram
    autonumber
    participant Svc as Your service
    participant AS as Authorization Server
    participant RS as Resource Server

    Note over Svc,RS: No user, no browser, no consent screen.
    Svc->>AS: POST /token<br/>grant_type=client_credentials<br/>client_id + client_secret
    AS-->>Svc: access_token, expires_in
    Note over Svc,AS: Usually no refresh token - just re-authenticate.
    Svc->>RS: Authorization: Bearer access_token
    RS-->>Svc: 200 OK
```

*No user, no browser, no consent screen, and normally no refresh token — you simply re-authenticate.*

**Device Authorization Grant** (RFC 8628) handles inputs-constrained clients — TVs, CLIs, IoT devices — where you cannot host a browser or type a password. The device asks the authorization server for a code, displays a short user code and a URL, and then polls the token endpoint while the user completes the flow on their phone or laptop. If you have ever run a CLI that printed "go to example.com/device and enter WXYZ-1234", that is this grant. It is also the right pattern for headless server tooling.

```mermaid
sequenceDiagram
    autonumber
    participant D as Device - TV or CLI
    actor User
    participant Phone as User's phone or laptop
    participant AS as Authorization Server

    D->>AS: POST /device_authorization with client_id and scope
    AS-->>D: device_code, user_code WXYZ-1234,<br/>verification_uri, polling interval
    D-->>User: "Go to example.com/device and enter WXYZ-1234"

    loop until approved or expired
        D->>AS: POST /token with grant_type=device_code
        AS-->>D: 400 authorization_pending
    end

    User->>Phone: opens verification_uri
    Phone->>AS: enters user_code, authenticates, consents
    AS-->>Phone: approved

    D->>AS: POST /token with grant_type=device_code
    AS-->>D: access_token and refresh_token
    Note over D,AS: The constrained device never handled a password.
```

*The constrained device never handles a password and never renders a consent screen; it only polls.*

**Refresh Token** is technically a grant type of its own: you POST the refresh token to the token endpoint and get a fresh access token, and often a new refresh token as well. Whether you get a new one is the single most operationally significant provider difference in OAuth, and I treat it in detail in the integration section, because rotation turns refresh from a stateless operation into one requiring careful concurrency control.

Two grants are deprecated and you should treat their presence in a provider's documentation as a warning sign. **Implicit** returned the access token directly in the URL fragment of the browser redirect, skipping the back channel entirely; it was a workaround for browsers that could not make cross-origin requests, that constraint disappeared with CORS, and it leaks tokens into browser history and referrers. **Resource Owner Password Credentials** has the user type their username and password directly into your application, which you then forward to the token endpoint — which is precisely the anti-pattern OAuth was invented to eliminate, and it is incompatible with multi-factor authentication. OAuth 2.1 removes both. If a provider only offers ROPC, you are not really doing OAuth and should treat the credential with the same care as a stored password.

```mermaid
flowchart TB
    Q1{"Is a human delegating<br/>access to their data?"}
    Q1 -->|"No - your service<br/>acting as itself"| CC["Client Credentials"]
    Q1 -->|"Yes"| Q2{"Can the device render a browser<br/>and take keyboard input?"}
    Q2 -->|"No - TV, CLI, IoT"| DV["Device Authorization Grant<br/>RFC 8628"]
    Q2 -->|"Yes"| AC["Authorization Code + PKCE"]
    AC --> Q3{"Do you need access while<br/>the user is away?"}
    Q3 -->|"Yes"| RT["Request the provider's offline scope,<br/>store and schedule the refresh token"]
    Q3 -->|"No"| ST["Access token only,<br/>no refresh machinery"]
    X1["Implicit"] -.->|"removed in OAuth 2.1<br/>leaks tokens via URL fragment"| AC
    X2["Resource Owner Password"] -.->|"removed in OAuth 2.1<br/>breaks MFA, recreates the anti-pattern"| AC
    style AC fill:#d4edda,color:#000
    style X1 fill:#f8d7da,color:#000
    style X2 fill:#f8d7da,color:#000
```

*The decision is usually three questions deep. The two red boxes are the paths OAuth 2.1 removes outright.*

## Where authentication fits

If what you need is "let users sign in with their Google account", you need **OpenID Connect**, not bare OAuth 2.0. OIDC is a thin, strictly-specified layer on top of the authorization code flow: you add `openid` to your scopes, and the token response gains an `id_token` alongside the access token. That ID token is a signed JWT with a defined set of claims — who the user is (`sub`), who issued it (`iss`), which client it was minted for (`aud`), when it expires (`exp`), and a `nonce` echoing a value you supplied. Because it is signed by the provider and contains an audience and a nonce, you can *verify* it rather than merely possess it, which is exactly what a bearer access token cannot give you.

```mermaid
flowchart LR
    TR["Token response from an<br/>OIDC-enabled provider"]
    TR --> AT["access_token<br/>opaque or JWT"]
    TR --> RT2["refresh_token"]
    TR --> IT["id_token<br/>always a signed JWT"]
    AT --> ATU["FOR: calling the resource server<br/>A bearer credential.<br/>Says nothing verifiable about<br/>who obtained it."]
    IT --> ITU["FOR: authenticating the user<br/>Verify signature, iss, aud,<br/>exp and nonce - then you know<br/>WHO, and that it was minted for YOU."]
    ATU -.->|"using this for login is the<br/>confused-deputy vulnerability"| ITU
    style AT fill:#fff3cd,color:#000
    style IT fill:#d4edda,color:#000
    style ATU fill:#f8d7da,color:#000
```

*One token response, three credentials, and only one of them can answer "who is this?" in a way you can verify.*

The failure mode worth naming explicitly is the "confused deputy" pattern that bit a generation of mobile apps. A client sends an access token to its own backend as proof of identity; the backend calls the provider's userinfo endpoint with it, gets back a user ID, and logs that user in. The flaw is that an access token issued to a *different* application, for the same user, looks identical — so a malicious app can take a token its own users granted it and replay it against your backend to log in as them. An ID token cannot be replayed this way because the `aud` claim names your client ID and the signature covers it. The rule that follows: validate the signature, the issuer, the audience, the expiry, and the nonce on every ID token, and never authenticate a user from an access token or a userinfo call alone.

```mermaid
sequenceDiagram
    autonumber
    actor Victim
    participant Mal as Malicious App
    participant AS as Provider
    participant You as Your Backend

    Note over Victim,Mal: Step 1 - the victim legitimately uses the malicious app
    Victim->>Mal: sign in with Provider
    Mal->>AS: normal authorization code flow
    AS-->>Mal: access_token for the VICTIM,<br/>audience = Malicious App

    Note over Mal,You: Step 2 - replay that token as if it were identity
    Mal->>You: POST /login carrying the access_token
    You->>AS: GET /userinfo with the supplied token
    AS-->>You: subject = victim user id
    You-->>Mal: session cookie for the VICTIM

    Note over You,AS: The token was never issued to you - but a bearer<br/>token carries no verifiable statement about who holds it.
    Note over You,AS: An ID token does: aud names YOUR client_id,<br/>nonce binds it to your request, and the signature covers both.
```

*Steps 5 through 8 are the whole vulnerability: your backend cannot distinguish a token minted for you from one minted for someone else, because a bearer token makes no verifiable claim about who holds it.*

OIDC also standardizes several things bare OAuth leaves open, which is a quiet practical benefit — a discovery document at `/.well-known/openid-configuration`, a published JWKS endpoint for signing keys, and defined claim names. When a provider supports OIDC, your integration gets meaningfully more uniform.

## What you actually build to integrate one provider

It is easy to read the flow diagram and conclude that integrating OAuth means writing one redirect and one token exchange. The flow is maybe fifteen percent of the work. What you are really building is a small credential lifecycle system, and it has roughly eight parts. Laying them out concretely is the most useful thing this document can do, because the gap between "the flow works in my demo" and "connections stay alive for eighteen months across ten thousand users" is entirely in these components.

```mermaid
flowchart TB
    subgraph ui["User-facing"]
        Connect["Connect button<br/>and connections page"]
    end
    subgraph flow["Flow endpoints"]
        Init["Initiation endpoint<br/>builds the authorize URL,<br/>mints state + PKCE verifier"]
        CB["Callback endpoint<br/>validates state, exchanges code,<br/>runs the identity probe"]
    end
    subgraph st["State"]
        FS[("Flow state<br/>state to verifier<br/>short TTL, single use")]
        CS[("Connection store<br/>encrypted refresh token,<br/>granted scopes, expiry, status")]
        KMS{{"KMS / secret manager<br/>client secrets and data keys"}}
    end
    subgraph rt["Runtime"]
        Refresh["Refresh worker<br/>proactive at ~75% of TTL,<br/>single-flight per connection"]
        Caller["API call wrapper<br/>attach token; on 401<br/>refresh once, retry once"]
        Revoke["Disconnect<br/>revoke upstream, then delete"]
    end
    subgraph obs["Observability"]
        Metrics["Per-provider: consent funnel,<br/>refresh success rate, token age,<br/>dead-connection count"]
    end
    P(["Provider"])

    Connect --> Init --> P
    P --> CB
    Init -.-> FS
    CB -.-> FS
    CB --> CS
    CS -.-> KMS
    Refresh --> CS
    Refresh --> P
    Caller --> CS
    Caller --> P
    Revoke --> P
    Revoke --> CS
    Caller -.-> Metrics
    Refresh -.-> Metrics
    CB -.-> Metrics
    style CS fill:#e1f5ff,color:#000
    style KMS fill:#fff3cd,color:#000
```

*The flow endpoints are the small box in the middle — the part everyone builds first. The state, runtime and observability components surrounding them are what determine whether a connection is still working in a year.*

**Provider configuration and secret storage.** You need somewhere to hold, per provider, the authorization and token endpoint URLs, your client ID and secret, the redirect URI you registered, the scopes you request, and the various quirks that provider requires. The client secret is a production secret and belongs in a secret manager, not in configuration files or environment variables baked into an image. Where the provider supports RFC 8414 or OIDC discovery, fetch the endpoints from the metadata document and cache them rather than hardcoding, because providers do move URLs.

**The initiation endpoint.** Something in your application builds the authorization URL and redirects the user. This is where you generate the `state` value and the PKCE verifier, and where you make a decision that people routinely get wrong: `state` must be bound to the user's session and stored server-side with a short TTL, along with the PKCE verifier and any context you need on return, such as which of your workspaces the connection belongs to or where to send the user afterward. Packing that context into the `state` string itself is tempting and is a vulnerability unless it is signed, because `state` arrives back from an untrusted browser. Store the context server-side, keyed by a random `state`, and treat the returned value as nothing more than a lookup key that must exist and must not have been used before.

**The callback endpoint.** This is the security-critical piece and it has a fixed checklist. Verify that `state` exists in your store, matches the current session, and has not already been consumed — then delete it, so a replayed callback fails. Handle the error case first, because the user may have clicked Deny and the provider will redirect back with `error=access_denied` and no code; that path needs a real user-facing outcome, not a stack trace. Exchange the code for tokens over the back channel, authenticating with your client secret in the way that specific provider expects. Then — and this step is frequently missed — call some cheap identifying API with the new token to learn *which account* you just connected, because the provider will not always tell you in the token response, and you need it both to display the connection to the user and to detect that they have reconnected an account you already have. Finally, persist, and redirect the user somewhere that makes sense.

```mermaid
flowchart TB
    S(["Callback received"]) --> E1{"error parameter present?"}
    E1 -->|"yes - access_denied"| UX["Show a real outcome.<br/>The user clicked Deny;<br/>this is not an exception."]
    E1 -->|"no"| V1{"state exists in the store?"}
    V1 -->|"no"| REJ["Reject - CSRF or replay"]
    V1 -->|"yes"| V2{"bound to the current session?"}
    V2 -->|"no"| REJ
    V2 -->|"yes"| V3{"not already consumed?"}
    V3 -->|"no"| REJ
    V3 -->|"yes"| CONS["Consume it - delete now,<br/>so a replayed callback fails"]
    CONS --> EX["Exchange the code at the token endpoint<br/>with client_secret and code_verifier"]
    EX --> PR["Identity probe:<br/>which account did we just connect?"]
    PR --> SC["Persist encrypted tokens, GRANTED scopes,<br/>absolute expiry, provider account id"]
    SC --> DONE(["Redirect to return_to"])
    style REJ fill:#f8d7da,color:#000
    style DONE fill:#d4edda,color:#000
    style CONS fill:#fff3cd,color:#000
```

*Every arrow into the red box is a real attack. The ordering matters: check the error parameter before anything else, and consume state before exchanging the code, so that a replayed callback fails closed.*

**Token storage.** Refresh tokens are password-equivalent and must be encrypted at rest with a key from a KMS, ideally with envelope encryption so that a database dump alone is insufficient. The row should carry more than the tokens: the provider, your internal user or tenant ID, the provider-side account ID, the granted scopes as actually returned rather than as requested, the access token expiry as an absolute timestamp, the refresh token expiry if the provider gives one, a status field, and timestamps for last successful use and last successful refresh. Storing granted scopes matters because providers sometimes grant less than you asked for, and you want to fail with a clear message rather than a confusing 403 six weeks later. Appendix F sketches the schema.

```mermaid
stateDiagram-v2
    [*] --> Pending : user clicks Connect
    Pending --> Active : callback validated
    Pending --> [*] : consent denied

    Active --> Active : proactive refresh succeeds
    Active --> Degraded : transient failure
    Degraded --> Active : backoff retry succeeds
    Degraded --> NeedsReauth : invalid_grant

    Active --> NeedsReauth : invalid_grant on refresh
    Active --> Revoked : user disconnects
    NeedsReauth --> Pending : user reconnects
    Revoked --> [*]

    note right of NeedsReauth
        Terminal for retries.
        Causes: revoked upstream,
        rotation race lost the chain,
        refresh token idle-expired.
        Keep retrying here and your
        client_id gets throttled.
    end note

    note right of Degraded
        5xx, timeout, 429.
        Still recoverable.
    end note
```

*NeedsReauth is the state worth designing around. It is reachable from several directions, it is terminal for retries, and the only exit is the user consenting again.*

**The refresh mechanism.** This is where most of the real engineering lives, and the naive version — check expiry before each call, refresh if expired — breaks in production for two reasons. The first is concurrency. If ten requests for the same connection notice an expired token simultaneously, they all refresh, and against a provider that rotates refresh tokens each use, nine of those refreshes invalidate the tenth; worse, providers with reuse detection may treat the burst as a stolen-token replay and revoke the entire chain, killing the connection permanently. You need single-flight semantics per credential: a distributed lock or a `SELECT ... FOR UPDATE` on the credential row, with the losing threads waiting and re-reading rather than refreshing. The second is that lazy refresh puts a token-endpoint round trip on your latency-critical path and, for providers whose refresh tokens expire from *inactivity*, never runs at all for dormant connections — so the connection silently dies and the user discovers it when they finally need it. The robust design is proactive refresh on a schedule at around seventy-five percent of the access token's lifetime, with jitter to avoid a synchronized stampede, plus lazy refresh as a fallback, plus a keep-alive pass for connections approaching a refresh-token idle deadline. Appendix D gives the pseudocode.

```mermaid
sequenceDiagram
    autonumber
    participant W1 as Worker 1
    participant W2 as Worker 2
    participant DB as Connection store
    participant AS as Authorization Server

    Note over W1,AS: Rotating provider. No single-flight. Both see an expired token.
    W1->>DB: read connection, refresh token R1
    W2->>DB: read connection, refresh token R1
    W1->>AS: refresh using R1
    AS-->>W1: access A2 and NEW refresh R2
    W1->>DB: persist R2
    W2->>AS: refresh using R1, already rotated away
    AS-->>W2: 400 invalid_grant
    Note over AS: Providers with reuse detection read this as a<br/>stolen-token replay and revoke the ENTIRE chain.
    AS->>AS: revoke R1 and R2
    W2->>DB: mark NEEDS_REAUTH
    Note over W1,DB: The connection is now permanently dead.<br/>The user must consent again.
```

*This is the naive implementation under concurrency. The damage is not a failed request — reuse detection revokes the entire token chain, so the connection is permanently dead.*

```mermaid
sequenceDiagram
    autonumber
    participant W1 as Worker 1
    participant W2 as Worker 2
    participant L as Distributed lock
    participant DB as Connection store
    participant AS as Authorization Server

    W1->>L: acquire lock refresh:conn-123
    L-->>W1: granted
    W2->>L: acquire lock refresh:conn-123
    Note over W2,L: blocks - does not refresh
    W1->>DB: re-read INSIDE the lock
    W1->>AS: refresh using R1
    AS-->>W1: access A2 and new refresh R2
    W1->>DB: persist A2 and R2 atomically, before anything else
    W1->>L: release
    L-->>W2: granted
    W2->>DB: re-read inside the lock
    Note over W2,DB: token is already valid - no refresh needed
    W2->>L: release
```

*The fix is small, and the subtle part is the re-read inside the lock. Serialize correctly but skip that re-read and you still perform a second, chain-breaking refresh.*

```mermaid
flowchart LR
    T0(["t=0 - tokens issued<br/>access TTL 3600s"]) --> T1["t ~ 2700s (75% of TTL) + jitter<br/>proactive refresh fires"]
    T1 -->|"success - new TTL, reschedule"| T0
    T1 -->|"transient failure"| BO["exponential backoff,<br/>still ahead of actual expiry"]
    BO --> T1
    T1 -->|"invalid_grant"| DEAD["NEEDS_REAUTH<br/>stop, notify the user"]
    LAZY["an API call gets a 401"] -.->|"fallback path only"| T1
    T0 -.->|"if nothing ever exercises this connection<br/>and no scheduler wakes it"| IDLE["refresh-token idle deadline<br/>Google 6 months, Xero 60 days"]
    IDLE --> DEAD
    KA["keep-alive sweep exists<br/>precisely to prevent this"] -.-> IDLE
    style DEAD fill:#f8d7da,color:#000
    style T1 fill:#d4edda,color:#000
    style IDLE fill:#fff3cd,color:#000
```

*The loop on the left is the healthy steady state. There are only two paths into the red box, and the idle-expiry path on the right is the one teams forget to build for, because it fires only on connections nobody is using.*

**The API call path.** Wrap outbound provider calls in something that attaches the current access token, and that on a 401 refreshes once and retries once — but only once, and only on 401, and with a clear terminal state if the retry also fails. Distinguish carefully between a 401 meaning "access token expired" and a 401 meaning "the user revoked you", because the first is routine and the second must stop all retries immediately. The signal is usually an `invalid_grant` error from the *token* endpoint during the refresh attempt: that means the refresh token itself is dead, the connection is unrecoverable, and continuing to retry will get your client ID rate-limited or blocked. Mark the connection as needing reauthorization, surface it to the user, and stop.

**Disconnection and revocation.** When a user disconnects, call the provider's revocation endpoint before deleting your local record. Deleting locally without revoking leaves a live credential that you no longer track, which is both a security problem and, in a SOC 2 or similar audit, an awkward one to explain. Handle the inverse too: users revoke access from the provider's own settings page, and most providers will not tell you, so the first you hear of it is a failing refresh.

**Observability.** Instrument the funnel — authorization starts, successful callbacks, denied consents, exchange failures — and the steady state — refresh success rate, token age distribution, count of connections in a failed state, 401 rate on API calls. Per provider, always, because these metrics are meaningless aggregated. A provider silently changing its token TTL or tightening its rate limits shows up here days before it shows up in support tickets. And never log a token, including in exception messages and HTTP client debug output, which is a more common leak path than it sounds.

## What changes when you support two hundred providers

Everything above describes one integration. The moment you are building a platform — an iPaaS, an agent tool layer, an embedded-integrations product — the problem inverts. Any individual provider is easy; what is hard is that a hundred easy problems, each slightly different, cannot be solved a hundred times. The strategic goal is to make the number of providers a *data* problem rather than a *code* problem, and to be realistic that you will never fully get there.

```mermaid
flowchart TB
    subgraph clients["Your product surfaces"]
        WebUI["Embedded connect UI"]
        Svc1["Product service"]
        Svc2["Agent runtime / tool layer"]
    end

    subgraph config["Provider definitions - data, not code"]
        Manifests[("Versioned manifests<br/>one document per provider")]
        Disco["RFC 8414 / OIDC discovery<br/>fetched and cached where supported"]
    end

    subgraph broker["Credential Broker - the only service that ever sees a secret"]
        Engine["Generic OAuth engine<br/>ONE implementation of authorize,<br/>exchange, refresh, revoke"]
        Hooks["Typed escape hatches<br/>pre_authorize, post_token,<br/>derive_identity, custom_refresh"]
        Proxy["Connect Proxy<br/>callers ask the broker to make the<br/>request; tokens never leave this box"]
        Sched["Refresh scheduler<br/>due-time queue, per-provider rate<br/>limits, jitter, backoff"]
        Tax["Error normalizer<br/>provider shapes to internal taxonomy"]
    end

    subgraph store["State"]
        Conns[("Connections<br/>encrypted tokens, granted scopes,<br/>status, next_refresh_at")]
        KMS{{"KMS<br/>envelope encryption, per-tenant DEKs"}}
    end

    subgraph ext["Providers"]
        P1(["Provider 1"])
        P2(["Provider 2"])
        Pn(["Provider N"])
    end

    WebUI --> Engine
    Svc1 --> Proxy
    Svc2 --> Proxy
    Manifests --> Engine
    Disco --> Engine
    Hooks -.->|"only where a manifest cannot express it"| Engine
    Engine --> Conns
    Proxy --> Conns
    Sched --> Conns
    Conns -.-> KMS
    Engine --> ext
    Proxy --> ext
    Sched --> ext
    Engine -.-> Tax
    Proxy -.-> Tax
    Sched -.-> Tax
    style broker fill:#1e3350,color:#dce7f5,stroke:#4f7aa8
    style Conns fill:#e1f5ff,color:#000
    style KMS fill:#fff3cd,color:#000
```

*Note what crosses the broker boundary: requests, never credentials. Provider definitions enter from the left as data, and the product services at the top never hold a token at all.*

The foundation is a **declarative provider manifest**. Every provider becomes a configuration document rather than a code module: auth type, endpoint URL templates with variables for things like a customer subdomain, the scope delimiter, where credentials go on the token request, how errors are shaped, what a cheap identity-probe call looks like, and the refresh policy. A single generic engine reads the manifest and executes the flow. The payoff is that adding a provider becomes a reviewable config change that a support engineer can make, and that cross-cutting improvements — a security fix in your PKCE handling, better refresh backoff — land for every provider at once instead of requiring a hundred patches. Appendix C sketches a manifest schema and Appendix E catalogs the deviations it needs to express.

You must pair that with a **narrow, typed escape hatch**, because a purely declarative model always breaks. Roughly the top eighty percent of providers fit a clean manifest; the remainder need real code — Salesforce returns an `instance_url` in the token response that you must use as the API base rather than a fixed hostname, Slack nests the user token under `authed_user` while putting the bot token at the top level, Atlassian requires a follow-up call to resolve a `cloudId` before any API call will work. Define a small set of named hooks with typed signatures — one to modify the authorization request, one to post-process the token response, one to derive the account identity, one to override refresh — and require every escape-hatch implementation to fit them. The discipline is that the hook interface is the contract; a provider that cannot be expressed through those hooks is a signal that the hook set needs one more member, not that this provider gets its own bespoke service.

```mermaid
flowchart LR
    M[/"provider.yaml<br/>endpoints, scope delimiter,<br/>client_auth style, token response paths,<br/>refresh policy, error map, identity probe"/]
    subgraph E["One generic engine, every provider"]
        direction TB
        R1["resolve endpoints<br/>template vars + discovery"]
        R2["build authorize URL"]
        R3["exchange code"]
        R4["map token response"]
        R5["schedule refresh"]
    end
    H1["pre_authorize"] -.-> R2
    H2["post_token"] -.-> R4
    H3["derive_identity"] -.-> R4
    H4["custom_refresh"] -.-> R5
    M --> R1
    R1 --> R2 --> R3 --> R4 --> R5
    R5 --> OUT[("Connection")]
    NOTE["Roughly 80% of providers need zero code.<br/>The rest fit these four typed hooks.<br/>If one genuinely does not, add a fifth hook -<br/>never a bespoke per-provider service."]
    style NOTE fill:#fff3cd,color:#000
    style M fill:#e1f5ff,color:#000
```

*The hooks are deliberately few and typed. A provider that fits none of them is a signal to add a fifth hook, not to fork the engine or stand up a bespoke service.*

Separate the **credential broker** from everything else. One service owns encryption keys, token storage, refresh scheduling, and the OAuth flows; no other service ever sees a raw token. The strongest form of this is to expose an authenticated proxy: callers ask the broker to make a request to a provider on behalf of a connection, and the broker attaches credentials and forwards it. Product code then never handles secrets at all, which collapses your audit surface to one service, makes key rotation a single-system operation, and means a compromise anywhere else in your platform does not yield provider credentials. This is the architecture Pipedream Connect uses with its Connect Proxy, and it is the right default for any platform where credentials belong to your customers' end users rather than to you.

```mermaid
sequenceDiagram
    autonumber
    participant Svc as Product service or agent
    participant Broker as Credential Broker
    participant KMS as KMS
    participant P as Provider API

    Svc->>Broker: POST /proxy<br/>connection_id, method, path, body
    Note over Svc: The caller holds no token and<br/>cannot enumerate credentials.
    Broker->>Broker: authorize - may this caller use this connection?
    Broker->>Broker: load connection, check expiry
    alt access token expired or near expiry
        Broker->>KMS: decrypt refresh token
        Broker->>P: POST /token - refresh, single-flight per connection
        P-->>Broker: new access token, possibly rotated refresh
        Broker->>KMS: encrypt and persist before proceeding
    end
    Broker->>P: forward the request with the Authorization header attached
    P-->>Broker: response
    Broker-->>Svc: response, headers scrubbed of credentials
    Note over Broker,P: Compromise anywhere else in the platform<br/>yields no provider credentials.
```

*The caller supplies a connection id and receives a response. Because it never holds a token, compromising that service yields nothing an attacker can reuse against the provider.*

**Refresh at platform scale is a scheduling problem, not an auth problem.** With a million live connections, refresh is a continuous background workload with hard constraints: provider token endpoints have their own rate limits, refresh-token rotation demands per-credential serialization, and different providers have wildly different TTLs. What works is a queue driven by a due-time index over credentials, with per-provider concurrency limits and token-bucket rate limiting, jittered scheduling so ten thousand connections created during one onboarding campaign do not all come due in the same second, exponential backoff with a cap on retryable failures, and immediate terminal classification of `invalid_grant` so dead connections leave the queue instead of consuming it. Separately, run a keep-alive sweep for providers whose refresh tokens expire on inactivity — Google's expire after six months unused, Xero's after sixty days — because a connection that nobody exercises will otherwise die quietly.

```mermaid
flowchart TB
    IDX[("connections WHERE status=ACTIVE<br/>ORDER BY next_refresh_at")] --> Q["Due-time queue"]
    KA["Keep-alive sweep for providers<br/>whose refresh tokens expire on idleness"] --> Q
    Q --> RL{"Per-provider token bucket<br/>and concurrency cap"}
    RL -->|"throttled"| DEFER["Defer with jitter"]
    DEFER --> Q
    RL -->|"budget available"| LK["Acquire per-connection lock"]
    LK --> RF["POST /token - refresh"]
    RF --> CL{"Classify the result"}
    CL -->|"success"| PERSIST["Persist tokens, reschedule at<br/>75% of the new TTL plus jitter"]
    PERSIST --> IDX
    CL -->|"transient"| BO["Exponential backoff, capped"]
    BO --> Q
    CL -->|"rate limited"| RA["Honour Retry-After"]
    RA --> Q
    CL -->|"invalid_grant"| TERM["NEEDS_REAUTH - leave the queue,<br/>notify the user, stop calling"]
    style TERM fill:#f8d7da,color:#000
    style PERSIST fill:#d4edda,color:#000
    style KA fill:#fff3cd,color:#000
```

*At a million connections, refresh is a queueing problem rather than an auth problem. The three exits on the right are the classification that keeps the queue from filling with corpses.*

Decide deliberately between a **managed OAuth app and bring-your-own client**, and then support both, because you will need to. Under the managed model you register one OAuth app per provider and all your customers ride on it; onboarding is frictionless, which is a genuine product advantage, but every customer shares your rate limits, the consent screen shows your brand rather than theirs, and provider app-review processes and per-app quotas become a shared dependency and a shared outage. Under BYO, each customer registers their own app and supplies the client ID and secret; they get their own quota and their own branding and their own compliance posture, at the cost of a setup flow that is genuinely painful for non-technical users. Mature platforms offer managed credentials for quick starts and BYO for production and enterprise, which means your manifest and storage model must treat client credentials as per-tenant from day one. Retrofitting that is expensive.

```mermaid
flowchart TB
    subgraph managed["Managed OAuth app - your client_id, shared by all customers"]
        direction TB
        M1["Frictionless: the user clicks Connect and is done"]
        M2["Every customer shares your rate limit<br/>and your app-review standing"]
        M3["The consent screen shows YOUR brand"]
        M4["One provider suspension takes down<br/>every customer at once"]
    end
    subgraph byo["Bring your own client - the customer's own client_id"]
        direction TB
        B1["The customer registers their own app"]
        B2["Their own quota, brand and compliance posture"]
        B3["Setup is genuinely painful for<br/>non-technical users"]
        B4["Blast radius is one customer"]
    end
    D["Design implication: treat client credentials as PER-TENANT from day one.<br/>Mature platforms ship managed for quick starts and BYO for enterprise.<br/>Retrofitting this into the manifest and storage model is expensive."]
    managed --> D
    byo --> D
    style M2 fill:#f8d7da,color:#000
    style M4 fill:#f8d7da,color:#000
    style B2 fill:#d4edda,color:#000
    style B4 fill:#d4edda,color:#000
    style D fill:#fff3cd,color:#000
```

*Both models are legitimate and most platforms end up needing both. The expensive mistake is assuming you will only ever need one.*

Build a **provider certification harness** and refuse to ship a provider without it. For each one, hold a sandbox account and run an automated end-to-end suite: complete the authorization flow, verify scopes come back as expected, make a real API call, force an access token expiry and verify refresh works, rotate the refresh token if the provider does and verify the new one persists, revoke from the provider side and verify you detect and classify it correctly. Run it on a schedule, not just at integration time, because providers change behavior without notice and a nightly signal is the difference between finding out from your dashboard and finding out from a customer. Version your manifests and roll them out gradually for the same reason.

```mermaid
flowchart LR
    A["Sandbox account held<br/>per provider"] --> B["1 - complete the<br/>full authorize flow"]
    B --> C["2 - assert granted scopes<br/>match expectation"]
    C --> D["3 - a real API call<br/>succeeds"]
    D --> E["4 - force access-token expiry,<br/>assert refresh works"]
    E --> F["5 - if rotating, assert the NEW<br/>refresh token was persisted"]
    F --> G["6 - revoke provider-side, assert<br/>we classify NEEDS_REAUTH"]
    G --> H(["Manifest promoted<br/>to a new version"])
    H --> I["Gradual rollout<br/>by tenant percentage"]
    N["Runs nightly, not only at integration time.<br/>Providers change behaviour without notice; this is the<br/>difference between your dashboard telling you and a customer telling you."]
    style H fill:#d4edda,color:#000
    style N fill:#fff3cd,color:#000
```

*Step 6 is the one most teams skip, and it is the one that catches a provider silently changing how it signals revocation.*

Finally, invest in **error normalization**, because raw provider errors are unusable at scale. Providers signal the same condition through wildly different shapes — some return the spec's `invalid_grant`, some return HTTP 200 with `{"ok": false}`, some return HTML. Map everything into a small internal taxonomy: retryable-transient, rate-limited-with-retry-after, needs-reauthorization, misconfigured-integration, permanent-provider-error. Everything downstream — retry logic, alerting, the message the end user sees, whether a connection is marked dead — keys off that taxonomy rather than off provider-specific strings. This single piece of normalization does more for operability than any other, because it is what lets you run one runbook instead of two hundred.

```mermaid
flowchart LR
    subgraph raw["What providers actually return for the same conditions"]
        direction TB
        R1["400 invalid_grant<br/>spec-compliant"]
        R2["200 OK with ok:false<br/>and an error string"]
        R3["401 with an HTML body"]
        R4["429 with Retry-After"]
        R5["403 REQUEST_LIMIT_EXCEEDED"]
        R6["503 or a connection timeout"]
    end
    subgraph tax["Your internal taxonomy"]
        direction TB
        T1["NEEDS_REAUTH"]
        T2["RATE_LIMITED"]
        T3["TRANSIENT"]
        T4["MISCONFIGURED"]
        T5["PROVIDER_PERMANENT"]
    end
    subgraph beh["Downstream behaviour"]
        direction TB
        B1["stop retrying, mark dead, notify the user"]
        B2["honour Retry-After, requeue"]
        B3["exponential backoff, capped"]
        B4["page the integration owner"]
        B5["circuit-break this provider"]
    end
    R1 --> T1
    R2 --> T1
    R3 --> T5
    R4 --> T2
    R5 --> T2
    R6 --> T3
    T1 --> B1
    T2 --> B2
    T3 --> B3
    T4 --> B4
    T5 --> B5
    style raw fill:#3a2226,color:#f0dcdf,stroke:#8a5560
    style tax fill:#1e3350,color:#dce7f5,stroke:#4f7aa8
    style beh fill:#1f3a28,color:#daeede,stroke:#4f8a62
```

*The left column is what you actually receive. The right column is the only thing your retry logic, alerting and user-facing messages should ever see.*

## Where this is heading

Two currents are worth tracking. The first is that the bearer-token compromise is slowly being reversed. **OAuth 2.1** consolidates twelve years of errata into a coherent baseline: PKCE mandatory for all authorization code flows, implicit and ROPC removed, exact string matching on redirect URIs, refresh tokens for public clients either sender-constrained or one-time-use, and bearer tokens forbidden in query strings. Alongside it, **DPoP** (RFC 9449) binds a token to a key the client proves possession of on every request, and **mTLS-bound tokens** (RFC 8705) do the same with client certificates. Both make a stolen token useless on its own, which is to say both are reintroducing OAuth 1.0a's core property using modern cryptography and without the canonicalization misery. If you are building new, assume the 2.1 baseline; it is a strict subset of what providers already support, so you lose nothing by adopting it early.

```mermaid
flowchart LR
    O1["OAuth 1.0a - 2009<br/>every request signed<br/>a stolen token is useless<br/>without the secret<br/>--<br/>safe on plain HTTP<br/>brutal to implement"]
    O2["OAuth 2.0 - 2012<br/>bearer tokens plus TLS<br/>--<br/>easy to implement<br/>easy to implement insecurely"]
    O3["Hardening - 2015 to 2025<br/>PKCE, RFC 9700 security BCP,<br/>OAuth 2.1, DPoP, mTLS binding"]
    O4["Sender-constrained again<br/>a stolen token is useless without<br/>the proof-of-possession key"]
    O1 -->|"drop signatures<br/>to win adoption"| O2
    O2 -->|"walk back the<br/>dangerous parts"| O3
    O3 --> O4
    O4 -.->|"the same property 1.0a had, with modern crypto<br/>and none of the canonicalization misery"| O1
    style O2 fill:#f8d7da,color:#000
    style O4 fill:#d4edda,color:#000
    style O1 fill:#e1f5ff,color:#000
```

*The dotted return arrow is the point. The industry spent a decade trading away OAuth 1.0a's core guarantee for implementability, and is now rebuilding that guarantee with cryptography that does not require byte-exact canonicalization.*

The second is that AI agents have made this whole area load-bearing in a way it was not five years ago. An agent that acts across dozens of a user's SaaS tools needs exactly what OAuth provides — scoped, revocable, per-user, auditable delegation — but at a volume and with a dynamism that hand-built integrations cannot serve. The Model Context Protocol's authorization spec leans directly on this stack: OAuth 2.1 with mandatory PKCE, RFC 8414 metadata discovery, RFC 9728 protected-resource metadata so a client can find the right authorization server from the resource it is trying to reach, and RFC 7591 dynamic client registration so a client the provider has never seen can register itself without a human filling in a form. That combination — discovery plus dynamic registration — is what the multi-provider problem looks like when it is solved properly at the specification layer rather than by maintaining two hundred manifests. It is not yet widely deployed, and manifests remain the practical answer today, but it is the direction, and it is worth knowing that the industry's answer to "how do I support a huge number of providers" is ultimately "make the providers describe themselves".

```mermaid
sequenceDiagram
    autonumber
    participant Agent as AI agent / MCP client
    participant RS as MCP server - the resource
    participant AS as Authorization Server

    Agent->>RS: request with no credentials
    RS-->>Agent: 401 with WWW-Authenticate pointing at<br/>resource metadata - RFC 9728
    Agent->>RS: GET /.well-known/oauth-protected-resource
    RS-->>Agent: which authorization server governs this resource
    Agent->>AS: GET /.well-known/oauth-authorization-server - RFC 8414
    AS-->>Agent: endpoints, supported grants, PKCE methods
    Agent->>AS: POST /register - RFC 7591 dynamic client registration
    AS-->>Agent: a client_id, issued to a client never seen before
    Agent->>AS: Authorization Code + PKCE - OAuth 2.1
    AS-->>Agent: access token
    Agent->>RS: retry with the bearer token
    RS-->>Agent: 200 OK
    Note over Agent,AS: No human registered an app. No manifest was written.<br/>This is the many-providers problem solved at the spec layer.
```

*No human registered an application and no engineer wrote a manifest. This is what the many-providers problem looks like when it is solved at the specification layer instead of by maintaining configuration.*

---

## Appendices

### Appendix A — OAuth 1.0a signature construction

The signature base string is three percent-encoded components joined by `&`: the uppercased HTTP method, the normalized request URL (scheme and host lowercased, default port omitted, query string and fragment stripped), and the normalized parameter string.

Parameter normalization collects the OAuth protocol parameters, the URL query parameters, and — only when `Content-Type` is `application/x-www-form-urlencoded` — the body parameters. `oauth_signature` itself is excluded. Each key and value is percent-encoded per RFC 3986, leaving only `A-Z a-z 0-9 - . _ ~` unescaped. Pairs are sorted by encoded key, ties broken by encoded value, then joined as `k=v` with `&`. The whole resulting string is then percent-encoded again as the third component of the base string.

```mermaid
flowchart TB
    subgraph collect["1 - Collect parameters"]
        direction TB
        C1["OAuth protocol params:<br/>oauth_consumer_key, oauth_nonce,<br/>oauth_timestamp, oauth_token, oauth_version"]
        C2["URL query params"]
        C3["Body params - ONLY if Content-Type is<br/>application/x-www-form-urlencoded<br/>(a JSON POST signs an empty body)"]
    end
    collect --> EXCL["2 - Exclude oauth_signature itself"]
    EXCL --> ENC["3 - Percent-encode every key and value per RFC 3986<br/>only A-Z a-z 0-9 - . _ ~ stay literal<br/>your language's urlencode is probably wrong"]
    ENC --> SORT["4 - Sort by encoded key, ties broken by encoded value"]
    SORT --> JOIN["5 - Join as k=v, separated by &"]
    JOIN --> BASE["6 - Base string =<br/>METHOD & pct(normalized URL) & pct(param string)"]
    KEY["Signing key =<br/>pct(consumer_secret) & pct(token_secret)<br/>the trailing & is required<br/>even when there is no token secret"]
    BASE --> HMAC["7 - BASE64(HMAC-SHA1(signing key, base string))"]
    KEY --> HMAC
    HMAC --> HDR["8 - Authorization: OAuth ... oauth_signature=pct(sig)"]
    style ENC fill:#fff3cd,color:#000
    style C3 fill:#fff3cd,color:#000
    style BASE fill:#e1f5ff,color:#000
```

*Step 3, and the body-parameter condition in step 1, are where nearly every mysterious 401 originates.*

The canonical example from RFC 5849 §3.4.1.1:

```text
POST&http%3A%2F%2Fexample.com%2Frequest&a2%3Dr%2520b%26a3%3D2%2520q
%26a3%3Da%26b5%3D%253D%25253D%26c%2540%3D%26c2%3D%26oauth_consumer_
key%3D9djdj82h48djs9d2%26oauth_nonce%3D7d8f3e4a%26oauth_signature_m
ethod%3DHMAC-SHA1%26oauth_timestamp%3D137131201%26oauth_token%3Dkkk
9d7dh3k39sjv7
```

The signing key is `percent_encode(consumer_secret) + "&" + percent_encode(token_secret)`, and the trailing `&` is required even when there is no token secret — as on the initial request-token call. The signature is `BASE64(HMAC-SHA1(key, base_string))`.

The result is sent as an `Authorization` header with every parameter value percent-encoded and quoted:

```http
Authorization: OAuth realm="Example",
  oauth_consumer_key="9djdj82h48djs9d2",
  oauth_token="kkk9d7dh3k39sjv7",
  oauth_signature_method="HMAC-SHA1",
  oauth_timestamp="137131201",
  oauth_nonce="7d8f3e4a",
  oauth_version="1.0",
  oauth_signature="bYT5CMsGcbgUdFHObYMEfcx6bsw%3D"
```

The three-legged flow in full:

| Step | Call | Sends | Receives |
| --- | --- | --- | --- |
| 1 | `POST /oauth/request_token` | signed with consumer secret only, `oauth_callback` | `oauth_token`, `oauth_token_secret` (temporary) |
| 2 | browser → `GET /oauth/authorize?oauth_token=…` | user authenticates and approves | redirect to callback with `oauth_token`, `oauth_verifier` |
| 3 | `POST /oauth/access_token` | signed with consumer + temp token secret, `oauth_verifier` | `oauth_token`, `oauth_token_secret` (long-lived) |

Debugging checklist, in the order that resolves the most cases: print the base string and diff against the provider's example; confirm your percent-encoder is RFC 3986 and not `application/x-www-form-urlencoded`; confirm host clock is NTP-synced; confirm you excluded `oauth_signature` from the parameter set; confirm body parameters are included if and only if the content type is form-encoded; confirm the trailing `&` in the signing key.

### Appendix B — Authorization Code + PKCE wire trace

Generate `code_verifier` as 43–128 characters from the unreserved set, and `code_challenge = BASE64URL(SHA256(code_verifier))` with padding stripped.

Front channel, browser redirect:

```http
GET https://provider.example.com/oauth/authorize
  ?response_type=code
  &client_id=abc123
  &redirect_uri=https%3A%2F%2Fapp.example.com%2Foauth%2Fcallback
  &scope=read%3Aitems%20write%3Aitems%20offline_access
  &state=Hk7fT2pQxR9mN4vL
  &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
  &code_challenge_method=S256
```

Provider redirects back:

```http
GET https://app.example.com/oauth/callback
  ?code=SplxlOBeZQQYbYS6WxSbIA
  &state=Hk7fT2pQxR9mN4vL
```

Back channel, server to server:

```http
POST /oauth/token HTTP/1.1
Host: provider.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic YWJjMTIzOnNlY3JldA==

grant_type=authorization_code
&code=SplxlOBeZQQYbYS6WxSbIA
&redirect_uri=https%3A%2F%2Fapp.example.com%2Foauth%2Fcallback
&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
```

Response:

```json
{
  "access_token": "2YotnFZFEjr1zCsicMWpAA",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "tGzv3JOkF0XG5Qx2TlKWIA",
  "scope": "read:items write:items"
}
```

Refresh:

```http
POST /oauth/token
grant_type=refresh_token
&refresh_token=tGzv3JOkF0XG5Qx2TlKWIA
```

Note in the response above that `scope` came back narrower than requested — `offline_access` is absent. Always persist the returned scope, never the requested one.

### Appendix C — Provider manifest schema sketch

```yaml
id: salesforce
display_name: Salesforce
auth_type: oauth2_authorization_code

endpoints:
  # {{subdomain}} resolved from per-connection user input
  authorize: "https://{{subdomain}}.my.salesforce.com/services/oauth2/authorize"
  token:     "https://{{subdomain}}.my.salesforce.com/services/oauth2/token"
  revoke:    "https://{{subdomain}}.my.salesforce.com/services/oauth2/revoke"
  # or, when supported, prefer discovery:
  # discovery: "https://accounts.google.com/.well-known/openid-configuration"

connection_inputs:
  - key: subdomain
    label: "Your Salesforce domain"
    required: true

scopes:
  delimiter: " "          # " " | "," | "+"
  default: [api, refresh_token, openid]
  offline_scope: refresh_token   # the scope that unlocks a refresh token

authorize_params:
  extra: { prompt: consent }     # Google also needs access_type: offline
  pkce: S256                     # S256 | plain | none

token_request:
  client_auth: basic             # basic | body | none
  accept_header: application/json  # GitHub needs this or it returns form-encoded

token_response:
  access_token_path: access_token
  refresh_token_path: refresh_token
  expires_in_path: expires_in
  # extras lifted into connection metadata and usable in API base URLs
  metadata:
    api_base: instance_url

refresh:
  policy: rotating               # none | static | rotating
  refresh_token_ttl: null
  idle_expiry: null              # e.g. P60D for Xero, P6M for Google

identity_probe:
  method: GET
  url: "{{api_base}}/services/oauth2/userinfo"
  account_id_path: user_id
  account_label_path: email

errors:
  needs_reauth: [invalid_grant, expired_token]
  rate_limited: [REQUEST_LIMIT_EXCEEDED]

hooks:
  post_token: null               # named, typed escape hatch when needed
```

### Appendix D — Refresh with concurrency control

```java
AccessToken getValidToken(ConnectionId id) {
  Connection c = cache.get(id);
  if (c != null && c.expiresAt().isAfter(now().plus(SKEW))) return c.accessToken();

  // Single-flight: only one refresher per connection, cluster-wide.
  try (Lock lock = locks.acquire("refresh:" + id, Duration.ofSeconds(30))) {
    // Re-read inside the lock; a peer may have refreshed while we waited.
    c = store.load(id);
    if (c.expiresAt().isAfter(now().plus(SKEW))) return c.accessToken();

    if (c.status() == NEEDS_REAUTH) throw new ReauthRequired(id);

    TokenResponse r;
    try {
      r = provider(c).refresh(c.refreshToken());
    } catch (OAuthError e) {
      if (taxonomy.classify(e) == NEEDS_REAUTH) {
        store.markNeedsReauth(id, e);      // terminal: stop all retries
        notifier.connectionBroken(id);
        throw new ReauthRequired(id);
      }
      throw e;                              // transient: caller backs off
    }

    // Rotating providers return a new refresh token; losing it kills the
    // connection, so persist atomically before returning.
    store.saveTokens(id, r.accessToken(), r.refreshTokenOr(c.refreshToken()),
                     now().plusSeconds(r.expiresIn()));
    scheduler.reschedule(id, refreshAt(r.expiresIn()));
    cache.put(id, ...);
    return r.accessToken();
  }
}

// Proactive: ~75% of TTL, jittered to avoid synchronized stampedes.
Instant refreshAt(long ttlSeconds) {
  long base = (long) (ttlSeconds * 0.75);
  long jitter = ThreadLocalRandom.current().nextLong(-base / 10, base / 10);
  return now().plusSeconds(base + jitter);
}
```

Three rules this encodes. Re-read state inside the lock, or you serialize correctly and still double-refresh. Persist a rotated refresh token before doing anything else with the response, since a crash between receipt and persistence permanently orphans the connection. Classify `invalid_grant` as terminal immediately, because retrying it is how you get your client ID throttled or blocked.

### Appendix E — Provider deviation catalog

| Provider | Deviation that will cost you an afternoon |
| --- | --- |
| Google | Refresh token returned only on first consent unless `access_type=offline&prompt=consent`; expires after 6 months unused; apps in Testing status get 7-day refresh tokens |
| Microsoft Entra | Refresh tokens rotate every use; `offline_access` scope required; v1 `resource` vs v2 `scope` parameter; tenant-specific vs `common` authority changes the token |
| Slack | Bot token at `access_token`, user token nested at `authed_user.access_token`; errors return HTTP 200 with `{"ok": false, "error": "..."}`; rotation is opt-in |
| Salesforce | `instance_url` in the token response is the required API base; per-org custom domains; sandbox uses a different host entirely |
| Shopify | Each shop is its own authorization server; offline tokens never expire, online tokens die with the user session in 24h |
| GitHub | Returns form-encoded unless you send `Accept: application/json`; comma-delimited scopes; OAuth App tokens historically non-expiring, GitHub App user tokens expire in 8h |
| Xero | 30-minute access tokens, 60-day rotating refresh tokens; must call `/connections` to resolve tenant IDs before any API call |
| QuickBooks | `realmId` arrives as a callback query param and is required for every API call; 100-day rotating refresh token |
| Atlassian | Must call `/oauth/token/accessible-resources` to resolve `cloudId` before the API base URL exists |
| Dropbox | Short-lived tokens only if you pass `token_access_type=offline`; otherwise you get a legacy long-lived token |
| Zoom | One-time-use rotating refresh tokens; reuse invalidates the chain |
| X / Twitter | Both OAuth 1.0a and OAuth 2.0 surfaces; 2.0 requires PKCE and `offline.access` for refresh |
| HubSpot | Requested scopes must exactly match the app configuration or the authorize call fails before the consent screen |
| Stripe Connect | Returns `stripe_user_id`, which is the account identifier you actually key on |

The pattern across the table: the deviations cluster in four places — how you get a refresh token at all, where the API base URL comes from, how scopes are delimited and validated, and what a rotating refresh token does to your concurrency model. A manifest schema that handles those four handles most of the long tail.

### Appendix F — Storage model sketch

```mermaid
erDiagram
    TENANT ||--o{ OAUTH_CONNECTION : owns
    TENANT ||--o{ OAUTH_FLOW_STATE : "has in flight"
    PROVIDER ||--o{ OAUTH_CONNECTION : "connected via"
    PROVIDER ||--|| PROVIDER_MANIFEST : "described by"
    OAUTH_CONNECTION ||--o{ REFRESH_ATTEMPT : records

    OAUTH_CONNECTION {
        uuid id PK
        uuid tenant_id FK
        text provider_id FK
        text provider_account_id "from the identity probe"
        text client_id "managed or BYO"
        bytea access_token_enc "treat as cache"
        bytea refresh_token_enc "password-equivalent"
        text dek_id "envelope key reference"
        textarray granted_scopes "as returned, not as requested"
        timestamptz access_expires_at
        timestamptz refresh_expires_at
        timestamptz next_refresh_at "drives the scheduler"
        text status "ACTIVE NEEDS_REAUTH REVOKED"
        text last_error_code
    }
    OAUTH_FLOW_STATE {
        text state PK "random, single use"
        text code_verifier "never sent to the browser"
        text session_id "binds to the browser session"
        text return_to
        timestamptz consumed_at
        timestamptz expires_at "short TTL"
    }
    PROVIDER_MANIFEST {
        text provider_id PK
        int version
        text auth_type
        jsonb endpoints
        jsonb refresh_policy
        jsonb error_map
    }
    REFRESH_ATTEMPT {
        uuid id PK
        uuid connection_id FK
        timestamptz at
        text outcome "feeds per-provider dashboards"
    }
```

*Two details carry real weight: granted_scopes stores what the provider returned rather than what you requested, and next_refresh_at is indexed because it is the scheduler's work queue.*

```sql
CREATE TABLE oauth_connection (
  id                  UUID PRIMARY KEY,
  tenant_id           UUID NOT NULL,
  user_id             UUID,                    -- null for tenant-level connections
  provider_id         TEXT NOT NULL,
  provider_account_id TEXT NOT NULL,           -- from the identity probe
  provider_account_label TEXT,                 -- shown in the UI
  client_id           TEXT NOT NULL,           -- managed or BYO
  access_token_enc    BYTEA NOT NULL,
  refresh_token_enc   BYTEA,
  dek_id              TEXT NOT NULL,           -- envelope encryption key ref
  granted_scopes      TEXT[] NOT NULL,         -- as returned, not as requested
  access_expires_at   TIMESTAMPTZ,
  refresh_expires_at  TIMESTAMPTZ,
  next_refresh_at     TIMESTAMPTZ,             -- drives the scheduler
  status              TEXT NOT NULL,           -- ACTIVE | NEEDS_REAUTH | REVOKED
  last_error_code     TEXT,
  last_used_at        TIMESTAMPTZ,
  last_refreshed_at   TIMESTAMPTZ,
  created_at          TIMESTAMPTZ NOT NULL,
  UNIQUE (tenant_id, provider_id, provider_account_id)
);
CREATE INDEX ON oauth_connection (next_refresh_at) WHERE status = 'ACTIVE';

CREATE TABLE oauth_flow_state (        -- short-TTL, one row per in-flight auth
  state          TEXT PRIMARY KEY,
  code_verifier  TEXT NOT NULL,
  session_id     TEXT NOT NULL,
  tenant_id      UUID NOT NULL,
  provider_id    TEXT NOT NULL,
  return_to      TEXT,
  consumed_at    TIMESTAMPTZ,
  expires_at     TIMESTAMPTZ NOT NULL
);
```

### Appendix G — Security checklist

- PKCE on every authorization code flow, confidential clients included.
- `state` generated with a CSPRNG, bound to the session, single-use, short TTL, deleted on consumption.
- Redirect URIs matched by exact string comparison on the provider side; no wildcards, no open redirects downstream of the callback.
- Refresh tokens encrypted at rest under a KMS-managed key; access tokens treated as cache.
- Client secrets in a secret manager, rotatable without a deploy.
- Tokens never in URLs, query strings, logs, exception messages, or HTTP client debug output.
- ID tokens validated on signature, `iss`, `aud`, `exp`, and `nonce` — never trust an access token or a userinfo response as proof of identity.
- Minimum viable scopes, requested incrementally; persist what was granted, not what was asked.
- Revocation called on user disconnect, before local deletion.
- `invalid_grant` classified as terminal; no retry storms against token endpoints.
- Per-provider rate limiting and circuit breaking on both token and resource endpoints.

### Appendix H — Specifications and references

Core: RFC 6749 (OAuth 2.0 framework), RFC 6750 (bearer token usage), RFC 5849 (OAuth 1.0a), RFC 7636 (PKCE).

Operational: RFC 7009 (revocation), RFC 7662 (introspection), RFC 8414 (authorization server metadata), RFC 9728 (protected resource metadata), RFC 7591 (dynamic client registration).

Grants and extensions: RFC 8628 (device authorization), RFC 7523 (JWT bearer), RFC 8693 (token exchange), RFC 9126 (pushed authorization requests), RFC 9396 (rich authorization requests).

Hardening: RFC 9700 (OAuth 2.0 Security Best Current Practice), RFC 9449 (DPoP), RFC 8705 (mTLS client authentication and certificate-bound tokens), the OAuth 2.1 draft.

Identity: OpenID Connect Core 1.0 and Discovery 1.0.

Practical reading: `oauth.net`, the IETF `oauth-wg` drafts, and Aaron Parecki's *OAuth 2.0 Simplified*. For the multi-provider architecture specifically, the public documentation for Pipedream Connect, Nango, and Merge is worth reading as three different answers to the same manifest-and-broker problem.
