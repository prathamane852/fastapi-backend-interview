# 07 — Authentication & Authorization

---

## 📖 Vocabulary

> *Definition* is the precise meaning; *Say it in an interview* is the quotable one-liner.

| Term | Definition | Say it in an interview |
|---|---|---|
| **Authentication (AuthN)** | Verifying that a principal is who it claims to be, by validating credentials (password, token, certificate). Failure → 401. | "Authentication answers 'who are you' and fails with 401." |
| **Authorization (AuthZ)** | Determining whether an authenticated principal is permitted to perform an action on a resource. Failure → 403. | "Authorization answers 'may you do this' and fails with 403." |
| **Credential** | Evidence presented to prove identity: password, bearer token, API key, client certificate. | "A credential is what you present to prove identity." |
| **Cryptographic hash** | A one-way function mapping input to a fixed-size digest; infeasible to invert or find collisions. | "A hash is one-way; you verify by re-hashing, never by decrypting." |
| **Salt** | Random per-password data combined with the password before hashing, so identical passwords yield different hashes and precomputed tables are useless. | "Salting defeats rainbow tables." |
| **Argon2 / bcrypt** | Memory- and CPU-hard password hashing functions designed to be slow and tunable, resisting brute force. Argon2id is the current recommendation. | "Passwords get Argon2 or bcrypt, never SHA-256." |
| **Token** | A credential issued after authentication that a client presents on subsequent requests. | "A token proves you authenticated earlier." |
| **Opaque token** | A random string with no intrinsic meaning; the server looks it up in a store to find the session. | "Opaque tokens are revocable instantly but need a lookup." |
| **JWT** | *JSON Web Token* (RFC 7519): a base64url-encoded `header.payload.signature`, where the signature authenticates the claims. Readable by anyone; tamper-evident. | "A JWT is signed, not encrypted: anyone can read it, nobody can alter it." |
| **Claim** | A name/value assertion in a JWT payload. Registered claims: `iss`, `sub`, `aud`, `exp`, `nbf`, `iat`, `jti`. | "Claims are the assertions in the token; `sub`, `exp` and `aud` are the ones I always check." |
| **Signature (HS256 / RS256 / ES256)** | HS256: HMAC with a shared secret. RS256: RSA signature with a private key, verified with the public key. ES256: ECDSA equivalent. | "HS256 shares a secret; RS256 lets verifiers use only a public key." |
| **Bearer token** | A token whose possession alone grants access, sent as `Authorization: Bearer <token>`. | "Whoever holds a bearer token is trusted, so protect it in transit and storage." |
| **Access token** | A short-lived token presented on each API call and verified statelessly. | "Access tokens are short-lived to limit damage if stolen." |
| **Refresh token** | A long-lived token, stored server-side (hashed), exchanged only at the token endpoint for a new access token; revocable. | "Refresh tokens restore revocability that stateless JWTs lack." |
| **Token rotation** | Issuing a new refresh token on every use and invalidating the old; reuse of a revoked token indicates theft. | "Rotate refresh tokens and treat reuse as compromise." |
| **Revocation** | Invalidating a token before its natural expiry, via server-side state (denylist, token version, refresh-token record). | "You can't revoke a stateless JWT without adding state, so keep it short-lived." |
| **OAuth 2.0** | An authorization framework (RFC 6749) enabling a client to obtain limited access to a resource on behalf of a resource owner, via defined flows (grant types). | "OAuth2 is about delegated authorization; OIDC adds identity on top." |
| **Authorization Code flow + PKCE** | The recommended OAuth2 flow: the user authenticates at the IdP, the client receives a code and exchanges it for tokens; PKCE binds the code to the client to prevent interception. | "Auth Code with PKCE is the flow for SPAs and mobile apps." |
| **Password flow (ROPC)** | OAuth2 grant where the client sends the user's username/password directly to the token endpoint. Deprecated in OAuth 2.1; acceptable only for first-party clients. | "The password flow is simple but deprecated; use it only for your own frontend." |
| **OpenID Connect (OIDC)** | An identity layer on OAuth2 adding a signed **ID token**, a `userinfo` endpoint and discovery metadata. | "OIDC tells you *who* the user is; OAuth2 alone only grants access." |
| **IdP (Identity Provider)** | A service that authenticates users and issues tokens: Cognito, Auth0, Okta, Keycloak, Google. | "The IdP owns login; my API just verifies its tokens." |
| **JWKS** | *JSON Web Key Set*: a JSON document of public keys published by an issuer at a well-known URL for verifying its signatures. | "I fetch the IdP's JWKS to verify tokens locally." |
| **`kid`** | *Key ID* header parameter identifying which JWKS key signed the token, enabling key rotation. | "`kid` tells me which public key to use during rotation." |
| **Discovery document** | `/.well-known/openid-configuration`: JSON listing an IdP's endpoints, supported algorithms and `jwks_uri`. | "The discovery document is where I find the JWKS URL and endpoints." |
| **XSS** | *Cross-Site Scripting*: injecting attacker-controlled script into a page, which can then read anything JavaScript can (e.g., `localStorage`). | "XSS steals anything JavaScript can read, so tokens in localStorage are exposed." |
| **CSRF** | *Cross-Site Request Forgery*: a malicious site causes the victim's browser to send an authenticated request (cookies auto-attached) to your API. | "CSRF abuses automatic cookies; SameSite and CSRF tokens defend against it." |
| **HttpOnly / Secure / SameSite** | Cookie attributes: `HttpOnly` hides it from JavaScript; `Secure` restricts to HTTPS; `SameSite=Strict/Lax` blocks cross-site sending. | "HttpOnly + Secure + SameSite is the safe cookie recipe for tokens." |
| **RBAC** | *Role-Based Access Control*: users hold roles; roles hold permissions. | "RBAC is coarse-grained: roles bundle permissions." |
| **Permission-based (ABAC/ReBAC)** | Fine-grained authorization on explicit permissions or attributes/relationships (`orders:write`, "owner of resource"). | "Permissions or relationships handle 'edit your own post', which roles can't." |
| **IDOR** | *Insecure Direct Object Reference*: accessing another principal's resource by manipulating an identifier because ownership isn't checked. | "IDOR is prevented by checking ownership on every resource access." |
| **mTLS** | Mutual TLS: both client and server present and verify certificates, authenticating both ends of the connection. | "mTLS authenticates services to each other at the transport layer." |
| **MFA / step-up authentication** | Requiring a second factor; *step-up* demands fresh MFA before sensitive actions even within an authenticated session. | "Step-up auth requires recent MFA for destructive admin actions." |
| **Privilege escalation** | Obtaining permissions beyond those legitimately granted (horizontal: another user's data; vertical: admin rights). | "Privilege escalation is prevented by server-side checks and never trusting client-supplied roles." |
| **Least privilege** | Granting each principal only the permissions strictly necessary for its function. | "Every token, role and service gets the minimum access it needs." |
| **Timing attack / user enumeration** | Inferring secrets or account existence from response-time differences or differing error messages. | "Use constant-time compares and identical error messages for bad user and bad password." |
| **Audience (`aud`) / Issuer (`iss`)** | `aud`: the intended recipient(s) of a token. `iss`: the entity that issued it. Both must be validated to prevent token misuse across services. | "Always check `aud` and `iss` so another service's token isn't accepted here." |

---

## Q1. How do you implement authentication in FastAPI?

### 🧒 Simple explanation
1. User sends email + password to a login endpoint.
2. Server checks the password against a stored *hash* and returns a signed token (JWT).
3. Client sends that token on every request in the `Authorization` header.
4. A dependency verifies the token and loads the user.

### 💻 Code — hashing and token creation
```python
# app/core/security.py
from datetime import datetime, timedelta, timezone
from jose import jwt
from passlib.context import CryptContext

pwd = CryptContext(schemes=["argon2"], deprecated="auto")

def hash_password(p: str) -> str:
    return pwd.hash(p)

def verify_password(p: str, hashed: str) -> bool:
    return pwd.verify(p, hashed)

def create_access_token(sub: str, roles: list[str], minutes: int = 15) -> str:
    now = datetime.now(timezone.utc)
    claims = {
        "sub": sub,
        "roles": roles,
        "iat": now,
        "exp": now + timedelta(minutes=minutes),
        "type": "access",
    }
    return jwt.encode(claims, settings.jwt_secret.get_secret_value(), algorithm="HS256")
```
### 🔍 Line by line
- `CryptContext(schemes=["argon2"], deprecated="auto")` — `passlib` helper configured to hash with Argon2 (slow, salted). `deprecated="auto"` lets you migrate old hashes later.
- `pwd.hash(p)` — produces a string containing algorithm, salt and hash. Store this, never the plaintext.
- `pwd.verify(p, hashed)` — re-hashes with the same salt and compares in constant time.
- `datetime.now(timezone.utc)` — always UTC.
- `claims` — `sub`: user ID as string; `roles`: custom claim; `iat`/`exp`: issued/expiry; `type`: distinguishes access from refresh tokens.
- `jwt.encode(claims, secret, algorithm="HS256")` — sign with a shared secret. `get_secret_value()` unwraps `SecretStr`.

### 💻 Code — login endpoint
```python
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy import select

@router.post("/token")
async def login(form: Annotated[OAuth2PasswordRequestForm, Depends()], db: DB):
    user = await db.scalar(select(User).where(User.email == form.username))
    if not user or not verify_password(form.password, user.hashed_password):
        raise HTTPException(401, "Incorrect email or password")
    return {"access_token": create_access_token(str(user.id), user.roles), "token_type": "bearer"}
```
- `OAuth2PasswordRequestForm` — parses a form body with `username` and `password` fields (the OAuth2 spec requires form encoding here, not JSON).
- `db.scalar(select(User).where(...))` — run a query and return the first column of the first row (the `User`).
- Same error message for "no such user" and "wrong password" — don't reveal which.
- Return shape `{"access_token", "token_type"}` is what the OAuth2 password flow expects and what Swagger's Authorize button uses.

The `get_current_user` dependency from file 04 completes the loop.

---

## Q2. What is JWT?

A **JSON Web Token**: three base64url parts separated by dots.
- **Header** — `{"alg": "HS256", "typ": "JWT"}`.
- **Payload** — the claims: `{"sub": "42", "exp": 1700000000, ...}`.
- **Signature** — HMAC/RSA over `header.payload`.

⚠️ The payload is **encoded, not encrypted** — anyone can decode and read it. The signature only guarantees it wasn't *altered*. Never put secrets in a JWT.

---

## Q3. How does JWT authentication work?

1. Client logs in → server verifies credentials → issues a signed JWT with an expiry.
2. Client stores it and sends `Authorization: Bearer <jwt>` on each request.
3. Server verifies the signature and `exp` **locally** — no database or session lookup — and trusts the claims.
4. When it expires, the client uses a refresh token to get a new one.

**First principles:** classic sessions need server-side storage shared by all app instances. JWTs move that state to the client and make it tamper-proof with cryptography — that's why they scale horizontally with no session store, and also why revoking one is hard (there's nothing on the server to delete).

