# Rocket.Chat inside Microsoft Teams — embedded tab wrapper

A sideloadable Teams app package that renders a Rocket.Chat workspace as a **personal (static) tab** in the Teams desktop client. Pure iframe wrapper: no bot, no message sync, no Graph, no SSO bridge. Teams is the container; Rocket.Chat runs as-is.

## Package contents

```
rocketchat-teams-app.zip     ← ready to sideload (points at open.rocket.chat)
package/
  manifest.json              ← app manifest, schema v1.27
  color.png                  ← 192×192 color icon
  outline.png                ← 32×32 white/transparent outline icon
wrapper/
  wrapper.html               ← OPTIONAL runtime-URL-entry page (see below)
```

App ID (GUID): `80898bb0-c18f-4662-8624-c21bdd392035` — regenerate if you fork this package for a second app in the same tenant.

Note: `packageName` was removed from the manifest schema in v1.17+ and is rejected by validation ("additional properties" error). The `id` GUID is the app's unique identifier; do not re-add `packageName` unless you downgrade to `manifestVersion` ≤1.16.

## 1. Swapping in your Rocket.Chat URL

The zip ships pointed at `https://open.rocket.chat/` so it sideloads and opens immediately. To point it at your own instance, edit `package/manifest.json`:

1. Replace both `contentUrl` and `websiteUrl` in `staticTabs[0]` with `https://chat.yourcompany.com/`.
2. Replace the `validDomains` entries with your domain, e.g. `"chat.yourcompany.com"` (a wildcard like `"*.yourcompany.com"` is allowed). **This is the step people miss — if the tab's domain is not in `validDomains`, Teams refuses to load it and you get a blank tab.** Note that `validDomains` must contain hostnames only, no `https://` scheme and no paths.
3. Re-zip `manifest.json`, `color.png`, `outline.png` **at the root of the zip** (no containing folder) and re-sideload.

```bash
cd package && zip -r ../rocketchat-teams-app.zip manifest.json color.png outline.png
```

## 2. Rocket.Chat server changes (required — the embed is blocked without these)

Rocket.Chat ships with `X-Frame-Options: sameorigin` by default, which blocks framing by Teams.

### What is configurable in the Rocket.Chat admin UI

**Admin → Settings → General → "Restrict access inside any Iframe"** — turn this **OFF**. This stops Rocket.Chat from sending `X-Frame-Options`. (There is a companion "Options to X-Frame-Options" text setting, but it is of limited use: the only per-domain value `X-Frame-Options` ever supported, `ALLOW-FROM`, is dead in all modern browsers/WebView2, so you cannot allow-list Teams via this header.)

