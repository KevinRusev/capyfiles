# Architecture

A high-level tour of how Capy Files is put together. Internal names, paths and configuration are left out on purpose.

## Guiding rule

**File content never leaves the device.** Every design decision follows from that: processing happens in the browser,
the server only delivers the app and AI models, and the few things it stores (accounts and subscription state) contain
nothing about the files people work on.

## The pieces

### 1. The app (browser)

- **React 19 + TypeScript + Vite**, styled with Tailwind CSS. Each tool is its own lazily loaded chunk, so opening one
  tool doesn't download the other 58.
- **One tool registry** describes every tool: name, category, accepted file types, limits, and whether it's released.
  Navigation, search, routing, the sitemap and the SEO metadata are all generated from it, so they cannot drift apart.
- **Heavy work runs in Web Workers**, off the main thread: PDF rewriting, image encoding, video transcoding, AI
  inference. The page stays responsive while a 500-page PDF or a one-hour recording is processed.
- **Engines:**
  - PDF: pdf.js (rendering, text), pdf-lib (editing), qpdf compiled to WebAssembly (repair, encryption, linearisation),
    plus a custom image-recompressing compressor.
  - Images: MozJPEG, OxiPNG, WebP and AVIF encoders in WebAssembly, a custom colour quantiser, HEIC decoding.
  - Video and audio: WebCodecs-based demuxing, encoding and muxing, so video is encoded by the device's hardware where
    available.
  - AI: see [on-device-ai.md](on-device-ai.md).

### 2. Offline and installability

- A **service worker** precaches the app shell and caches engine chunks the first time a tool uses them. Big optional
  parts (video encoders, AI runtimes) are fetched on demand instead of on everyone's first visit.
- The app is an installable **PWA** with its own icon, shortcuts and offline behaviour.
- Updates are offered with a "new version ready" prompt rather than swapped in mid-task.

### 3. The edge (Cloudflare)

- **Static assets** (the pre-rendered pages, scripts, styles) are served from Cloudflare's edge with long-lived,
  content-hashed caching.
- A **Worker** handles the small dynamic part:
  - **Model delivery** from R2 object storage with correct content types, range requests (so downloads can resume) and
    immutable caching.
  - **A small JSON API** for accounts and subscriptions, with schema validation, rate limiting and origin checks.
  - **Payment webhooks** from Stripe, verified by signature, processed idempotently and in order; refunds, disputes
    and withdrawals update the plan the same way.
  - **Scheduled jobs** for renewal reminders, claimed before sending so a retry never sends twice.
- **D1** (SQLite at the edge) stores accounts, subscription state and legally required records. It holds nothing
  derived from user files.

### 4. Accounts and payments

- **Firebase Authentication** (Google, Microsoft, GitHub, or a one-time email link; no passwords). The Worker verifies sign-in tokens
  itself on every request.
- **Stripe Managed Payments** (Link is the merchant of record) handles VAT, receipts and invoices. Checkout is
  embedded in the site and only exists once the buyer has ticked both consent boxes. Subscription state reaches the
  app only through verified webhooks, and the browser receives it as a signed entitlement it verifies itself.
- The free tools never require an account, and the sign-in code is only loaded for people who sign in.

## Security headers and cross-origin isolation

- **Strict Content Security Policy**: scripts only from the site itself (inline scripts allowed by hash), no `eval`, and
  WebAssembly compilation allowed explicitly. Plus HSTS, `nosniff`, a strict referrer policy, a locked-down
  Permissions-Policy and frame protection.
- **Two kinds of pages:**
  - *Tool pages* are **cross-origin isolated** (COOP + COEP), which unlocks `SharedArrayBuffer` and therefore
    multi-threaded WebAssembly and AI.
  - *Sign-in, checkout, account and legal pages* are **not isolated**, because isolation breaks third-party sign-in and
    payment popups.
- The header policy lives in **one place**. The build turns it into the static header rules and the Worker applies the
  same policy to its own responses; a test suite checks that every path gets exactly the headers the policy says.
- Moving between the two kinds of page is always a **full page load** (headers only apply on document load), handled
  by the app's link component.

## SEO

- Every public page is **pre-rendered to static HTML at build time** by rendering the real React routes, including
  lazily loaded tools.
- Each page gets its title, description, canonical URL, Open Graph and Twitter tags, and JSON-LD (web application,
  breadcrumbs). Paid tools are never described as free; a test enforces it.
- The build also writes the sitemap and robots rules, and unknown URLs get a real 404 page.

## Quality gates

Every build runs, in order:

1. TypeScript strict type checks (app and Worker).
2. Lint (correctness and React hook rules are errors).
3. **900+ unit tests** (header policy, webhooks and signatures, entitlement rules, model delivery, PDF internals,
   resumable downloads, SEO rules and more).
4. Production build and pre-render of every page.
5. **Licence compliance check**: every shipped dependency and AI model is checked against an allowlist. GPL, AGPL,
   SSPL, non-commercial or unknown licences fail the build, and the built output is scanned to prove removed libraries
   are really gone. Third-party notices are generated automatically.