---

## Q4. What is the difference between authentication and authorization?

- **Authentication** — *Who are you?* → **401 Unauthorized** when it fails ("I don't know you").
- **Authorization** — *Are you allowed?* → **403 Forbidden** when it fails ("I know you, and no").
Always authenticate first, then authorize.

---

## Q5. Where should JWT tokens be stored?

For browsers the trade-off is **XSS vs CSRF**:

| Storage | XSS risk | CSRF risk | Verdict |
|---|---|---|---|
| `localStorage` | ❌ any injected script can read it | ✅ not auto-sent | Common but risky |
| JS memory variable | lower (lost on refresh) | ✅ | Good for access tokens |
| `HttpOnly; Secure; SameSite=Strict` cookie | ✅ JS can't read | ❌ auto-sent → need SameSite/CSRF token | **Recommended** |

**Best practice:** short-lived access token in memory (or HttpOnly cookie) + refresh token in an `HttpOnly; Secure; SameSite=Strict` cookie scoped to the refresh path. Mobile: OS keychain/keystore. Server-to-server: secrets manager, never in code.

---

## Q6. What are access tokens and refresh tokens?

- **Access token** — short-lived (5–15 min), sent on every request, verified statelessly. If stolen, damage window is small.
- **Refresh token** — long-lived (days/weeks), sent **only** to the refresh endpoint. Stored server-side (hashed), so it *can* be revoked. Rotated on each use.

