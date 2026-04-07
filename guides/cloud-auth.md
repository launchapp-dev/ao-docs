# Cloud Authentication — OAuth 2.1 and CLI PKCE Flow

Animus Cloud is an OAuth 2.1 authorization server. The CLI authenticates using PKCE (Proof Key for Code Exchange), and the resulting access token is used for all subsequent API calls. This guide explains the full flow, the available endpoints, and how tokens are managed.

---

## How CLI Login Works

When you run `animus cloud login`, the CLI performs a standard OAuth 2.1 authorization code flow with PKCE:

```
animus cloud login
    │
    ├── 1. Start a local HTTP server on port 19823
    ├── 2. Generate PKCE code_verifier + code_challenge (S256)
    ├── 3. Open your browser at the authorize endpoint
    │         ↓
    │   animus.launchapp.dev/api/auth/oauth2/authorize
    │   ?client_id=animus-cli
    │   &response_type=code
    │   &redirect_uri=http://localhost:19823/callback
    │   &scope=openid+profile+email+offline_access
    │   &code_challenge=<SHA256(verifier)>
    │   &code_challenge_method=S256
    │   &state=<random>
    │         ↓
    │   [If not signed in → /login page]
    │   [Sign in via GitHub or email]
    │         ↓
    │   [/consent page — approve scopes]
    │         ↓
    │   Redirect → http://localhost:19823/callback?code=AUTH_CODE&state=STATE
    │
    ├── 4. CLI receives callback, validates state
    ├── 5. Exchange code + code_verifier for tokens
    │         POST /api/auth/oauth2/token
    └── 6. Store access_token in ~/.ao/cloud/credentials.json
```

The browser closes automatically after approval. The CLI prints a confirmation and is ready to use.

If you are on a headless machine, use `--no-browser` to print the URL instead:

```bash
animus cloud login --no-browser
# Opening URL is skipped — paste into any browser:
# https://animus.launchapp.dev/api/auth/oauth2/authorize?...
```

---

## OAuth 2.1 Endpoints

The Animus Cloud authorization server exposes the following OAuth endpoints, all under `https://animus.launchapp.dev/api/auth/oauth2/`.

### Authorization Endpoint

```
GET /api/auth/oauth2/authorize
```

Starts the authorization flow. Required parameters:

| Parameter | Value |
|---|---|
| `client_id` | `animus-cli` |
| `response_type` | `code` |
| `redirect_uri` | `http://localhost:19823/callback` |
| `scope` | Space-separated list of scopes (see below) |
| `code_challenge` | base64url(SHA256(code_verifier)) — no padding |
| `code_challenge_method` | `S256` |
| `state` | Random string for CSRF protection |

If the user is not signed in, the server redirects to `/login`. After authentication, the server redirects to `/consent` where the user approves the requested scopes. Once approved, the server redirects back to `redirect_uri` with `code` and `state` query parameters.

### Token Endpoint

```
POST /api/auth/oauth2/token
Content-Type: application/x-www-form-urlencoded
```

**Authorization code exchange** (first login):

```
grant_type=authorization_code
&code=AUTH_CODE
&code_verifier=CODE_VERIFIER
&client_id=animus-cli
&redirect_uri=http://localhost:19823/callback
```

**Response:**

```json
{
  "access_token": "eyJ...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "eyJ...",
  "scope": "openid profile email offline_access"
}
```

Access tokens expire after **1 hour**. Refresh tokens expire after **30 days**.

### Token Refresh

Use the same `/token` endpoint with `refresh_token` grant:

```
grant_type=refresh_token
&refresh_token=REFRESH_TOKEN
&client_id=animus-cli
```

The response includes a new `access_token` (and optionally a new `refresh_token`).

### UserInfo Endpoint

```
GET /api/auth/oauth2/userinfo
Authorization: Bearer {access_token}
```

Returns the authenticated user's identity claims:

```json
{
  "sub": "user_id",
  "email": "you@example.com",
  "email_verified": true,
  "name": "Your Name",
  "image": "https://..."
}
```

---

## Scopes

| Scope | What it grants |
|---|---|
| `openid` | Verify identity — confirms who you are using your Animus account |
| `profile` | Public profile — your display name and avatar |
| `email` | Email address — access your account email |
| `offline_access` | Offline access — enables refresh tokens so sessions persist across restarts |

The CLI requests all four scopes by default. The consent page in the browser shows each scope with its description before you approve.

---

## Token Storage

After a successful login, credentials are stored at:

```
~/.ao/cloud/credentials.json
```

File permissions are set to `0600` (owner read/write only). The file contains:

```json
{
  "server": "https://animus.launchapp.dev",
  "token": "eyJ..."
}
```

Every CLI command that communicates with the cloud reads this file and sends `Authorization: Bearer {token}` with each request. If the access token has expired, re-run `animus cloud login` to obtain fresh tokens.

---

## Server-Side Token Verification

The cloud API accepts three forms of credentials (in precedence order):

1. **Session cookie** — browser clients logged in via the dashboard
2. **OAuth access token** — Bearer token issued by `/api/auth/oauth2/token`
3. **Better Auth session token** — direct session tokens issued by the auth server

For CLI calls, the Bearer access token is verified by the server calling the `/api/auth/oauth2/userinfo` endpoint internally. If the token is valid, the user identity is resolved and the request proceeds.

---

## Consent Page

When a CLI client requests authorization, the browser shows a consent page at `/consent`. The page:

- Displays the application name and icon (`Animus CLI`)
- Lists each requested scope with a human-readable description
- Requires an explicit **Approve** click — deny redirects the CLI with an error

If you have previously approved the same scopes for the same client, the consent page may be skipped on subsequent logins (depending on your session state).

---

## Troubleshooting

**"Failed to bind port 19823"** — Another `animus cloud login` process is already running, or the port is in use. Kill the other process and retry.

**Browser did not open** — Use `--no-browser` and paste the URL manually, or check that `BROWSER` / `xdg-open` is configured on your system.

**"state mismatch" error** — The state parameter did not match. This is a CSRF protection check. Retry the login flow from scratch.

**Token expired** — Access tokens expire after 1 hour. Run `animus cloud login` to refresh. Refresh tokens expire after 30 days of inactivity.

---

## Related

- [`animus cloud login` reference](../reference/cli/cloud.md#animus-cloud-login)
- [Cloud Getting Started](../tutorials/cloud-getting-started.md)
- [Cloud Deployment Guide](cloud-deployment.md)