What is **not** configurable in the admin UI: a CSP `frame-ancestors` allow-list. Rocket.Chat does not expose `frame-ancestors` as an admin setting (a long-standing open request, see RocketChat/Rocket.Chat#18485). So the correct pattern is: disable the header in the admin UI, then **re-add a scoped `frame-ancestors` policy at the reverse proxy** so you're not open to framing by any site.

### NGINX

```nginx
server {
    ...
    location / {
        proxy_pass http://rocketchat_backend;

        # Belt-and-braces: strip any X-Frame-Options coming from the app
        proxy_hide_header X-Frame-Options;
        proxy_hide_header Content-Security-Policy;

        # Allow framing ONLY by Teams (classic + new Teams hosts) and ourselves
        add_header Content-Security-Policy "frame-ancestors 'self' https://teams.microsoft.com https://*.teams.microsoft.com https://teams.cloud.microsoft https://*.cloud.microsoft" always;
    }
}
```

Notes:

- `frame-ancestors` supersedes `X-Frame-Options` in every modern engine; where both are present, browsers honor `frame-ancestors`. Stripping XFO at the proxy avoids conflicts.
- The new Teams client is served from `*.cloud.microsoft` hosts; the classic domain is `teams.microsoft.com`. Including both covers desktop (WebView2 renders the same origins), web, and the transition period.
- If you use the optional wrapper page (below), its hosting domain must **also** be in the `frame-ancestors` list, because the ancestor chain becomes `teams.microsoft.com → wrapper host → rocket.chat`.
- If Rocket.Chat sets its own CSP for other reasons, merge rather than `proxy_hide_header` it.

## 3. Sideloading for testing

1. Tenant prerequisite: an admin must allow custom app uploads (Teams admin center → Teams apps → Setup policies → **Upload custom apps** ON, or org-wide app settings → allow custom apps).
2. In Teams desktop: **Apps → Manage your apps → Upload an app → Upload a custom app**, pick `rocketchat-teams-app.zip`.
3. The app appears in the left rail (personal scope). Pin it via right-click if desired.
4. To update after editing the manifest: remove the app, bump `"version"` in the manifest, re-upload.

Optional pre-flight check: upload the zip to the Developer Portal for Teams (dev.teams.microsoft.com → Apps → Import app) — it runs full manifest validation and pinpoints schema errors.

## 4. Runtime URL entry (optional wrapper)

The shipped package hardcodes the workspace URL — simplest, zero hosting. If you want end users to type their own workspace URL on first launch, use `wrapper/wrapper.html`:

1. Host it on any HTTPS static host you control (e.g. `https://apps.yourcompany.com/rc-tab/wrapper.html`).
2. Point `contentUrl`/`websiteUrl` at the wrapper URL and add the wrapper's domain to `validDomains`.
3. The wrapper asks for the workspace URL once, stores it in `localStorage` (per device), and iframes it. A small "change workspace" button lets users switch.

Caveats specific to this mode:

- The RC server's `frame-ancestors` must include the wrapper's domain (see above).
- Storage partitioning means the saved URL lives inside the Teams webview only — users re-enter it once per device/client.
- The wrapper calls `app.initialize()` / `app.notifySuccess()` via TeamsJS, so it stays compatible even if a loading indicator is enabled later.

## 5. What was verified vs. inferred (as of July 2026)

**Verified against current Microsoft docs:**

- Latest app manifest schema is **v1.27** (OfficeDev/microsoft-teams-app-schema releases; MS Learn schema reference). This package targets `manifestVersion: "1.27"`.
- A bare external URL **does work** in a static tab without the Teams JS SDK, **provided `showLoadingIndicator` is not set to true**. Per MS Learn "Build a content page for tab": calling `app.initialize()` + `app.notifySuccess()` is required only when the loading indicator is enabled — otherwise Teams shows an error/blank after a 30s timeout. This manifest deliberately omits `showLoadingIndicator`, so pointing straight at the RC root is safe. Decision point resolved: **no thin wrapper is required** for the hardcoded variant.
- Rocket.Chat's iframe restriction lives at **Admin → Settings → General → "Restrict access inside any Iframe"** and controls `X-Frame-Options` only; `frame-ancestors` is not admin-configurable and must be done at the proxy.

**Inferred / not independently verifiable without your environment:**

- The exact `frame-ancestors` host list. `teams.microsoft.com` + `*.cloud.microsoft` covers current classic/new Teams; if you see a blocked frame, check the webview console for the reporting ancestor origin and add it.
- Behavior of your specific auth providers (SAML/OAuth) inside the webview — test with your IdP.

## 6. Known limitations of the iframe approach

- **Auth:** users log into Rocket.Chat manually inside the tab; there is no Teams SSO. Rocket.Chat's own session token is kept in `localStorage` (not cookies), which is why plain username/password login generally survives framing and third-party-cookie blocking. However: (a) webview **storage partitioning** means the Teams-embedded session is separate from the user's browser session — they log in again inside Teams; (b) **OAuth/SAML logins can break** inside the iframe if the IdP itself refuses framing or relies on third-party cookies — Google, Microsoft, and most IdPs block being framed, so redirect-based SSO flows may dead-end. If your instance uses SAML/OAuth only, test this first; popup-based flows fare better than redirect flows. If anything auth-related sets cookies at your proxy, mark them `SameSite=None; Secure` or they'll be dropped in the third-party context.
- **No deep-linking:** Teams notifications, `msteams://` links, and channel deep links don't map to RC rooms; RC's own desktop notifications may not fire from inside the webview.
- **Calls/media:** Jitsi/Pexip/RC calls need camera/mic/screen-capture permissions inside the webview; behavior varies by Teams client build. Treat A/V inside the tab as best-effort.
- **Downloads/uploads** are handled by the Teams webview and can behave differently than a browser.
- **One shared iframe context:** no per-channel tabs, no presence sync, no cross-product search — Teams knows nothing about the content.
