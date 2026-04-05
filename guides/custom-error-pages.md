# Custom Error Pages

The AO cloud dashboard serves branded error pages for HTTP 404 (Not Found) and HTTP 500 (Internal Server Error) responses. This guide describes what each page shows, how they behave in the context of a single-page application, and how to customise them for self-hosted deployments.

For security headers applied to all responses including error pages see [Security Headers](security-headers.md).

---

## Overview

Generic browser error pages break the user's mental model and provide no recovery path. AO's branded error pages:

- Keep the site's visual identity (logo, colour tokens, typography)
- Explain what went wrong in plain language
- Offer one or more actionable next steps
- Include a support reference for persistent issues

Both pages are statically rendered — they contain no JavaScript dependencies and load even when the application bundle fails to initialise.

---

## 404 — Not Found

Served when a request path does not match any known route.

### What the page shows

| Element | Content |
|---|---|
| AO logo | Links back to the dashboard home (`/`) |
| Status code | `404` rendered in the display font |
| Headline | "Page not found" |
| Body copy | "The page you're looking for doesn't exist or has been moved." |
| Primary action | **Go to Dashboard** button → `/` |
| Secondary action | **Contact Support** link → `https://ao.dev/support` |
| Footer | Build version and current UTC timestamp |

### Trigger conditions

| Condition | Example |
|---|---|
| Unknown route in the SPA router | `/settings/nonexistent-section` |
| Project or deployment ID not found in the API | `/projects/proj_does_not_exist` |
| Direct navigation to a deep link that no longer exists | A bookmark to a deleted deployment |

> **Note (v60 bugfix):** Prior to v60, unauthenticated users navigating directly to a protected route (e.g. bookmarked deep links, shared URLs) were incorrectly served the 404 error page. This happened because the authentication guard ran after the route resolver attempted to load the resource, which returned a 404 before the redirect could fire. Starting in v60 the authentication check runs first in the route guard chain. Unauthenticated users are now redirected to `/login?redirect=<original-path>` and land on the correct page after signing in.

### SPA routing vs. server 404

The dashboard is a React single-page application served from a single `index.html`. The web server is configured to fall back to `index.html` for all unmatched paths so the React router can handle client-side navigation. A true 404 (HTTP status `404`) is only returned when:

1. The request path matches a known static asset pattern (`.js`, `.css`, `.png`, etc.) but the file does not exist.
2. The API returns a 404 for a resource load, triggering the in-app error boundary which renders the 404 page with a `404` status code via `<meta name="ao:http-status">`.

---

## 500 — Internal Server Error

Served when an unhandled exception occurs during server-side rendering or when the API returns a 5xx response that the client cannot recover from.

### What the page shows

| Element | Content |
|---|---|
| AO logo | Links back to the dashboard home (`/`) |
| Status code | `500` rendered in the display font |
| Headline | "Something went wrong" |
| Body copy | "An unexpected error occurred. Our team has been notified." |
| Incident ID | Short alphanumeric ID for support reference (e.g. `INC-A3F9`) |
| Primary action | **Try Again** button — reloads the current page |
| Secondary action | **Go to Dashboard** button → `/` |
| Tertiary action | **Contact Support** link with the incident ID pre-filled |
| Footer | Build version and current UTC timestamp |

### Trigger conditions

| Condition | Example |
|---|---|
| Unhandled exception in server-side code | Uncaught promise rejection in the API gateway |
| React error boundary catch | A component throws during render |
| API gateway returns 502 / 503 / 504 | Upstream cloud daemon unreachable |
| Build artefact missing at runtime | Corrupted deployment |

### Incident ID

Each 500 response generates a short unique incident ID. The ID is:

- Logged server-side with the full stack trace
- Included in the `X-Ao-Incident-Id` response header
- Displayed on the page so users can quote it when filing a support request

Retrieve the incident ID from a response header:

