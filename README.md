# textya · RSVP Reader

A single-page RSVP (Rapid Serial Visual Presentation) reader.

## Hosting

This is a static site. The entry point is `index.html` at the repo root, ready to deploy on Cloudflare Pages with no build step.

## Local preview

```sh
python3 -m http.server 8080
# then open http://localhost:8080
```

## Reading articles from URLs

Status: **implemented via Jina Reader (Approach A below).** Paste a URL into the
"Fetch URL" field in the source panel — or pass `#url=https://…` in the hash —
and the article text loads into the reader.

How it works: HTML articles are fetched through **Jina Reader**
(`https://r.jina.ai/<url>`), which bypasses CORS server-side and returns
pre-extracted article Markdown. `markdownToText()` strips the Markdown syntax to
plain prose, which feeds the existing RSVP pipeline. URLs ending in `.pdf` are
fetched directly and parsed with the inlined PDF.js. URLs are scheme-allowlisted
to `http:`/`https:` and fetches carry a 20s timeout. See `loadFromUrl` in
`index.html`. Privacy note: the URL is sent to Jina (a third party) — see the
Jina Security Deviation in `CLAUDE.md`; Approach D below is the planned
self-hosted replacement.

### The problem (design background)

- The app is purely client-side, so `fetch()` to an arbitrary article URL is blocked by CORS for most sites.
- The previous `stripHtml()` only removed `script / style / nav / footer / header / aside` and preferred `<article>` / `<main>`. It missed ads, paywalls, sidebars, comments, JS-rendered content, and has been removed in favour of Jina.
- Goal: paste a URL → get clean article text → feed the existing RSVP pipeline. **Serverless only** — Cloudflare Workers are fine, long-running servers are not.

### Approaches considered

- **A. Jina Reader (`https://r.jina.ai/<url>`)** — Zero infra. Returns pre-extracted Markdown, so we'd delete most of our parsing. Downsides: every URL is logged by a third party; rate limits; the free tier could go away.
- **B. Public CORS proxy + `@mozilla/readability` via esm.sh** — We'd own the extraction, but the fetch still routes through a flaky third party (corsproxy.io, allorigins). Combines A's dependency risk with more code. Listed for completeness; not recommended.
- **C. Bookmarklet** — User drags a bookmarklet to their bookmarks bar; on any article page it grabs `document.documentElement.outerHTML`, runs Readability inline, and opens rsvp-reader with the cleaned text in `#text=`. No third party. Works on paywalled / logged-in / JS-rendered pages. Cost: one-time install; awkward on mobile.
- **D. Cloudflare Worker (still serverless)** — A ~50-line Worker takes `?url=`, fetches, runs Readability + DOMPurify server-side, returns clean text. Free tier is 100k req/day. We control it; no third-party logging. Cost: one-time `wrangler deploy`; needs a Cloudflare account.

### What shipped, and what's next

Shipped **A (Jina Reader)** for the "paste a URL" flow — zero infra, clean
Markdown, minimal `index.html` change. The trade-off is third-party URL logging
and free-tier rate limits.

Next, to remove the third-party dependency: **D (self-hosted Cloudflare
Worker)** as a drop-in replacement for the Jina endpoint, and optionally **C
(bookmarklet)** for paywalled / logged-in / SPA pages that a server-side fetch
can't reach. B is not recommended.

### Repo-layout options

1. **Keep static root, add `/worker/` subdir** — `index.html` stays zero-build at the root. `/worker/wrangler.toml` + `/worker/src/index.ts` ship the Worker independently. Cleanest separation; recommended.
2. **One root `package.json` + Vite** — Bundle the bookmarklet and the Worker from a shared dependency tree. Every `index.html` change now passes through a build. More uniform; heavier.
3. **Static root + `/worker/` + `/tools/` for the bookmarklet build** — Splits the bookmarklet-bundling concern (esbuild one-shot that inlines Readability) into its own folder. Middle ground.

### Open questions

- Confirm C+D, or pick a different combination?
- Repo layout 1 / 2 / 3?
- Whose Cloudflare account hosts the Worker?
- Bookmarklet UX: surface a "drag me to your bookmarks" button on the load screen?
- Privacy stance for the Worker: log nothing, or aggregate counts only?
