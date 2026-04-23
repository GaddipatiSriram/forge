# Keycloak / IAM Learning Q&A

Study log for understanding Identity and Access Management before diving
deeper into Keycloak configuration. Kept so the conceptual foundation is
searchable and can be revisited when the abstractions in the admin UI stop
making sense.

Reading-time budget to solid mental model: ~5 hours. Plus 2-3 hours hands-on.

---

## Q1. Why IAM exists at all — what problem is Keycloak solving?

**A.** Without centralized identity, every application owns its own user
table, password storage, and user lifecycle. Consequences:

1. **Security** — each app is a breach surface with its own password hashing,
   its own rate limiting, its own (often missing) MFA. One sloppy app leaks
   credentials that users reused elsewhere.
2. **UX** — users juggle N passwords, N login forms.
3. **Ops / HR** — adding a new hire or offboarding a leaver means touching N
   systems.

A central identity provider (IdP) like Keycloak consolidates this: apps stop
storing credentials and delegate login to the IdP. MFA, password policy, user
lifecycle, and audit all live in one place.

The four pillars of a full IAM system:

- **Identity store** — where user records live (DB, LDAP, federated).
- **Authentication** — login UX, MFA, brute-force protection.
- **SSO / federation** — log in once, access many apps; or accept identity
  from Google/GitHub/AD.
- **Authorization** — roles, groups, policies that decide what the user can do.

Keycloak covers all four.

Recommended intro reads (each ~15 min): Okta "What is IAM" overview, Auth0
"Identity 101". Either one is enough to get the vocabulary.

---

## Q2. AuthN vs AuthZ — why do protocols keep these separate?

**A.**

- **Authentication (AuthN)**: proves *who you are*. Protocols: OIDC, SAML.
- **Authorization (AuthZ)**: decides *what you can do*. Mechanisms: OAuth 2.0
  scopes, roles, policies (XACML, OPA, Keycloak Authorization Services).

They are distinct because a system may need one without the other:

- A background job accessing an API needs AuthZ (access token) but has no
  human user to authenticate.
- A corporate directory lookup may need AuthN (is this really Alice?)
  without any specific resource being authorized.

Keep these mentally separate. Every modern "login" protocol is actually
both glued together: OIDC is "OAuth 2.0 (authorization) + an identity
layer (authentication)".

---

## Q3. Where should I start with protocols — OAuth, OIDC, or SAML?

**A.** Start with **OAuth 2.0**. It's the foundation that everything else
builds on (OIDC sits on top of it; even some modern SAML integrations look
OAuth-ish). SAML 2.0 is older, XML-heavy, and still everywhere in the
enterprise SaaS world — but you'll understand it much faster once you know
OAuth + OIDC.

**The one OAuth resource worth reading**: `oauth.com` by Aaron Parecki. Free
online book, ~100 pages, clearest explanation on the internet. Minimum
required reading:

- "The Protocol Flow"
- "Authorization Code Grant"
- "Access Tokens"
- "Refresh Tokens"
- "PKCE"

Concepts to fully internalize before moving on:

- **Four roles**: resource owner (user), client (app), authorization server
  (Keycloak), resource server (API).
- **Authorization Code flow + PKCE** — the default for web apps and SPAs.
  90% of what you'll ever configure.
- **Client credentials flow** — machine-to-machine; no user involved.
- **Device code flow** — for devices without a browser (e.g., a TV or CLI).
- **Scopes** — what access the user is consenting to.
- **Why implicit flow is deprecated** — PKCE solves the problems it tried to.
- **Refresh tokens** — how to stay logged in without storing passwords.

Rough time: 2 hours.

---

## Q4. What does OpenID Connect (OIDC) add on top of OAuth 2.0?

**A.** OAuth 2.0 by itself only handles *authorization* — it gets you an
opaque access token that grants access to some resource. It does NOT tell
the client who the user is.

OIDC adds an identity layer:

- **ID token** — a JWT (signed, self-contained) asserting who the user is.
  The client reads the claims directly (`sub`, `email`, `preferred_username`,
  etc.) without needing to call another endpoint.
- **`/userinfo` endpoint** — get additional user attributes.
- **`/.well-known/openid-configuration`** — the discovery document clients
  use to auto-configure. You saw this today when we debugged the issuer URL
  mismatch.

