# Security Headers

The AO cloud dashboard and any self-hosted AO web interfaces ship with a standard set of HTTP security headers in production. This guide explains each header, the default policy, and how to tighten or relax settings for your deployment.

For general cloud deployment see [Cloud Deployment](cloud-deployment.md). For data privacy guarantees see [Privacy & Data Policy](privacy.md).

---

## Overview

Security headers are HTTP response headers that instruct browsers how to handle your site's content. They protect against cross-site scripting (XSS), clickjacking, protocol downgrade attacks, and unintended resource loading. AO applies them at the edge/reverse-proxy layer so they are enforced regardless of which runtime serves the application.

The headers shipped in v36:

| Header | Purpose |
|---|---|
| `Content-Security-Policy` | Allowlist of origins permitted to load scripts, styles, images, and other resources |
| `Strict-Transport-Security` | Instructs browsers to use HTTPS exclusively and optionally join the HSTS preload list |
| `X-Frame-Options` | Prevents the page from being embedded in a `<frame>`, `<iframe>`, or `<object>` |
| `X-Content-Type-Options` | Stops browsers from MIME-sniffing a response away from the declared `Content-Type` |
| `Referrer-Policy` | Controls how much referrer information is included with outbound requests |
| `Permissions-Policy` | Restricts access to browser APIs (camera, microphone, geolocation) |

---

## Content-Security-Policy

### Default policy

```
Content-Security-Policy:
  default-src 'self';
  script-src  'self' 'nonce-{REQUEST_NONCE}';
  style-src   'self' 'unsafe-inline';
  img-src     'self' data: https://avatars.githubusercontent.com;
  font-src    'self';
  connect-src 'self' https://api.ao.dev wss://api.ao.dev;
  frame-src   'none';
  object-src  'none';
  base-uri    'self';
  form-action 'self';
  upgrade-insecure-requests;
```

Each directive operates on the principle of denying everything not explicitly allowed:

| Directive | Allowed sources | Notes |
|---|---|---|
| `default-src` | Same origin | Fallback for unspecified directives |
| `script-src` | Same origin + per-request nonce | Inline scripts must carry the nonce attribute |
| `style-src` | Same origin + inline | Inline styles are allowed for styled-components compatibility |
| `img-src` | Same origin, data URIs, GitHub Avatars CDN | Restricts third-party image loading |
| `connect-src` | Same origin, AO API (HTTPS + WSS) | WebSocket for live agent log streaming |
| `frame-src` | None | No iframes in the dashboard |
| `object-src` | None | Blocks Flash/plugins |
| `upgrade-insecure-requests` | — | Rewrites HTTP sub-resource requests to HTTPS automatically |

### Nonce-based script allowlisting

Every script tag rendered server-side receives a cryptographically random nonce injected at request time:

```html
<script nonce="a3f9bc2e14d8…">
  window.__AO_CONFIG__ = { … };
</script>
```

The `Content-Security-Policy` header carries the matching `nonce-{value}` in `script-src`. Inline scripts without the nonce — including those injected by XSS — are blocked.

### Extending the policy

If you self-host the dashboard behind your own reverse proxy, you can add directives in your Nginx or Caddy config:

```nginx
# nginx — add an analytics endpoint to connect-src
add_header Content-Security-Policy
  "default-src 'self'; connect-src 'self' https://api.ao.dev wss://api.ao.dev https://analytics.example.com; …"
  always;
```

```caddyfile
# Caddyfile
header Content-Security-Policy "default-src 'self'; connect-src 'self' https://api.ao.dev wss://api.ao.dev https://analytics.example.com; …"
```

> **Do not remove `nonce-{REQUEST_NONCE}` from `script-src`** — doing so will break the dashboard's bootstrap script.

### CSP reporting

To capture policy violations without blocking them during a rollout, add a `report-uri` or `report-to` directive pointing at your reporting endpoint:

```
Content-Security-Policy-Report-Only:
  default-src 'self';
  script-src  'self' 'nonce-{REQUEST_NONCE}';
  report-uri  https://csp-reports.example.com/ao;
```

Switch from `Content-Security-Policy-Report-Only` to `Content-Security-Policy` once you have verified no legitimate sources are being blocked.

---

## Strict-Transport-Security (HSTS)

### Default value

```
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

This instructs browsers to:

1. Only connect to this origin over HTTPS for the next 365 days.
2. Apply the same policy to all subdomains.

### HSTS preload

To join the browser-maintained HSTS preload list (the strongest protection — no initial HTTP request is ever made), add the `preload` directive and submit your domain at `hstspreload.org`:

```
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
```

**Requirements before enabling preload:**

- Your TLS certificate must be valid and issued by a publicly trusted CA.
- `includeSubDomains` must be present — all subdomains must also serve HTTPS.
- `max-age` must be at least 31,536,000 seconds (1 year).
- Preload removal takes time — only enable this for domains you intend to keep on HTTPS permanently.

### During initial deployment

If you are migrating an existing site from HTTP to HTTPS, start with a short `max-age` and no `includeSubDomains` while you validate:

```
Strict-Transport-Security: max-age=300
```

Increase `max-age` incrementally (300 → 86400 → 604800 → 31536000) after confirming all resources load over HTTPS.

---

## X-Frame-Options

### Default value

```
X-Frame-Options: DENY
```

The dashboard cannot be embedded in any frame or iframe on any origin. This prevents clickjacking attacks where an attacker overlays an invisible frame over your authenticated session.

### Allowing framing from a specific origin

`X-Frame-Options` does not support origin allowlists. Use CSP `frame-ancestors` instead, which supersedes `X-Frame-Options` in modern browsers while retaining `X-Frame-Options` for legacy browser compatibility:

```
# Allow embedding from your own domain only
Content-Security-Policy: frame-ancestors 'self' https://internal.example.com;
X-Frame-Options: SAMEORIGIN
```

---

## X-Content-Type-Options

### Default value

```
X-Content-Type-Options: nosniff
```

Prevents Internet Explorer and Chrome from MIME-sniffing a response away from the declared `Content-Type`. Without this header, a maliciously named file uploaded through the dashboard (e.g. a `.jpg` containing JavaScript) could be executed as a script by some browsers.

This header has no configuration surface — it is always `nosniff`.

---

## Referrer-Policy

### Default value

```
Referrer-Policy: strict-origin-when-cross-origin
```

| Request type | Referrer sent |
|---|---|
| Same-origin navigation | Full URL (path + query) |
| Cross-origin to HTTPS | Origin only (e.g. `https://app.ao.dev`) |
| HTTPS → HTTP downgrade | No referrer |

This balances analytics utility (same-origin full URLs are preserved) with privacy (no sensitive paths leaked in cross-origin requests).

---

## Permissions-Policy

### Default value

```
Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=(), usb=()
```

The dashboard does not require access to any browser hardware APIs. All listed features are disabled for the top-level frame and all embedded frames.

---

## Verifying Headers

Check the headers on any response using `curl`:

```bash
curl -sI https://app.ao.dev | grep -E "Content-Security|Strict-Transport|X-Frame|X-Content|Referrer|Permissions"
```

Or use [securityheaders.com](https://securityheaders.com) for a graded report.

---

## Related

- [Cloud Deployment](cloud-deployment.md) — deploying and managing the cloud daemon
- [Cloud Dashboard](cloud-dashboard.md) — browser-based project and agent management
- [Privacy & Data Policy](privacy.md) — data guarantees and provider policies
- [Custom Error Pages](custom-error-pages.md) — branded 404 and 500 responses