```python
from fastapi import Cookie
from fastapi.responses import JSONResponse

@router.post("/refresh")
async def refresh(refresh_token: Annotated[str, Cookie()], db: DB):
    payload = decode(refresh_token, expected_type="refresh")
    record = await db.get(RefreshToken, payload["jti"])
    if record is None or record.revoked or record.user_id != int(payload["sub"]):
        await revoke_all_for_user(db, payload["sub"])
        raise HTTPException(401, "Invalid refresh token")
    record.revoked = True
    new_refresh = await issue_refresh(db, record.user_id)
    resp = JSONResponse({"access_token": create_access_token(...), "token_type": "bearer"})
    resp.set_cookie("refresh_token", new_refresh, httponly=True, secure=True,
                    samesite="strict", path="/auth/refresh")
    return resp
```
### 🔍 Line by line
- `refresh_token: Annotated[str, Cookie()]` — read from a cookie.
- `decode(..., expected_type="refresh")` — verify signature and that `type == "refresh"`.
- `payload["jti"]` — token ID linking to a DB row.
- Reuse of a **revoked** token = evidence of theft → `revoke_all_for_user` (kill the whole family).
- `record.revoked = True` — single use.
- `issue_refresh` — mint a replacement (**rotation**).
- `set_cookie(..., httponly=True, secure=True, samesite="strict", path="/auth/refresh")` — JS can't read it, HTTPS only, not sent cross-site, only sent to the refresh path.

