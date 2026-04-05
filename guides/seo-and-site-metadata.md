# SEO and Site Metadata

The Animus cloud dashboard ships with a complete set of HTML metadata: standard `<meta>` tags, Open Graph and Twitter Card markup, a canonical URL strategy, a `favicon.ico` / PNG icon set, and a web app manifest (`manifest.json`). This guide documents each element and explains how to customise them for self-hosted deployments.

For branded error pages see [Custom Error Pages](custom-error-pages.md). For security headers see [Security Headers](security-headers.md).

---

## Overview

Metadata falls into three categories:

| Category | Elements | Purpose |
|---|---|---|
| **SEO** | `<title>`, `<meta name="description">`, canonical `<link>` | Search engine indexing and ranking |
| **Social** | Open Graph tags, Twitter Card tags | Link preview appearance on social platforms and messaging apps |
| **PWA / browser** | Favicon, icon set, `manifest.json`, theme-colour | Browser tab, home-screen icon, and installed-app presentation |

---

## Page Title and Description

### Title format

Page titles follow the pattern:

```
{Page Name} — Animus
```

Examples:

| Page | Title |
|---|---|
| Projects overview | `Projects — Animus` |
| Specific project | `my-project — Animus` |
| Deployment detail | `Deployment dep_01HXYZ — Animus` |
| Billing | `Billing — Animus` |
| 404 error | `Page Not Found — Animus` |

### Meta description

Each page sets a unique `<meta name="description">` tag:

```html
<!-- Dashboard home -->
<meta name="description"
      content="Animus cloud dashboard — manage projects, monitor agents, and review logs from your browser.">

<!-- Project page -->
<meta name="description"
      content="Project my-project: deployment status, active agents, and workflow history.">
```

Descriptions are capped at 160 characters. They are generated server-side using the project or deployment data returned by the API.

### Robots directives

| Page type | `robots` value | Rationale |
|---|---|---|
| Authenticated dashboard pages | `noindex, nofollow` | Private data — must not be indexed |
| Public marketing pages (if any) | `index, follow` | Intentionally public |
| Error pages (404) | `noindex` | Avoid indexing broken URLs |
| Error pages (500) | `noindex, noarchive` | Transient error, no caching |

```html
<meta name="robots" content="noindex, nofollow">
```

---

## Canonical URLs

Every page sets a `<link rel="canonical">` pointing to the canonical form of the URL — HTTPS, no trailing slash, no query parameters that affect page identity:

```html
<link rel="canonical" href="https://app.ao.dev/projects/proj_01HXYZ">
```

This prevents duplicate-content issues when the same page is reachable via multiple paths (e.g., HTTP vs. HTTPS, with or without `www`).

---

## Open Graph Tags

Open Graph metadata controls how the dashboard link appears when shared on Slack, LinkedIn, X, Discord, and other platforms that support the OG protocol.

### Default (dashboard-wide) tags

```html
<meta property="og:site_name" content="Animus">
<meta property="og:type"      content="website">
<meta property="og:url"       content="https://app.ao.dev/">
<meta property="og:title"     content="Animus Cloud Dashboard">
<meta property="og:description"
      content="Manage autonomous agent workflows, monitor deployments, and review logs from your browser.">
<meta property="og:image"     content="https://app.ao.dev/og-image.png">
<meta property="og:image:width"  content="1200">
<meta property="og:image:height" content="630">
<meta property="og:image:alt" content="Animus Cloud Dashboard — project overview screenshot">
<meta property="og:locale"    content="en_US">
```

### Page-specific overrides

Authenticated pages do not serve Open Graph tags with private content (project names, deployment IDs). The tags for authenticated routes fall back to the default dashboard-wide values.

Public-facing pages (marketing, documentation) set page-specific `og:title`, `og:description`, and `og:url`.

### Open Graph image

The OG image is a static `1200 × 630` PNG at `/og-image.png`. It shows:

- The Animus wordmark on a dark background
- The tagline "Autonomous agent workflows, built for your team"
- The `app.ao.dev` domain in the lower-right corner

To replace the OG image in a self-hosted deployment, provide a `1200 × 630` PNG at the same path in your `dist/` or `public/` directory.

---

## Twitter Card Tags

Twitter Card tags control link preview appearance on X (formerly Twitter).

```html
<meta name="twitter:card"        content="summary_large_image">
<meta name="twitter:site"        content="@ao_dev">
<meta name="twitter:creator"     content="@ao_dev">
<meta name="twitter:title"       content="Animus Cloud Dashboard">
<meta name="twitter:description"
      content="Manage autonomous agent workflows, monitor deployments, and review logs from your browser.">
<meta name="twitter:image"       content="https://app.ao.dev/og-image.png">
<meta name="twitter:image:alt"   content="Animus Cloud Dashboard — project overview screenshot">
```

