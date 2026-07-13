# 14 — Authentication

Account authentication is **Nyxite-native and server-owned by default**, with two **co-equal** primary methods: **password + required TOTP**, and **passkeys (WebAuthn)** (sufficient alone). The Nyxite **server issues its own access + refresh tokens**; native auth and the **enterprise** Keycloak-OIDC option are **pluggable identity providers** that both resolve to **one internal server bearer token**, so the client is IdP-agnostic. Whichever method is used, login yields an API token but **no content key** — decryption is governed entirely by the on-device identity keys ([07](07-key-and-device-management.md)). The two layers are deliberately separate (see [SPECIFICATION §10](https://github.com/Nyxite/Nyxite), [OPEN-DECISIONS #9](https://github.com/Nyxite/Nyxite), [server 08](https://github.com/Nyxite/NyxiteServer)).

## 14.1 Login flows

**Default — native credential / passkey, against the Nyxite server.** No browser redirect is required; the client submits credentials directly to the server over TLS and stores the returned server token.

- **Password + TOTP**: the app collects the password and the TOTP code and posts them to the server's native auth endpoint over TLS. The server verifies the password verifier — **Argon2id over an HMAC-SHA256(password, pepper) pre-hash**, where the **pepper is a server-side secret the client never sees** ([OPEN-DECISIONS "password pepper"](https://github.com/Nyxite/Nyxite), server [08-authentication](https://github.com/Nyxite/NyxiteServer)) — and the TOTP itself (the app does not implement TOTP), then returns **access + refresh tokens**. **Client-side this changes nothing**: the app still sends the plain password over TLS as today; it does **not** hold or apply the pepper. This account-auth verifier is independent of content-key derivation — the login password still **never** feeds any content key ([06](06-cryptography.md), #9). The pepper is symmetric and has **no PQC dimension**; the **recovery-phrase Argon2id path is not peppered**.
- **Passkeys (WebAuthn)**: the app drives a WebAuthn registration (enroll) or assertion (login) ceremony against the server's relying-party challenge using the platform/roaming authenticator, and posts the assertion to the server, which returns the same **access + refresh tokens**. A passkey is phishing-resistant and sufficient on its own (no separate TOTP step).
- These endpoints and the instance host are runtime-configurable per account for self-hosted instances.

**Enterprise option — Keycloak OIDC (Authorization Code + PKCE).** When an instance is configured for enterprise SSO, the app instead runs the OIDC flow, which resolves to the **same internal server token**. SSO/OIDC is a **licensed enterprise feature** (L-3): a **community-mode** server does not advertise it, so the app shows native login only, and a lapsed license stops new OIDC logins (server [16 §16.5](https://github.com/Nyxite/NyxiteServer)). The client is unchanged — it renders whatever methods the server advertises:

- **Authorization Code + PKCE** via **`IdentityModel.OidcClient`** against the Keycloak realm. Config: `Authority` (issuer URL), `ClientId`, redirect URI, scopes (`openid profile email` + any Nyxite scope).
- Use the **system browser** (not an embedded WebView) for the authorization request, for security and SSO. The redirect comes back via a **loopback `http://127.0.0.1:<port>`** listener (desktop-standard for native apps) and/or the registered **`nyxite://` scheme**. In this path Keycloak handles credentials, registration, password reset, and **TOTP enrollment/challenge**.
- On callback, exchange the code for tokens and read identity claims (`sub`, name/email, `roles` = `user`/`admin`); the result is exchanged for the server's own bearer token so the rest of the app stays IdP-agnostic.

## 14.2 TOTP / 2FA

- For **native** login, TOTP is a **required second factor verified by the Nyxite server**; passkey login needs no separate TOTP. For the **enterprise Keycloak** path, TOTP is enforced inside the OIDC flow. Either way the app does not implement TOTP itself.
- If a protected API call returns `403 2fa_required`, route the user back through a step-up auth ([05 §5.4](05-api-client.md)).

## 14.3 Token storage & refresh

- Store the server's **access + refresh tokens** in the **OS keystore** (DPAPI / Secret Service) — **never** in the DB or logs ([04 §4.6](04-local-data-model.md), [17](17-security.md)).
- **Token lifetimes** (match server): the **access token is ~5 min**, so refresh is routine; `AuthHandler` attaches the bearer; on `401 token_expired`, perform a single silent refresh and retry; on refresh failure, surface re-login. (Guest share-session and relay-ticket lifetimes are in [§14.5](#145-share-token-sessions-guests).)
- Logout clears tokens and locks the `UserSession` (zeroizes the in-memory identity key) but **does not** delete enrolled keys/local data unless the user chooses "forget this device/account" ([07](07-key-and-device-management.md)).

## 14.4 Relationship to device enrollment

After a successful login, the app checks device enrollment + identity-key availability ([07 §7.3](07-key-and-device-management.md)):
- New account → generate identity keypair, enroll device, set recovery key.
- Existing account, new device → device-to-device approval or recovery-key unwrap.
- Enrolled device → unlock the identity key (app lock / OS credential) into the `UserSession`.

Until the identity key is present, the app can show structure (encrypted names will be unreadable) but cannot decrypt content; it presents an enrollment screen.

**Account recovery ≠ content recovery.** The server's forgot-password / email-reset flow restores **login only** (it re-establishes the API token); it never restores content. Content recovery is unchanged and stays on the key path: the user's **recovery phrase** unwraps the recovery blob, or an **enrolled device** approves the new one ([07](07-key-and-device-management.md)). An account/email compromise therefore yields no content.

## 14.5 Share-token sessions (guests)

For link shares, the app obtains a **short-lived, relay-scoped share session token** from `GET /share/{token}` (no account login) to authorize relay/ciphertext access; the decryption key is the URL fragment ([13 §13.3](13-sharing.md), [09 §9.8](09-realtime-collaboration.md)). **Lifetimes** (match server): the guest **share-session token is 15 min, renewable**; the **relay socket ticket is single-use, 60 s**. Guest sessions are bounded by token lifetime + share validity and run without the full key/device subsystem.

**Guest storage model**: a guest session runs **without the per-account SQLCipher DB**. Guest content is held in an **ephemeral in-memory cache only** and is **never written to any account database or on-disk store**; decrypted views are transient and discarded when the session ends ([09 §9.8](09-realtime-collaboration.md)).

## 14.6 App lock

Independently of the auth session, the app supports an **app lock** (OS credential / Windows Hello where available, or an app passphrase) that gates unwrapping the DB master key and identity key on launch/after timeout ([17 §17.3](17-security.md), [07 §7.2](07-key-and-device-management.md)). This protects local data even while the server session is still valid — important on a shared desktop where mere OS login would otherwise unseal DPAPI/Secret Service.

## 14.7 Multi-account (v1.0.0)

The app supports **multiple accounts and instance switching from v1.0.0**, matching the Android client. Each account is a fully isolated tenant on the device:

- **Identity & DI scoping**: an **account scope** (child `IServiceScope`) keyed by `accountId` owns that account's `UserSession`, server tokens, identity-key handle, repositories, and clients. Switching accounts disposes the previous scope (zeroizing its in-memory keys) and activates the target's ([01 §1.8](01-architecture.md)).
- **Storage isolation**: each account gets its **own SQLCipher database** (`nyxite-{accountId}.db`) with its **own DB master key**, its own `BlobStore` subdirectory, its own FTS index, and its own OS-keystore entries for tokens/keys. No content, names, search index, or keys are ever shared across accounts on the device ([04 §4.1](04-local-data-model.md), [16 §16.1](16-offline-and-storage-policies.md), [17 §17.2](17-security.md)).
- **Per-account auth**: each account may target a **different instance host** (API base + share base, plus an OIDC authority for enterprise-SSO instances) and its own login method (native or enterprise Keycloak), so a user can hold, say, a personal self-hosted instance and a shared one simultaneously ([§14.1](#141-login-flows)).
- **Active account**: a single "active account" drives the UI at a time; an **account switcher** lets the user add, switch, and remove accounts ([15 §15.1](15-ui-and-navigation.md)). Background sync is **tiered** ([19 §19.9](19-open-questions.md)): the active account runs **full-corpus prefetch + relay**; recently-used accounts run **structure/delta only**, deferring heavy prefetch until activated — all subject to the lock model.
- **Guest sessions** (link shares, [§14.5](#145-share-token-sessions-guests)) are independent of accounts and may be opened without any account signed in.

**Lock interaction**: app lock gates *all* accounts; per-account identity-key unwrap still happens on first use after unlock. Removing an account deletes its DB, cache, index, tokens, and OS-keystore-wrapped keys ("forget account"), independent of other accounts.