---

## Q7. How do you implement OAuth2 with FastAPI?

Two roles:

**As your own auth server (password flow, first-party apps)** — Q1 above with `OAuth2PasswordBearer` + `OAuth2PasswordRequestForm`.

**As a client of an external provider (Google/GitHub/Cognito) — Authorization Code flow** with `authlib`:
```python
from authlib.integrations.starlette_client import OAuth

oauth = OAuth()
oauth.register(
    "google",
    client_id=..., client_secret=...,
    server_metadata_url="https://accounts.google.com/.well-known/openid-configuration",
    client_kwargs={"scope": "openid email profile"},
)

@router.get("/login/google")
async def login_google(request: Request):
    return await oauth.google.authorize_redirect(request, redirect_uri=settings.callback_url)

@router.get("/callback/google")
async def callback(request: Request):
    token = await oauth.google.authorize_access_token(request)
    userinfo = token["userinfo"]
    user = await upsert_user(userinfo["email"])
    return {"access_token": create_access_token(str(user.id), user.roles)}
```
- `server_metadata_url` — OIDC discovery document; authlib reads endpoints and keys from it.
- `authorize_redirect` — sends the browser to Google with `state` (CSRF protection) and PKCE.
- `authorize_access_token` — exchanges the returned `code` for tokens (server-to-server) and validates `state`.
- `token["userinfo"]` — ID-token claims.
- Issue **your own** token afterwards; don't reuse Google's as your API credential.

---

## Q8. What is OAuth2 Password Flow?

The user types username/password *directly into the client app*, which POSTs them to the token endpoint and receives an access token. Simplest flow; what `OAuth2PasswordBearer` implements. The OAuth 2.1 draft **deprecates** it because it trains users to give passwords to apps. Fine for first-party (your frontend ↔ your API); use Authorization Code + PKCE for anything else.

## Q9. What is OpenID Connect?

An **identity layer on top of OAuth2**. OAuth2 alone says "this app may access these resources"; OIDC adds "and this is *who the user is*" via a signed **ID token** (a JWT with `sub`, `email`, `name`…), a `/userinfo` endpoint, and a **discovery document** at `/.well-known/openid-configuration` that publishes endpoints and the **JWKS**.

---

## Q10. How do you implement role-based access control?

See file 04 Q14 (`require_roles`). Design notes:
- Store roles on the user (or a `user_roles` table).
- Put roles in the JWT to avoid DB hits — but then role changes take effect only at token expiry (fine with 15-minute tokens; force re-login for demotions).
- Router-level for coarse control, route-level for exceptions.
- RBAC can't express "editors may edit *their own* posts" → that's resource-level authorization (next).