Resources: OpenID Foundation's "OIDC Core" spec overview, or Auth0's "Intro
to OpenID Connect" (easier read). Focus on:

- Difference between **access token** (bearer, sent to APIs) and **ID token**
  (identity proof, read by the client).
- **JWT structure** — header.payload.signature, base64url-encoded.
- **JWT validation** — signature check, `iss`, `aud`, `exp`, `nbf` claims.
- **`jwt.io`** — paste a real token and look inside. Nothing demystifies
  JWTs faster.

Rough time: 1 hour.

---

## Q5. How does SAML fit in, and do I need to learn it now?

**A.** SAML 2.0 is an older (2005) XML-based SSO protocol. It's dominant in
enterprise SaaS (Salesforce, ServiceNow, many Atlassian-era apps) but
painful to work with — XML, XML signatures, SP-initiated vs IdP-initiated
flows.

For a homelab platform focused on modern apps (ArgoCD, Vault, Harbor, Argo
Workflows, SonarQube), **OIDC is enough**. All of those speak OIDC natively.
Skip SAML until you actually need to federate with an enterprise SaaS that
forces it.

When you do learn it later: the mental model is similar to OIDC (IdP + SP,
assertions instead of tokens) but the wire format and signing rules are
wildly different.

---

## Q6. What are Keycloak's core abstractions I need to understand?

**A.** Read the **Keycloak Server Administration Guide** at `keycloak.org/guides`.
Focus on these chapters first:

