# textya · RSVP Reader

## Purpose

Single-page RSVP (Rapid Serial Visual Presentation) reader. User pastes text, drops a file, or passes a URL hash param; the app flashes one word at a time at a configurable WPM. No accounts, no backend, no per-user state beyond `localStorage`.

## Audience

End users on desktop or mobile browsers. The app is intentionally self-contained so it works offline and on Cloudflare Pages with zero build.

## External Services & Credentials

- **Google Fonts** (`fonts.googleapis.com`, `fonts.gstatic.com`) — loaded at runtime in `index.html:9–11`. Tracked as a Security Deviation (see below) until fonts are self-hosted.
- **Jina Reader** (`r.jina.ai`) — user-initiated only. When a user fetches an article by URL (panel "Fetch URL" field or `#url=` hash param), the URL is sent to Jina, which bypasses CORS server-side and returns pre-extracted article Markdown. No credentials sent. Tracked as a Security Deviation (see below). See `loadFromUrl` in `index.html`.
- **No credentials.** No API keys, no auth, no secrets.
- **Future:** a Cloudflare Worker for URL-article extraction is under discussion (see README "Reading articles from URLs — design options") as a self-hosted replacement for Jina. Not yet built.

## Constraints

- Static single file (`index.html`) deployable to Cloudflare Pages with no build step.
- Vanilla JS only — no framework, no bundler, no `package.json`.
- Must work offline once loaded (OpenDyslexic is self-hosted; Google Fonts is the only external dependency).
- Mobile-friendly: drag-drop, paste, file picker, and hash params must all work on touch.

## Out of Scope

- Server-side rendering, authentication, multi-user state.
- Cloud sync of reading position or settings — `localStorage` only.
- Editing the inlined PDF.js bundle by hand — replace the whole bundle when upgrading.
- Adding npm/Vite/Webpack to the root project — see README for layout options when the Worker lands.

## Tooling

- Language: Vanilla JavaScript (ES2020+), HTML5, CSS3
- Package manager: none
- Linter / formatter: none currently (candidate: Biome — single binary, no config)
- Test runner: none currently
- Run / preview: `python3 -m http.server 8080`
- Test: n/a
- Lint: n/a
- Build: n/a (static)
- Deploy: push to `main`; Cloudflare Pages serves the repo root

## Security Context

- **Approved frameworks:** vanilla browser APIs only. No CDN-loaded JS frameworks. PDF.js is inlined and version-pinned.
- **Identity provider:** none — no accounts.
- **Secret manager:** n/a — no secrets stored.
- **Data classification:** pasted/dropped/imported text is local-only and never transmitted. The one exception: when a user fetches an article by URL, the **URL** (not the page text) is sent to Jina Reader for extraction. See the Jina deviation below.
- **Compliance requirements:** none.
- **Threat model:** the realistic attackers are (a) malicious URLs in `#url=` hash params from phishing links, (b) malicious PDFs dropped or fetched, (c) malicious HTML returned by `fetch()`. All inputs flow through `escapeHtml` (`index.html:2525`) before reaching any `innerHTML` sink, which is the load-bearing safety property.
- **Prohibited patterns:** new uses of `innerHTML`, `outerHTML`, `document.write`, `insertAdjacentHTML`, `eval`, or `new Function()` without an explicit comment justifying why the input is trusted. Adding event-handler attributes (`onerror=`, `onclick=`) by string concatenation is banned.

## Applicable Security References

The universal Tier 1 rules in `~/.claude/skills/secure-coding/SKILL.md` apply at all times. For this app the relevant references are:

- XSS / output encoding (Rule 6) — every user-supplied string must hit `escapeHtml` before any `innerHTML` write.
- Input validation at trust boundaries — `fetch()` URLs from `#url=` must be scheme-allowlisted to `http:`/`https:` only.
- Dependency hygiene — inlined PDF.js must be pinned to a CVE-free version; document the SHA when bumping.
- (Future) SSRF + rate limiting for the planned Cloudflare Worker — URL allowlist, request timeouts, response size cap, per-IP rate limit.

## Security Deviations

- **2026-05-18 — Google Fonts loaded from CDN**
  - Scope: `index.html:9–11`
  - Rule deviated from: third-party CDN exposure / availability + privacy
  - Reason: avoids ~2 MB of self-hosted font weights; many fonts the app offers are uncommon enough that few users will pick them, so paying download cost for all of them up-front is wasteful.
  - Mitigation: `display=swap` set so missing fonts fall back to system fonts; CDN failure does not block render.
  - Remediation plan: self-host the 3–4 most-used fonts and remove the rest from the initial load. Tracked, not scheduled.

- **2026-05-18 — Inlined PDF.js version not currently audited**
  - Scope: `index.html:1835–3160` (minified bundle)
  - Rule deviated from: dependency hygiene
  - Reason: version was inlined without a version comment when first added.
  - Mitigation: PDFs only render in the reader's own context; the bundle runs in a worker for parsing.
  - Remediation plan: identify the bundled version, check against PDF.js CVE list (PDF.js < 4.2.67 has CVE-2024-4367, arbitrary JS execution via crafted font), upgrade if vulnerable, add a `// pdfjs vX.Y.Z` comment at the top of the bundle.

- **2026-05-18 — No Content-Security-Policy**
  - Scope: `index.html` `<head>` (lines 1–12)
  - Rule deviated from: defense-in-depth against script injection
  - Reason: app currently relies on input encoding (`escapeHtml`) and DOM-text-only sinks; a CSP was never added.
  - Mitigation: the only `innerHTML` sinks pass values through `escapeHtml`; class names at those sinks are hardcoded, not interpolated from user input.
  - Remediation plan: add `<meta http-equiv="Content-Security-Policy">` allowing `'self'` for scripts, `fonts.googleapis.com`/`fonts.gstatic.com` for fonts, and `r.jina.ai` in `connect-src` for URL fetches; verify nothing breaks.

- **2026-05-18 — No URL scheme allowlist on `#url=` fetch** — RESOLVED 2026-05-26
  - Scope: `loadFromUrl` in `index.html` (now the single entry point for both `#url=` and the panel "Fetch URL" field).
  - Rule deviated from: input validation at trust boundaries
  - Resolution: URLs are parsed with `new URL()` and rejected unless the protocol is `http:`/`https:`; the fetch carries `AbortSignal.timeout(20000)`. Invalid or disallowed schemes show a toast and never trigger a request.

- **2026-05-26 — Article URLs sent to Jina Reader (third party)**
  - Scope: `loadFromUrl` in `index.html` (HTML-article branch → `https://r.jina.ai/<url>`).
  - Rule deviated from: third-party exposure / privacy (user URLs transit a service we don't control; Jina may log them, and free-tier rate limits apply).
  - Reason: the app is purely client-side, so direct `fetch()` to arbitrary articles is CORS-blocked; Jina bypasses CORS and returns clean pre-extracted Markdown in one call, with zero infra.
  - Mitigation: user-initiated only (never automatic); `http:`/`https:` scheme allowlist before any request; 20s timeout; no credentials/cookies sent; only the URL leaves the browser (page text is whatever Jina returns and is rendered text-only through the existing `escapeHtml` path, so no new injection sink).
  - Remediation plan: self-host extraction via the Cloudflare Worker described in the README ("Reading articles from URLs"), removing the third-party dependency and URL logging.