## Q11. How do you implement permission-based authorization?

Permissions are finer than roles: `orders:read`, `orders:write`, `users:delete`. Roles become *bundles* of permissions.

```python
ROLE_PERMS = {
    "viewer": {"orders:read"},
    "editor": {"orders:read", "orders:write"},
    "admin":  {"orders:read", "orders:write", "users:delete"},
}

def require_perm(perm: str):
    async def _check(user: CurrentUser) -> User:
        perms = set().union(*(ROLE_PERMS.get(r, set()) for r in user.roles))
        if perm not in perms:
            raise HTTPException(403, f"Missing permission: {perm}")
        return user
    return _check

@router.delete("/users/{id}", dependencies=[Depends(require_perm("users:delete"))])
async def delete_user(id: int): ...
```
- `set().union(*(...))` — union of the permission sets of every role the user has.

**Resource-level** ("owner or admin") — check inside the service with the loaded entity:
```python
async def get_order_for_user(order_id: int, user: CurrentUser, db: DB) -> Order:
    order = await db.get(Order, order_id)
    if order is None:
        raise NotFound("order")
    if order.owner_id != user.id and "admin" not in user.roles:
        raise HTTPException(404)   # 404, not 403: don't reveal that it exists
    return order
```
For complex policies use a policy engine (OPA, Cerbos, Casbin) so rules live in one auditable place.

---

## Q12. How do you expire JWT tokens?

Set the `exp` claim at creation; `jwt.decode` rejects expired tokens automatically (`ExpiredSignatureError`). Keep access tokens short (5–15 min). Optionally check `nbf` (not before) and allow small `leeway` (~30 s) for clock skew.

## Q13. How do you revoke a JWT?

Stateless JWTs can't be revoked without adding state. Options, lightest first:
1. **Short expiry + refresh-token revocation** (default) — revoke the refresh token in the DB; the access token dies within minutes.
2. **Denylist by `jti`** in Redis with TTL = remaining lifetime: on logout, `SET denylist:<jti> 1 EX <seconds_left>`; `get_current_user` checks it (one Redis GET).
3. **Token version** on the user record and in the claim; bump it to invalidate all that user's tokens (needs a DB/cache read per request).
4. **Opaque tokens** (random string → server lookup) — instant revocation, but you've reinvented sessions.

---

## Q14. What are the security risks of JWT?

- **`alg: none` / algorithm confusion** — always pass `algorithms=[...]` explicitly; never trust the header's `alg`.
- **Weak HMAC secrets** — brute-forceable; use ≥ 256-bit random secrets or asymmetric keys.
- **No revocation** — stolen token is valid until expiry.
- **Secrets in payload** — it's only base64.
- **Long expiry** — larger blast radius.
- **XSS theft** from `localStorage`.
- **Missing `aud`/`iss` checks** — a token from another service accepted here.
- **No key rotation** — use `kid` + JWKS.
- **Shared secrets across services** → cross-service replay; use per-service audiences.

## Q15. What is token rotation?

Issuing a **new** token each time one is used (refresh tokens) or on a schedule (signing keys).
- *Refresh rotation*: each refresh returns a new refresh token and invalidates the old; reuse of an old one signals theft → revoke the chain.
- *Key rotation*: publish several public keys in JWKS with `kid`; sign with the newest; keep old ones for verification until their tokens have expired.

---

## Q16. How would you secure an internal FastAPI service?

Defense in depth — assume the network isn't enough:
1. **Network** — private subnet / cluster-internal Service, `NetworkPolicy` allowing only known callers, no public ingress.
2. **Transport** — mTLS via a service mesh (Istio/Linkerd) so both sides prove identity.
3. **Service identity** — callers present a JWT from an internal IdP (or SPIFFE/SPIRE SVID) with `aud = orders-service`; verify in a dependency.
4. **Least privilege** — per-caller scopes.
5. **No shared static API keys**; if unavoidable, rotate via Secrets Manager.
6. **Rate limits and timeouts** even internally.
7. **Audit logging** of the principal on every request.
8. **Disable docs** or protect them.

---

## Q17. How do you validate tokens issued by an external identity provider?