```bash
curl -sI https://app.ao.dev/some-failing-path | grep X-Ao-Incident-Id
# X-Ao-Incident-Id: INC-A3F9
```

---

## Error Page Behaviour in AO Cloud

The cloud dashboard includes both pages as pre-rendered HTML artefacts in the production build. They share the design system's CSS custom properties (colour tokens) but have no runtime JavaScript dependencies. This ensures:

- The page renders even if the JavaScript bundle fails to load
- The page is correctly indexed by search engines (for the 404 case) and marked `noindex` (for the 500 case)
- All security headers (CSP, HSTS, X-Frame-Options) are applied to error responses in the same way as successful responses

---

## Self-Hosted Deployments

If you self-host the AO dashboard, copy the pre-built error pages from the production build output and configure your web server to serve them for the appropriate status codes.

### Build output

After `npm run build` the error pages are at:

```
dist/
  404.html
  500.html
```

### Nginx

```nginx
server {
    root /var/www/ao-dashboard/dist;

    # SPA fallback — must come before error_page directives
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Static assets — return a real 404 if the file is missing
    location ~* \.(js|css|png|svg|ico|woff2?)$ {
        try_files $uri =404;
    }

    error_page 404 /404.html;
    error_page 500 502 503 504 /500.html;

    location = /404.html {
        internal;
    }

    location = /500.html {
        internal;
    }
}
```

### Caddy

```caddyfile
app.ao.dev {
    root * /var/www/ao-dashboard/dist

    handle_errors {
        @404 expression {http.error.status_code} == 404
        @500 expression {http.error.status_code} >= 500

        handle @404 {
            rewrite * /404.html
            file_server
        }
        handle @500 {
            rewrite * /500.html
            file_server
        }
    }

    # SPA catch-all
    try_files {path} /index.html
    file_server
}
```

### Vercel / Netlify

Both platforms pick up `404.html` automatically from the build output. For 500 errors, configure a custom error page in the platform dashboard or `vercel.json` / `netlify.toml`:

```json
// vercel.json
{
  "cleanUrls": true,
  "trailingSlash": false,
  "rewrites": [
    { "source": "/(.*)", "destination": "/index.html" }
  ],
  "routes": [
    { "src": "/404", "dest": "/404.html", "status": 404 }
  ]
}
```

```toml
# netlify.toml
[[redirects]]
  from    = "/*"
  to      = "/index.html"
  status  = 200

[[redirects]]
  from    = "/404"
  to      = "/404.html"
  status  = 404
```

---

## Customising Branding

The error pages reference the same CSS custom properties as the main application. To restyle them for a self-hosted deployment, override the properties in a `custom.css` file linked in the page `<head>`:

```css
/* custom.css */
:root {
  --ao-color-brand:       #0f4c81;
  --ao-color-background:  #ffffff;
  --ao-color-text:        #1a1a2e;
  --ao-font-display:      "Inter", sans-serif;
}
```

Reference your stylesheet before the error page's default CSS:

```html
<head>
  <link rel="stylesheet" href="/custom.css">
  <link rel="stylesheet" href="/assets/error-pages.css">
</head>
```

---

## Accessibility

Both error pages meet WCAG 2.1 AA:

- All interactive elements are keyboard-navigable and focus-visible
- Colour contrast ratio ≥ 4.5:1 for normal text, ≥ 3:1 for large text
- `lang="en"` set on `<html>`; `role="main"` on the primary content region
- Status code is wrapped in `<h1>` and accompanied by a visible text description (not just a number)

---

## Related

- [Security Headers](security-headers.md) — CSP, HSTS, and X-Frame-Options on all responses
- [SEO and Site Metadata](seo-and-site-metadata.md) — meta tags, Open Graph, favicon, and manifest
- [Cloud Dashboard](cloud-dashboard.md) — navigating the web application
- [Troubleshooting](troubleshooting.md) — diagnosing persistent 500 errors