- **Realms** — top-level tenant boundary. Users, clients, roles, and
  sessions are realm-scoped. `master` is for administering Keycloak itself;
  apps should use their own realm (which is why we're standing up `forge`).
  Users in one realm cannot see or authenticate into another.
- **Clients** — each application that authenticates users or consumes tokens.
  Client types:
  - **Public** — browser-delivered code (SPA, mobile). No client secret.
    Uses PKCE.
  - **Confidential** — server-side app that can safely hold a secret.
  - **Bearer-only** — APIs that only validate incoming tokens and never
    initiate login.
- **Users, roles, groups** — authorization primitives.
  - Roles: realm-level or client-level.
  - Groups: containers of users that can have roles attached.
  - Composite roles: roles that include other roles.
- **Protocol mappers** — transformations from user/session data into JWT
  claims. E.g., "put the user's group membership into a `groups` claim" so
  your app can read it.
- **Authentication flows** — the ordered steps Keycloak runs during login
  (username → password → optional OTP → conditional MFA). Fully
  configurable per realm.
- **Identity providers (federation)** — Keycloak delegating authentication
  to Google, GitHub, LDAP, AD, or another OIDC/SAML IdP. "Login with
  Google" is Keycloak talking OIDC to Google on your behalf.
- **Client scopes** — reusable bundles of protocol mappers + role mappings
  assigned to clients.

Rough time: 1 hour.

---

## Q7. What hands-on exercises will make it stick?

**A.** Don't just read. On the existing mgmt-forge Keycloak:

1. **Create a playground realm and test user.**
   - Realm: `playground`
   - User: `alice` with a known password
   - No impact on any real app; can be deleted freely.

2. **Drive Authorization Code flow by hand.**
   - Create a **public** client in `playground` with redirect URI
     `http://localhost:8080/callback`.
   - Open the auth URL in a browser:
     `https://keycloak.apps.mgmt-forge.engatwork.com/auth/realms/playground/protocol/openid-connect/auth?client_id=...&response_type=code&redirect_uri=http://localhost:8080/callback&scope=openid&code_challenge=...&code_challenge_method=S256`
   - Log in as alice. Browser redirects to `localhost:8080/callback?code=...` —
     copy the code out of the URL bar (no server needed).
   - Exchange the code for tokens via `curl` to the `/token` endpoint.
   - Paste the resulting `id_token` into `jwt.io` and inspect claims.

3. **Client credentials flow.**
   - Create a **confidential** client (service account enabled).
   - `curl -d "grant_type=client_credentials&client_id=...&client_secret=..."`
     to the `/token` endpoint.
   - No user involved — this is how a backend service authenticates itself.
   - Inspect the access token; note `sub` is the service account, not a user.

4. **Decode tokens obsessively.** Every time you touch an OIDC interaction,
   paste the tokens into `jwt.io` and read the claims. Over a few sessions
   you'll stop thinking of tokens as opaque.

5. **Break things deliberately.**
   - Set redirect URI to the wrong value — see the error Keycloak returns.
   - Let a token expire — observe the behavior.
   - Revoke a session from the admin console — see what happens on next
     API call.

Today (during Keycloak bring-up) you already did one of these inadvertently:
the `curl -X POST grant_type=password` that fetched a JWT for `admin` was an
OAuth 2.0 password grant against the master realm. Password grant is
deprecated but illustrative.

Rough time: 1-2 hours to complete all five.

---

## Q8. Why is this ordering — problem space → OAuth → OIDC → Keycloak — better than jumping straight to Keycloak?

**A.** The Keycloak admin UI is a thin wrapper over OIDC, SAML, and OAuth
2.0 concepts. Without the protocol layer underneath, every checkbox is
magic: "what's a 'client scope'?", "what's 'Direct Access Grants Enabled'?",
"what does 'implicit flow' mean?"

With the protocol layer, the UI becomes a direct mapping: pick your flow,
declare redirects, pick your claims. Configuration that seemed arbitrary
starts reading as "this exposes OIDC parameter X".

Trying to learn Keycloak without OAuth/OIDC is like reading nginx docs
without knowing HTTP. You can copy-paste your way to something that works
but you can't debug it and you can't design for it.

---

## Q9. After the basics — what are the next layers to study?

**A.** In rough priority, for what this platform will eventually need:

1. **Token validation in consuming apps** — every app that accepts tokens
   has to verify the signature using Keycloak's public keys from the JWKS
   endpoint, check `iss`, `aud`, `exp`, `nbf`. Libraries do this but
   understanding what they do matters when something breaks.
2. **Front-channel vs back-channel logout** — how SSO logout propagates
   across apps. Relevant when multiple apps share a Keycloak realm.
3. **Keycloak Authorization Services** — fine-grained resource + permission
   model inside Keycloak (UMA 2.0). Probably skip until needed; for most
   platforms, role-based access on the client side is enough.
4. **mTLS, token binding, DPoP** — proof-of-possession for access tokens.
   Advanced; needed for sensitive APIs.
5. **Admin REST API + keycloak-config-cli** — how we'll manage realms
   declaratively from Git. Once you understand the concepts, reading the
   admin API is mostly "match concept X to endpoint Y".
6. **SAML** — when you need to federate with an enterprise IdP that
   insists on it.

---

## Q10. Glossary — terms that will show up everywhere.

- **IdP** — Identity Provider. The thing that authenticates users. Keycloak
  in our stack.
- **SP** — Service Provider (SAML term). The app that trusts the IdP.
- **RP** — Relying Party (OIDC term). Same idea as SP.
- **Client** — OIDC/OAuth term for the app that asks for tokens. Can be an
  RP and/or a consumer of access tokens.
- **Bearer token** — a token that whoever holds it can use. Possession =
  authority. Hence: don't leak them.
- **JWT** — JSON Web Token. Self-contained, signed. Used for ID tokens,
  sometimes access tokens, API keys, etc.
- **JWKS** — JSON Web Key Set. Public keys published at
  `/.well-known/jwks.json` used to verify JWT signatures.
- **Claims** — fields inside a JWT (`sub`, `iss`, `aud`, `exp`, custom ones).
- **Scope** — OAuth concept: a named permission the client is asking for.
- **Consent** — user-visible screen where they approve the scopes a client
  is requesting.
- **Federation** — IdP delegating login to another IdP.
- **Realm** — Keycloak-specific: tenant boundary.

---

## Resources cheat sheet

- **oauth.com** — Aaron Parecki's free book on OAuth 2.0.
- **keycloak.org/guides** — official Keycloak docs (surprisingly good).
- **openid.net/developers/how-connect-works** — OIDC primer.
- **jwt.io** — decode/inspect JWTs interactively.
- **jose.readthedocs.io** — deeper dive into JWT signing/encryption.
- Auth0 blog and Okta developer blog — both have high-quality explainers
  on specific concepts (search by topic).