Fetch the provider's **JWKS** via OIDC discovery, cache it, and verify signature + `iss` + `aud` + `exp`:

```python
import httpx, time
from jose import jwt

class JWKSVerifier:
    def __init__(self, issuer: str, audience: str):
        self.issuer, self.audience = issuer, audience
        self._keys: dict | None = None
        self._fetched = 0.0

    async def keys(self) -> dict:
        if self._keys is None or time.time() - self._fetched > 3600:
            async with httpx.AsyncClient(timeout=5) as c:
                cfg = (await c.get(f"{self.issuer}/.well-known/openid-configuration")).json()
                self._keys = (await c.get(cfg["jwks_uri"])).json()
                self._fetched = time.time()
        return self._keys

    async def verify(self, token: str) -> dict:
        header = jwt.get_unverified_header(token)
        jwks = await self.keys()
        key = next((k for k in jwks["keys"] if k["kid"] == header["kid"]), None)
        if key is None:
            self._keys = None
            jwks = await self.keys()
            key = next((k for k in jwks["keys"] if k["kid"] == header["kid"]), None)
            if key is None:
                raise HTTPException(401, "Unknown signing key")
        try:
            return jwt.decode(token, key, algorithms=["RS256"], audience=self.audience, issuer=self.issuer)
        except jwt.JWTError as e:
            raise HTTPException(401, f"Invalid token: {e}")
```
### 🔍 Line by line
- `keys()` — cache the JWKS for an hour; on first call fetch the discovery doc, read `jwks_uri`, download the keys.
- `jwt.get_unverified_header(token)` — read `kid` **without** verifying (safe: only used to pick a key).
- `next((k for k in ... if k["kid"] == ...), None)` — find the matching key or `None`.
- Unknown `kid` → keys may have rotated → clear cache and refetch **once**.
- `jwt.decode(token, key, algorithms=["RS256"], audience=..., issuer=...)` — full verification: signature, expiry, this API is the intended audience, this IdP is the issuer.

## Q18. How do you integrate FastAPI with AWS Cognito?

Cognito is an OIDC provider, so the verifier above applies with:
- `issuer = f"https://cognito-idp.{region}.amazonaws.com/{user_pool_id}"`, JWKS at `{issuer}/.well-known/jwks.json`.
- Two token types: **ID token** (`aud` = app client ID, has user attributes) and **access token** (`client_id` claim instead of `aud`, has `scope` and `cognito:groups`). For APIs verify the **access token**: check `token_use == "access"` and `client_id` matches.
- Map `cognito:groups` → roles.
- Flow: frontend uses Cognito Hosted UI / Amplify (Authorization Code + PKCE) → gets tokens → calls FastAPI with the access token → dependency verifies via JWKS → user identified by `sub`. Or terminate auth at API Gateway with a Cognito authorizer and forward claims as headers.

---

## Q19. How would you protect admin endpoints?

1. Separate router with `require_roles(Role.admin)` **and** a distinct prefix (`/admin/...`) so the ingress can add IP allow-lists / VPN-only rules.
2. **Step-up auth** (recent MFA) for destructive actions — check an `auth_time`/`amr` claim.
3. Short-lived admin tokens (5 min) with a separate audience.
4. Audit-log every admin call with principal, target, before/after.
5. Rate-limit and alert on 403s to the admin prefix.
6. Hide from public docs (`include_in_schema=False`).
7. Consider a separate deployment for admin APIs — different blast radius.

## Q20. How would you prevent privilege escalation?

- **Never trust client-supplied role/ownership fields** — request schemas must not contain `is_admin`, `role`, `owner_id`; set them server-side.
- **Authorize on the server for every request**, including reads; never rely on the UI hiding buttons.
- **Check ownership at the resource level** (IDOR): user A must not fetch `/orders/123` belonging to B.
- **Derive identity from the verified token**, never from a body/header the client controls (except from a trusted gateway on a private network).
- **Least-privilege tokens** with scopes; internal services don't get admin scope.
- **Role changes invalidate tokens** (token version bump).
- **Deny by default** at the router level; public routes explicit.
- **Impersonation features** need strong audit trails and MFA.
- **Tests**: for every endpoint, a test that a lower-privileged user gets 403/404.
