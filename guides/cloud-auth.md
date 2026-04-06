# Authenticating with Animus Cloud

Animus Cloud handles authentication in two directions:

1. **CLI authentication** — `ao cloud login` lets the Animus CLI act on your behalf.
2. **OAuth 2.1 provider** — Animus Cloud acts as an authorization server so your own applications can use Animus as an identity provider (SSO).

---

## CLI Login Flow

`ao cloud login` uses a browser-based OAuth handshake. No API token setup required.

```bash
ao cloud login
```

### What happens

1. The CLI starts a local HTTP server on port `19823`.
2. The CLI opens your system browser to `https://animus.launchapp.dev/api/cli/auth/start`.
3. Animus redirects to GitHub OAuth. Sign in with your GitHub account.
4. After GitHub authentication, the cloud server redirects back to `localhost:19823` with a session token.
5. The CLI stores the token in `~/.ao/cloud.json`.
6. All subsequent `ao cloud` commands use `Authorization: Bearer <token>` automatically.

```
CLI                      Browser                  Animus Cloud             GitHub
 │                          │                          │                     │
 │── start local :19823 ───►│                          │                     │
 │── open browser ─────────►│                          │                     │
 │                          │── GET /auth/start ──────►│                     │
 │                          │                          │── POST /sign-in ───►│
 │                          │                          │◄── OAuth callback ──│
 │                          │◄── redirect /auth/complete ──────────────────  │
 │◄── localhost:19823/callback?token=TOKEN ────────────│                     │
 │── store token ───────────│                          │                     │
```

### Flags

| Flag | Description |
|---|---|
| `--no-browser` | Print the login URL instead of opening the browser. Use on headless or remote machines. |
| `--server <URL>` | Connect to a self-hosted Animus Cloud instance. Default: `https://animus.launchapp.dev`. |

### Headless / remote machines

On a machine without a browser (CI runners, remote SSH), use `--no-browser`:

```bash
ao cloud login --no-browser
```

The CLI prints the login URL. Open it in any browser, complete GitHub login, and the CLI receives the token via port `19823`. Ensure port `19823` is not blocked by a firewall on the machine running the CLI.

### Checking who you're logged in as

After login, verify the authenticated identity:

```bash
ao cloud status
```

### Token storage

Credentials are stored in `~/.ao/cloud.json`. To log out, delete this file:

```bash
rm ~/.ao/cloud.json
```

Re-run `ao cloud login` to re-authenticate with a fresh session.

---

## OAuth 2.1 Provider

Animus Cloud is also an **OAuth 2.1 authorization server**. Your own applications can use Animus as an identity provider — users authorize access with their Animus account, and your app receives an access token.

This is the standard OAuth 2.1 authorization code flow with PKCE (Proof Key for Code Exchange). PKCE is required for all clients — there are no implicit grants.

### Supported scopes

| Scope | What it grants |
|---|---|
| `openid` | Verify the user's identity |
| `profile` | Display name and avatar |
| `email` | Account email address |
| `offline_access` | Refresh tokens for long-lived sessions |

### Authorization endpoints

| Endpoint | Purpose |
|---|---|
| `GET /api/auth/oauth2/authorize` | Start the authorization flow |
| `POST /api/auth/oauth2/token` | Exchange code for tokens |
| `GET /api/auth/oauth2/userinfo` | Fetch user profile (requires `openid` scope) |
| `GET /api/auth/.well-known/openid-configuration` | OIDC discovery document |

### Authorization code + PKCE flow

**Step 1 — Generate a code verifier and challenge**

```js
// Node.js example
import crypto from "crypto";

const codeVerifier = crypto.randomBytes(32).toString("base64url");
const codeChallenge = crypto
  .createHash("sha256")
  .update(codeVerifier)
  .digest("base64url");
```

**Step 2 — Redirect the user to the authorization endpoint**

```
GET https://animus.launchapp.dev/api/auth/oauth2/authorize
  ?response_type=code
  &client_id=<YOUR_CLIENT_ID>
  &redirect_uri=<YOUR_REDIRECT_URI>
  &scope=openid profile email offline_access
  &state=<RANDOM_STATE>
  &code_challenge=<CODE_CHALLENGE>
  &code_challenge_method=S256
```

The user sees the Animus **consent page** listing the permissions your app is requesting. They can approve or deny.

**Step 3 — Receive the authorization code**

After the user approves, Animus redirects to your `redirect_uri`:

```
https://your-app.example.com/callback?code=<AUTH_CODE>&state=<STATE>
```

Verify `state` matches what you sent (CSRF protection).

**Step 4 — Exchange code for tokens**

```bash
POST https://animus.launchapp.dev/api/auth/oauth2/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=<AUTH_CODE>
&redirect_uri=<YOUR_REDIRECT_URI>
&client_id=<YOUR_CLIENT_ID>
&code_verifier=<CODE_VERIFIER>
```

Response:

```json
{
  "access_token": "eyJ...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "...",
  "scope": "openid profile email offline_access",
  "id_token": "eyJ..."
}
```

**Step 5 — Fetch user info**

```bash
GET https://animus.launchapp.dev/api/auth/oauth2/userinfo
Authorization: Bearer <ACCESS_TOKEN>
```

Response:

```json
{
  "sub": "user_01HXYZ",
  "name": "Sam Smith",
  "email": "sam@example.com",
  "picture": "https://avatars.githubusercontent.com/..."
}
```

### Refreshing tokens

Use the refresh token to get a new access token without re-prompting the user:

```bash
POST https://animus.launchapp.dev/api/auth/oauth2/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token=<REFRESH_TOKEN>
&client_id=<YOUR_CLIENT_ID>
```

Access tokens expire after **1 hour**. Refresh tokens expire after **30 days**.

### Registering an OAuth client

OAuth clients are managed through the Animus Cloud dashboard. You need:

- **Client ID** — public identifier for your app
- **Redirect URI(s)** — allowed callback URLs
- **Scopes** — the scopes your app will request

Public clients (browser apps, CLI tools) use PKCE and have no client secret.

---

## Related

- [Cloud Deployment Guide](cloud-deployment.md) — deploy and manage cloud daemons
- [ao cloud — CLI Reference](../reference/cli/cloud.md) — full command and flag reference
- [Privacy & Data Policy](privacy.md) — what Animus Cloud stores