`summary_large_image` renders the OG image at full width above the link title.

---

## Favicon and Icon Set

### File inventory

```
public/
  favicon.ico          # 32×32 multi-resolution ICO (16, 32 px frames)
  favicon-16x16.png    # 16×16 PNG for classic browsers
  favicon-32x32.png    # 32×32 PNG
  apple-touch-icon.png # 180×180 PNG for iOS home-screen bookmarks
  icon-192.png         # 192×192 PNG for Android home-screen / PWA
  icon-512.png         # 512×512 PNG for splash screens and app stores
  og-image.png         # 1200×630 PNG for Open Graph / Twitter Card
  safari-pinned-tab.svg # Monochrome SVG for Safari pinned tabs
```

### HTML link tags

```html
<link rel="icon" href="/favicon.ico" sizes="any">
<link rel="icon" href="/favicon-32x32.png" type="image/png" sizes="32x32">
<link rel="icon" href="/favicon-16x16.png" type="image/png" sizes="16x16">
<link rel="apple-touch-icon" href="/apple-touch-icon.png">
<link rel="mask-icon" href="/safari-pinned-tab.svg" color="#0f4c81">
<link rel="manifest" href="/manifest.json">
<meta name="theme-color" content="#0f4c81">
<meta name="msapplication-TileColor" content="#0f4c81">
```

### Replacing icons in self-hosted deployments

1. Replace the files in `public/` with your own assets at the same names and dimensions.
2. Update `color` in the `mask-icon` link and `content` in the `theme-color` and `msapplication-TileColor` meta tags to match your brand colour.
3. Rebuild (`npm run build`) — the updated assets will be copied to `dist/`.

---

## manifest.json

The Web App Manifest enables progressive-web-app installation on Android and Chrome desktop. It also provides metadata used by some password managers and browser extensions.

### Default manifest

```json
{
  "name": "Animus Cloud Dashboard",
  "short_name": "Animus",
  "description": "Manage autonomous agent workflows, monitor deployments, and review logs.",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#0f1117",
  "theme_color": "#0f4c81",
  "orientation": "any",
  "icons": [
    {
      "src": "/icon-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "/icon-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any maskable"
    }
  ],
  "categories": ["productivity", "developer tools"],
  "lang": "en-US",
  "dir": "ltr"
}
```

### Key fields

| Field | Value | Notes |
|---|---|---|
| `name` | `"Animus Cloud Dashboard"` | Full name shown during installation prompt |
| `short_name` | `"Animus"` | Name shown on home screen under the icon |
| `start_url` | `"/"` | URL opened when launched from home screen |
| `display` | `"standalone"` | Hides the browser chrome when installed |
| `background_color` | `"#0f1117"` | Splash screen background while the app loads |
| `theme_color` | `"#0f4c81"` | Browser UI chrome colour (address bar, task switcher) |
| `purpose` on icons | `"any maskable"` | Supports both square and rounded-corner icon masks |

### Customising for self-hosted deployments

Edit `public/manifest.json` before building:

```json
{
  "name": "Acme Animus Eye",
  "short_name": "Acme Animus",
  "description": "Acme's internal autonomous agent platform.",
  "theme_color": "#1a3a5c",
  "background_color": "#0a0f1a"
}
```

Run `npm run build` to embed the updated manifest in the production output.

---

## Verifying Metadata

### Check meta tags

```bash
curl -s https://app.ao.dev | grep -E '<meta|<link rel|<title'
```

### Check the manifest

```bash
curl -s https://app.ao.dev/manifest.json | jq .
```

### Validate Open Graph tags

Use the [Open Graph Debugger](https://developers.facebook.com/tools/debug/) or `ogp.me` to preview how a URL will render when shared. Pass the URL of a public page (not an authenticated route) — authenticated pages intentionally omit rich OG metadata.

### Lighthouse audit

Run a Lighthouse audit in Chrome DevTools (Audits tab → SEO) against a public page to verify title, description, canonical, and robots tags are correctly set.

---

## Related

- [Custom Error Pages](custom-error-pages.md) — branded 404 and 500 pages
- [Security Headers](security-headers.md) — CSP, HSTS, and X-Frame-Options
- [Cloud Dashboard](cloud-dashboard.md) — navigating the web application
- [Cloud Deployment](cloud-deployment.md) — deploying and managing the cloud daemon
