# Plannotator — Deep Dive

<scope-and-method>

**Question being answered:** can Plannotator give us a mobile-first, cloud-hosted surface where a Claude Code agent publishes a Markdown document, the owner annotates it from a phone (voice dictation + pasted images), and the agent pulls the annotations back?

**What was actually read (verified):**

- Full source tarball of `backnotprop/plannotator@main`, downloaded 2026-07-29 from `https://codeload.github.com/backnotprop/plannotator/tar.gz/refs/heads/main`. 1,351 files. All file paths below are relative to the repo root.
- `README.md` (432 lines, read in full).
- `CONTRIBUTING.md`, `LICENSE-MIT`, `LICENSE-APACHE`, `package.json`, `packages/ui/package.json`, `packages/core/package.json`.
- `packages/ui/HANDOFF.md` (431 lines) — the single most important document in the repo for our purposes.
- `packages/server/remote.ts`, `packages/server/share-url.ts`, `packages/ui/utils/sharing.ts`, `packages/ui/utils/callback.ts`, `packages/ui/types.ts`, `packages/ui/utils/parser.ts` (export section), `packages/ui/components/ImportModal.tsx`, `packages/ui/hooks/useIsMobile.ts`.
- `apps/portal/SELF-HOSTING.md`, `apps/portal/ANNOTATE.md`, `apps/portal/index.tsx`, `apps/paste-service/**` (handler, cors, bun + cloudflare targets, wrangler.toml).
- `apps/hook/.claude-plugin/plugin.json`, `apps/hook/hooks/hooks.json`, `.claude-plugin/marketplace.json`.
- A dedicated sub-investigation of the mobile UI reading `packages/editor/App.tsx`, `packages/editor/components/AppHeader.tsx`, `packages/ui/components/Viewer.tsx`, `AnnotationPanel.tsx`, `AnnotationToolbar.tsx`, `AnnotationToolstrip.tsx`, `CommentPopover.tsx`, `FloatingQuickLabelPicker.tsx`, `hooks/useAnnotationHighlighter.ts`, `hooks/usePinpoint.ts`, `utils/blockTargeting.ts`, `utils/inputMethod.ts`, `components/sidebar/SidebarContainer.tsx`, `tests/UI-TESTING.md`.
- GitHub API: repo metadata, release list, commit counts.
- npm registry: `@plannotator/ui`, `@plannotator/core`.
- WebFetch: `docs.plannotator.ai` (index, `llms.txt`, `open-source/workflows/sharing`, `open-source/reference/local-api`), `plannotator.ai/workspaces`.

**Facts about the project at time of writing (verified):**

| Fact | Value |
|---|---|
| Stars | 7,407 |
| Open issues | 129 |
| Latest GitHub release | `v0.25.0`, 2026-07-27 |
| Releases in the 10 weeks to 2026-07-27 | 25 (`v0.19.20` → `v0.25.0`) — averaging ~2.5/week |
| Commits in the 30 days to 2026-07-29 | ≥100 (API page cap hit) |
| Test files | 224 (`*.test.ts` / `*.test.tsx`) |
| App + package LOC (ts/tsx/js/css, incl. tests) | ~201,000 |
| Toolchain | Bun + TypeScript + React 19 + Tailwind v4 + Vite; ships as a single compiled Bun binary |

Everything below is tagged **VERIFIED** (I read the code or the doc) or **UNVERIFIED** (inference, clearly reasoned but not observed).

</scope-and-method>

<1-architecture>

<what-runs>

**VERIFIED.** Plannotator is a **local CLI binary that boots an ephemeral Bun HTTP server and opens your browser at it.** There is no persistent service, no daemon, no database.

The flow, from `apps/portal/ANNOTATE.md` and `packages/server/annotate.ts`:

```
plannotator annotate README.md
  -> CLI reads the file to a string
  -> startAnnotateServer() — Bun.serve() on a random port (127.0.0.1)
  -> opens the browser at that port
  -> browser loads a single-file HTML bundle (the React SPA)
  -> SPA GETs /api/plan  -> { plan: markdown, mode: "annotate", ... }
  -> user annotates
  -> SPA POSTs /api/feedback with an exported Markdown string
  -> server resolves the promise the CLI is awaiting, prints to stdout, EXITS
```

The server lives exactly as long as one review. `README.md` line 334 makes the consequence explicit:

> "Settings are saved in cookies (not localStorage) because each hook invocation runs on a random port."

**Is the server stateless?** In the HTTP sense, effectively yes — but it is *not* a general-purpose server. It is a one-shot RPC rendezvous: the process holds an unresolved promise, the browser resolves it, the process dies. It cannot serve two reviews, cannot be restarted against the same review, and has no concept of a user or a session token. Persistent artefacts (plan history, drafts, archive, `config.json`) are written to `~/.plannotator` on the machine running the agent (`PLANNOTATOR_DATA_DIR`).

</what-runs>

<monorepo-layout>

**VERIFIED.** Bun workspaces: `"workspaces": ["apps/*", "packages/*"]` (`package.json:19`).

`apps/` = deliverables and per-agent integration shims. `packages/` = the actual product.

| Path | LOC | Role |
|---|---|---|
| `packages/ui` | 64,463 | **The web UI.** Markdown viewer, annotation engine, comment popover, panels, theme. Published to npm. |
| `packages/review-editor` | 32,646 | Code-review (diff) app UI. Irrelevant to us. |
| `packages/shared` | 25,915 | Node/server-side logic: git, jj, p4, GitHub/GitLab PR fetch, prompts. Private to the monorepo. |
| `packages/server` | 24,433 | **The Bun HTTP server** and all `/api/*` handlers. |
| `packages/editor` | 11,250 | The plan/document review **application shell** (`App.tsx`, 5,015 lines) that composes `packages/ui`. |
| `packages/ai` | 6,136 | Ask-AI / review-agent runtime. |
| `packages/core` | 3,136 | **Zero-dependency, browser-safe** utils + types (`Annotation`, compress, crypto). Published to npm. |
| `apps/hook` | 7,745 | CLI entry point + the Claude Code plugin manifest. Builds to the `plannotator` binary. |
| `apps/portal` | 66 | The **share portal SPA** — a 16-line `index.tsx` that mounts `@plannotator/editor`. |
| `apps/paste-service` | 365 | Encrypted blob store (Bun or Cloudflare Worker target). |
| `apps/review`, `apps/marketing`, `apps/vscode-extension`, `apps/{opencode,pi,droid,amp}-*`, `apps/{codex,copilot,gemini,kiro-cli}` | — | Build targets and per-agent glue. |

**Ownership answers:**
- **Web UI** → `packages/ui` (components) + `packages/editor` (app shell).
- **Annotation model** → `packages/ui/types.ts` (the `Annotation` interface) and `packages/core/types`.
- **Agent integration** → `apps/hook/server/*` (CLI + hook parsing) and `apps/{agent}/*` (per-harness manifests).

</monorepo-layout>

<the-buried-lede>

**VERIFIED, and this changes the whole analysis.** `packages/ui` and `packages/core` are **published npm packages deliberately engineered to be embedded in someone else's hosted web app.**

npm registry, checked 2026-07-29:
- `@plannotator/ui` — latest `0.28.0`, published by `backnotprop`.
- `@plannotator/core` — latest `0.22.0`.

`packages/ui/HANDOFF.md:1-14` states the design intent outright:

> "This document is for the team building the commercial **Workspaces** app. It explains ... exactly how Workspaces plugs its own backend (storage, auth, realtime, AI) into the same document UI that Plannotator uses — **without forking or rebuilding it**."
>
> "Every place the UI talks to a backend is an **optional seam**. Each seam has a default that reproduces today's Plannotator behavior (hitting `/api/*` over fetch). If Workspaces passes its own implementation, the UI uses that instead."

The front door is one call, `configurePlannotatorUI({...})` (`packages/ui/configure.ts`), and the seam catalog (`HANDOFF.md:82-95`) is exactly the list of things a cloud host needs to own:

| Seam | What it controls | Default |
|---|---|---|
| `storageBackend` | Where UI settings persist | Cookies |
| `identityProvider` | Who the user is (stamps `author`) | Generated name |
| `imageSrcResolver` | Path/ref → loadable URL | `/api/image?path=…` |
| `uploadTransport` | Where pasted images upload | `POST /api/upload` |
| `docPreviewFetcher` | Linked-doc hover preview | `GET /api/doc` |
| `fileTreeBackend` | File browser + live watch | `GET /api/reference/files` |
| `draftTransport` | Auto-saved annotation drafts | `GET/POST/DELETE /api/draft` |
| `externalAnnotationTransport` | Live/agent comments streamed in | SSE + polling |
| `aiTransport` | Ask-AI | `POST /api/ai/*` |
| `serverSync` | Push settings to server | Local no-op |

`HANDOFF.md:181-206` publishes an **explicit allowlist** of imports safe for a host, and `HANDOFF.md:210-226` an explicit denylist of components that hardcode Plannotator's local endpoints. `components/Viewer` (the full annotatable document), `components/CommentPopover`, `components/AnnotationPanel`, and `utils/parser` (`exportAnnotations`) are all on the **supported** list. `utils/sharing` is on the **unsupported** list.

`HANDOFF.md:421-428` — "The law" — commits the maintainer to the seam pattern:

> "1. Don't reimplement the document UI from scratch. Add a seam; don't rebuild.
> 2. Every seam's default must reproduce today's Plannotator behavior."

This means there is a **third option** the brief did not list — not "adopt as-is", not "fork the monorepo", but **`npm install @plannotator/ui @plannotator/core` and build a small cloud app around it.** See `<the-third-option>` below.

</the-buried-lede>

</1-architecture>

<2-remote-and-hosted-operation>

<2a-what-share-mode-actually-is>

**VERIFIED.** Share mode is **not an upload to a SaaS with accounts**. It is a **stateless link-encoding scheme with an optional zero-knowledge blob cache.** Two paths:

**Path 1 — small documents: the URL *is* the database.**
`packages/ui/utils/sharing.ts:162-183`:

```ts
export async function generateShareUrl(markdown, annotations, globalAttachments, baseUrl = DEFAULT_SHARE_BASE, rawHtml) {
  const payload: SharePayload = { p: markdown, a: toShareable(annotations), ... };
  const hash = await compress(payload);
  return `${baseUrl}/#${hash}`;
}
```

`compress()` is deflate-raw + base64url (`packages/core/compress.ts`). The whole document plus every annotation is in the URL fragment. **A fragment is never sent to a server.** `share.plannotator.ai` serves a static SPA that reads `window.location.hash` and renders. Data path: your machine → clipboard → recipient's browser. Zero server involvement.

**Path 2 — large documents: an encrypted blob + key-in-fragment.**
`packages/ui/utils/sharing.ts:239-303`:

```ts
const compressed = await compress(payload);
const { ciphertext, key } = await encrypt(compressed);   // AES-256-GCM
const response = await fetch(`${pasteApi}/api/paste`, { method: 'POST', body: JSON.stringify({ data: ciphertext }) });
const shortUrl = `${shareBase}/p/${result.id}#key=${key}${pasteParam}`;
```

The ciphertext goes to the paste service; **the key lives in the fragment and never reaches the server.** The PrivateBin model, correctly implemented. Defaults (`sharing.ts:220-221`):

```ts
const DEFAULT_PASTE_API = 'https://plannotator-paste.plannotator.workers.dev';
const DEFAULT_SHARE_BASE = 'https://share.plannotator.ai';
```

So the *default* is backnotprop's Cloudflare Worker — but it only ever holds opaque ciphertext, auto-deleted after 7 days (`apps/paste-service/core/handler.ts:13`, `ttlSeconds: 7 * 24 * 60 * 60`), 5 MB cap.

**The return trip is manual.** `packages/ui/components/ImportModal.tsx` — "Import Teammate Review" — is a **text box you paste a URL into**. The reviewer annotates in the portal, regenerates a share URL that now includes their annotations, sends it back by some out-of-band channel, and the owner pastes it in. There is no push, no polling, no server.

**One important exception — the `?cb=` callback.** `packages/ui/utils/callback.ts` implements an automated return path:

```ts
/** Callback config parsed from ?cb=<encoded_url>&ct=<token> in the URL. */
export async function executeCallback(action, config, annotatedUrl) {
  const res = await fetch(config.callbackUrl, {
    method: "POST",
    body: JSON.stringify({ action, token: config.token, annotated_url: annotatedUrl }),
  });
}
```

If a share URL carries `?cb=<https-url>&ct=<token>`, the portal renders Approve / Send Feedback buttons that **POST the annotated share URL to that callback**. Consumed at `packages/editor/App.tsx:3493-3519`. The `cb` scheme is validated to `http:`/`https:` only (`callback.ts:53-58`).

**Nothing in the OSS repo generates or receives these callbacks** (verified by grep — the only non-test consumers are `App.tsx` and the module's own tests). It exists for an unshipped bot. **This is nonetheless the exact hook our use case needs**, and it is ~100 lines of already-written, already-tested client code. See `<verdict>`.

**Images do NOT survive sharing.** VERIFIED and important, because pasted images are a hard requirement. `ImageAttachment` is `{ path: string; name: string }` (`packages/ui/types.ts:23-26`) — a filesystem path, not bytes. Uploads land in `os.tmpdir()/plannotator/` (`packages/server/image.ts:18`) and are served by `GET /api/image?path=…` (`packages/ui/components/ImageThumbnail.tsx:13`). `SharePayload` carries only those paths (`sharing.ts:24-32`). The static share portal has no `/api/image`. **A pasted image in a shared review renders as a broken link on the recipient's device.**

</2a-what-share-mode-actually-is>

<2b-what-is-actually-self-hostable>

**VERIFIED.** Two small, genuinely independent pieces — **not the app**.

**1. The share portal** (`apps/portal/SELF-HOSTING.md`, quoted in full at the top):

> "The share portal is a static single-page application. It has **no backend, no database, and makes no network requests**. All plan data is encoded in the URL hash."
>
> Build: `bun run build:portal` → `apps/portal/dist/`. Deploy: "Upload the `dist/` folder to any static hosting provider." Nginx / S3+CloudFront / Vercel / Netlify / Cloudflare Pages recipes given. Point Plannotator at it with `PLANNOTATOR_SHARE_URL`.

Deployment shape: **static files. No container, no DB, no object storage, one env var.** (Note: `apps/portal/index.tsx` is 16 lines mounting `@plannotator/editor` — the portal is literally the same React app as the local review UI, hydrated from a URL hash instead of `/api/plan`.)

**2. The paste service** (`apps/paste-service/`, 365 LOC). Two targets:

- **Bun** (`targets/bun.ts`): a ~25-line `Bun.serve()`. Env: `PASTE_PORT` (default 19433), `PASTE_DATA_DIR` (default `~/.plannotator/pastes`), `PASTE_TTL_DAYS` (7), `PASTE_MAX_SIZE` (5 MB), `PASTE_ALLOWED_ORIGINS`. Storage = **flat files** (`stores/fs.ts`).
- **Cloudflare Worker** (`targets/cloudflare.ts` + `wrangler.toml`): storage = **KV namespace**. Env var `ALLOWED_ORIGINS`.

The entire API is two routes (`core/handler.ts:102-147`): `POST /api/paste` → `{id}`, `GET /api/paste/:id` → `{data}`. It stores opaque ciphertext and has **no auth** — security rests entirely on the 8-character (~47.6-bit) random id plus the fact that the contents are encrypted with a key the server never sees. Docs also mention prebuilt paste-service binaries for macOS/Linux/Windows.

**What is NOT self-hostable: the review app itself.** There is no server-mode, no Dockerfile for the review server, no multi-session server, no auth layer. `README.md` calls Plannotator "a **local**, browser-based review surface" (line 40) and the docs index calls it "a **local** open-source tool". That framing is accurate.

</2b-what-is-actually-self-hostable>

<2c-can-a-phone-on-mobile-data-open-a-review>

**VERIFIED. Yes for *reading and annotating a snapshot*. No for a live, agent-connected review.**

**What works today, unmodified:** the agent (or you) generates a share link; you open it on your phone over mobile data; `share.plannotator.ai` is a public static site; the document and annotations decode from the fragment client-side; the same annotation UI loads. You annotate, then export a new link and get it back to the machine yourself.

**What guards it: nothing but the link.** There is **no auth, no account, no expiry on hash links.** Anyone with the URL has the document. For the paste path, secrecy = 47.6 bits of id entropy + AES-256-GCM with the key in the fragment. That is a reasonable capability-URL model, but it means **the link is the credential** — a URL in a chat log, a browser sync, or a screenshot is a full disclosure. For a solo user reviewing his own plans this is probably acceptable; it is not an access-control system.

**What does not work:**
1. **No live connection.** The phone talks to a static site. The agent's local server is not involved and is unreachable.
2. **The round trip is manual copy-paste** unless someone builds the `?cb=` receiver.
3. **Pasted images break** (see 2a).
4. **Payload ceiling.** Hash links are practical to about 2 KB of Markdown per the sharing docs; beyond that you need the paste service (5 MB cap).

</2c-can-a-phone-on-mobile-data-open-a-review>

<2d-plannotator-remote-is-about-the-agent-not-the-reviewer>

**VERIFIED, unambiguously. These are being conflated in the README's presentation, and they are different problems.**

`packages/server/remote.ts` is 152 lines and does exactly two things:

```ts
// remote.ts:46-58
export function isRemoteSession(): boolean {
  const remoteOverride = getRemoteOverride();      // PLANNOTATOR_REMOTE=1/0
  if (remoteOverride !== null) return remoteOverride;
  if (process.env.SSH_TTY || process.env.SSH_CONNECTION) return true;   // legacy auto-detect
  return false;
}

// remote.ts:82-87 — pick a FIXED port instead of a random one, so it can be forwarded
return { ports: [isRemoteSession() ? DEFAULT_REMOTE_PORT : 0], isRange: false };  // 19432

// remote.ts:145-151
/** Bind local sessions to loopback, but keep remote sessions reachable via the
 *  container or host network interface for SSH/devcontainer/Docker forwarding. */
export function getServerHostname(): string {
  return isRemoteSession() ? "0.0.0.0" : LOOPBACK_HOST;
}
```

That is the entire feature: **pick a predictable port and bind to all interfaces so an SSH tunnel or devcontainer port-forward can reach it.** The `LocalForward 9999 localhost:9999` recipe in the README is the other half.

**It is 100% about the *agent* running somewhere else (SSH box, devcontainer, Docker) while the *reviewer* sits at the desktop that owns the browser.** It does nothing whatsoever for a reviewer who is remote from the agent. The mental model is inverted from ours.

The one hedge for genuinely-remote reviewers is `packages/server/share-url.ts:105-119`, which in remote sessions prints a share link to stderr:

```
Open this link on your local machine to review:
https://share.plannotator.ai/#<hash>
(12.3 KB — plan, annotations added in browser)
```

Note the wording: *"on your local machine"*. Even the escape hatch assumes you are going back to your desktop.

</2d-plannotator-remote-is-about-the-agent-not-the-reviewer>

<2e-security-the-thing-that-actually-blocks-hosting>

**VERIFIED, and this is the single most important negative finding.**

**The Plannotator server has no authentication of any kind.** I grepped `packages/server/**` and `apps/hook/server/**` for `authorization`, `bearer`, `token`, `requireAuth` — the only hits are a *log-redaction regex* in `marker-review.ts:563` that scrubs secrets out of review output. There is no auth middleware, no session token, no `Origin` check, no CSRF protection.

The docs confirm it in writing (`docs.plannotator.ai/open-source/reference/local-api`):

> "The API is **session-scoped, unauthenticated**, and not a versioned public web API."
> "Local mode binds to `127.0.0.1`; remote mode binds to `0.0.0.0`."

And the surface it exposes is not read-only. From `packages/server/annotate.ts` alone:

| Endpoint | Effect |
|---|---|
| `POST /api/source/save` | **writes files** |
| `POST /api/upload` | writes files to tmp |
| `POST /api/open-in` | **launches a local application** (editor/Obsidian/Bear) |
| `GET /api/image?path=` | reads a path off disk |
| `GET /api/doc`, `/api/reference/files` | reads/enumerates the filesystem |
| `POST /api/ai/*` | spends your model tokens |
| `POST /api/config` | rewrites settings |

Elsewhere in `packages/server` there is an **agent terminal** runtime (`agent-terminal.ts`, `agent-terminal-runtime.ts`) and agent-job launching (`agent-jobs.ts`).

**Conclusion: exposing this port to the public internet would be reckless.** `PLANNOTATOR_REMOTE=1` binding `0.0.0.0` is safe *only* because the intended topology is a private devcontainer network or an SSH tunnel. Putting a reverse proxy and a public DNS name in front of it — the obvious naive route to "review from my phone" — hands an unauthenticated stranger file-read, file-write and app-launch on the dev machine. **This is not a bug in Plannotator; it is a design boundary we would have to build outside of.** (UNVERIFIED: I did not attempt an exploit, and I have not audited every handler for path traversal — the point stands regardless, since the endpoints are documented as unauthenticated by design.)

</2e-security-the-thing-that-actually-blocks-hosting>

</2-remote-and-hosted-operation>

<3-mobile>

**VERIFIED: there is real, deliberate mobile code — but it is layout work plus one 17-line touch bridge, with zero automated coverage and no evidence of real-device testing.** The honest verdict is "usable enough to demo, rough enough to annoy daily."

<3a-what-exists-and-works>

- **Viewport meta is present** on every app shell: `apps/hook/index.html:5`, `apps/portal/index.html:5`, `apps/review/index.html:5` — `<meta name="viewport" content="width=device-width, initial-scale=1.0" />`.
- **A real breakpoint hook**, `packages/ui/hooks/useIsMobile.ts` — `matchMedia('(max-width: 767px)')`.
- **The annotation panel becomes a proper full-screen sheet with backdrop and close button** below 768px (`packages/ui/components/AnnotationPanel.tsx:135-138, 293-303`): `fixed top-12 bottom-0 right-0 z-[60] w-full max-w-sm` plus `<div className="fixed inset-0 z-[59] bg-background/60 backdrop-blur-sm" onClick={onClose} />`.
- **Tapping a highlight auto-opens that panel** on mobile (`packages/editor/App.tsx:3163, 3195`).
- **Header labels collapse** to icons / short labels below `md` (`ToolbarButtons.tsx:99-104`, `AppHeader.tsx:208-211, 237`).
- **`CommentPopover` is horizontally clamped** (`CommentPopover.tsx:72, 79`): `Math.min(MAX_POPOVER_WIDTH, window.innerWidth - 32)` and `Math.max(16, Math.min(left, window.innerWidth - width - 16))`. At 390px it renders 358px wide, pinned.
- **A touch-selection bridge exists** (`packages/ui/hooks/useAnnotationHighlighter.ts:974-991`): on `(pointer: coarse)`, a debounced `selectionchange` listener fires 400ms after the selection settles and converts it into a highlight.
- **"Pinpoint" is a genuine tap-to-annotate mode** (`packages/ui/hooks/usePinpoint.ts:108-113, 138-179`) — a real `touchstart` handler plus a capture-phase `click` that resolves the semantic element under the finger and annotates the whole block. **No native selection is ever created, so no callout and no drag handles.** Granularity is block/inline (paragraph, list item, table cell), not word-level. This is the right primitive for a phone.
- **The annotation toolstrip is touch-aware** (`AnnotationToolstrip.tsx:285, 297-301`) — labels stay expanded on touch instead of being hover-revealed, and the container `flex-wrap`s.
- **The Settings modal adapts** (`Settings.tsx:1056-1058`) with a horizontally scrolling tab bar below `md`.

</3a-what-exists-and-works>

<3b-what-is-broken-or-missing>

- **Pinpoint is not the default on touch and is nearly undiscoverable.** `packages/ui/utils/inputMethod.ts:5` — `const DEFAULT_METHOD: InputMethod = 'drag';`. Nothing anywhere switches on `pointer: coarse`. The only auto-switch (`useInputMethodSwitch.ts`) is an **Alt-key hold** — keyboard-only, meaningless on a phone. To reach the good mode a phone user must find and hit a **28×28px** toggle (`AnnotationToolstrip.tsx:263`, `ICON_SIZE = 28`). **This is a one-line fix and the single highest-leverage mobile change available.**
- **iOS native selection callout is likely NOT suppressed.** The only mitigation is a `contextmenu` preventDefault (`Viewer.tsx:502-513`), which handles Android Chrome long-press but is **not** the mechanism behind iOS Safari's post-selection "Copy | Look Up | Share" bar. Repo-wide greps: **`-webkit-touch-callout` — 0 occurrences. `user-select` in the markdown viewer — 0.** Meanwhile the `selection` branch of the create handler is the only one that does *not* call `removeAllRanges()`, so on iOS the native selection UI is still on screen when the annotation toolbar portals in — at `rect.top - 48` (`AnnotationToolbar.tsx:102`), which is roughly where iOS parks its callout. The v0.13.0 release notes *claim* iOS is handled; the shipped code does not implement it. (UNVERIFIED on a real device — this is a code-level collision, not an observed one.)
- **`AnnotationToolbar` is not viewport-clamped at all** (`AnnotationToolbar.tsx:91-121`): `setPosition({ top: rect.top - 48, left: rect.left + rect.width / 2 })` with no `Math.max`, no horizontal clamp, no flip-below. Near the top of the scroll viewport it goes off-screen; on a 390px screen a ~200px toolbar centred on a short line fragment overflows the edge.
- **`CommentPopover` is not vertically clamped** (`CommentPopover.tsx:70-76`). It flips above when `spaceBelow < 280` — which on a phone with the keyboard up is the *normal* case — and nothing keeps `top ≥ 0`. There is a `mode: 'dialog'` fallback (`:265-360`) that is a proper centred modal and would be perfect at 390px, **but it only engages when the user taps an Expand icon.** There is no `useIsMobile`/`matchMedia`/`innerWidth` check anywhere in the file.
- **The on-screen keyboard is entirely unaccounted for.** `visualViewport` has **zero occurrences in the entire repository.** The popover repositions on `window` `resize`/`scroll` only; iOS Safari does not fire `window.resize` for the keyboard. Combined with the textarea auto-focusing on mount (`:175-183`), the likely sequence is: popover positions → keyboard opens → Save button sits behind the keyboard with no reflow. (UNVERIFIED without a device.)
- **No table of contents or navigation on a phone.** `SidebarContainer.tsx:126` is `hidden lg:flex` — invisible below **1024px**, with no drawer replacement. The marketing docs confirm this is intentional. You scroll a long document blind.
- **Tap targets are undersized** — the panel and AI toggles are `p-1.5` around a `w-4 h-4` icon = 28×28px (`AppHeader.tsx:309-343`), against 44px iOS / 48px Android guidance. The Feedback button loses its label entirely below `md` and degrades to a bare icon with a `title` tooltip, which is useless on touch.
- **A latent data-integrity hazard on touch:** the 400ms selection bridge has no mode guard, and the `redline` branch commits a DELETION immediately (`useAnnotationHighlighter.ts:852-854`). Pausing mid-drag while adjusting a selection handle in Redline mode commits a deletion against a partial range and wipes the selection.
- **Three mutually inconsistent "is this mobile?" detectors** coexist: width `< 768` (`useIsMobile`), `matchMedia('(pointer: coarse)')` (`Viewer.tsx:277`, `useAnnotationHighlighter.ts:975`), and `'ontouchstart' in window || navigator.maxTouchPoints > 0` (`AnnotationToolstrip.tsx:285`). Nothing reconciles them.

</3b-what-is-broken-or-missing>

<3c-evidence-of-testing>

**None automated.** `tests/UI-TESTING.md:222-226` is the only guidance and it is a manual DevTools device-emulation checklist — which by construction cannot reproduce the iOS callout, the native selection handles, or the keyboard/`visualViewport` behaviour, i.e. exactly the three things at risk. No test in `packages/` sets `innerWidth`, mocks `(pointer: coarse)`, or dispatches a `TouchEvent`. The `adr/` directory has **zero** mobile/touch/iOS/Android hits. Two release notes (v0.12.0, v0.13.0) describe mobile work; one of their claims is not backed by the shipped code.

**Summary judgement: mobile is a second-class citizen that someone spent one real afternoon on.** The bones are good (breakpoints, overlay panels, a tap-to-annotate mode). The last mile — defaults, clamping, keyboard, callout — is not done.

</3c-evidence-of-testing>

</3-mobile>

<4-voice>

<4a-confirmed-absent>

**VERIFIED. There is no dictation, speech, or transcription feature anywhere.** Case-insensitive grep across `packages/`, `apps/`, `docs/` for `SpeechRecognition`, `webkitSpeech`, `dictat`, `MediaRecorder`, `getUserMedia`, `microphone`, `whisper`, `transcri` returned **zero** functional hits. The only matches were:
- `packages/review-editor/components/FileRowBits.tsx:68` — a comment describing "a whisper-quiet M" (typography).
- ~15 hits on "transcript" meaning **agent session transcript** (`apps/hook/server/session-log.ts`, `codex-session.ts`) — reading Claude Code / Codex JSONL rollouts. Unrelated.

</4a-confirmed-absent>

<4b-how-hard-to-add>

**Small and localized. This is the easiest of our requirements.**

There is exactly **one** component where an annotation comment is typed: **`packages/ui/components/CommentPopover.tsx` (515 lines).** It is used by both the popover and the expanded-dialog modes, and it is on the **supported host import list** (`HANDOFF.md:194`).

The structure is already the right shape for a mic button:

- Single `<textarea>` at `:312`, with `text` / `setText` state at `:102`-ish and a `textareaRef` at `:107`.
- A focus helper `focusOnMountRef` at `:175-183` shows the idioms for driving the field programmatically.
- An **existing action-button row in the footer** (`:446-478`) already hosting Save, Ask AI, and the attachment button — the mic goes here with no layout invention.
- **`AttachmentsButton` is the precedent**: it routes uploads through the `uploadTransport` seam (`HANDOFF.md:199`). A `transcriptionTransport` seam would be the same pattern, and adding a seam is explicitly the sanctioned way to extend this package ("The law", `HANDOFF.md:425`).

Two implementation routes:
1. **Browser `SpeechRecognition`** — ~40 lines, no backend, no cost. But it is Chrome/Safari-flavoured and on iOS routes audio to Apple's servers anyway. Zero infrastructure.
2. **`MediaRecorder` → POST to a transcription endpoint** — ~80 lines client + a small server route. Better accuracy, works everywhere, costs money, requires the backend we would be building regardless.

Either way: **one file, one new button, one piece of state.** No architectural change. The mic is not the hard part — the hosting and the round-trip are.

**Caveat (UNVERIFIED):** `getUserMedia` requires a secure context. Fine on `https://` and on `localhost`, but it will **not** work over plain `http://` to a LAN IP or a forwarded port — another reason the "just expose the local server" route dead-ends.

</4b-how-hard-to-add>

</4-voice>

<5-annotation-model>

<5a-the-schema>

**VERIFIED** — `packages/ui/types.ts`.

```ts
export enum AnnotationType {
  DELETION = 'DELETION',
  COMMENT = 'COMMENT',
  GLOBAL_COMMENT = 'GLOBAL_COMMENT',
}

export interface Annotation {
  id: string;
  blockId: string;        // Legacy - not used with web-highlighter
  startOffset: number;    // Legacy
  endOffset: number;      // Legacy
  type: AnnotationType;
  text?: string;          // For comments
  originalText: string;   // The text that was selected
  createdA: number;       // (sic — the field is genuinely named `createdA`)
  author?: string;
  source?: string;        // External tool identifier (e.g. "eslint")
  images?: ImageAttachment[];
  isQuickLabel?: boolean;
  quickLabelTip?: string;
  diffContext?: 'added' | 'removed' | 'modified';
  artifact?: ArtifactAnnotationMeta;
  mathTargets?: Array<{ blockId: string; tex: string; displayMode: boolean }>;
  prUrl?: string;
  startMeta?: AnnotationTextMeta;   // web-highlighter DOM anchors
  endMeta?: AnnotationTextMeta;
}

export interface AnnotationTextMeta {
  parentTagName: string;
  parentIndex: number;
  textOffset: number;
}
```

Only **three** document annotation types. "Quick labels" and "Looks good 👍" are not separate types — they are `COMMENT` with `isQuickLabel: true`. (Code review has a richer, separate `CodeAnnotation` type with `type: 'comment'|'suggestion'|'concern'` and Conventional-Comments labels — not relevant to document review.)

`Block` (`types.ts:89-103`) is the parsed-document unit: `{ id, type: 'paragraph'|'heading'|'blockquote'|'list-item'|'code'|'hr'|'table'|'html'|'directive'|'math', content, order, startLine, ... }`. Note `startLine` — this is what lets the export cite source line numbers.

</5a-the-schema>

<5b-block-ids-and-anchoring-across-revisions>

**VERIFIED, and this is a genuine weakness.** `blockId` is explicitly marked **"Legacy - not used with web-highlighter"** (`types.ts:62`). Anchoring is really `originalText` + `startMeta`/`endMeta`.

`HANDOFF.md:257` is admirably blunt about what those anchors are:

> "`startMeta`/`endMeta` are **web-highlighter's DOM anchors**, captured against the *rendered* document ... They are **positional, not content-addressed** — they encode 'the 14th `P`, character 32', not 'this sentence'."

Reattachment order (`HANDOFF.md:259-264`, in `useAnnotationHighlighter`):
1. Math targets by `blockId` + exact TeX.
2. `highlighter.fromStore(startMeta, endMeta, originalText, id)` — works only if the rendered DOM has the same shape.
3. **Text-search fallback** — exact whitespace-normalised search for `originalText`. **Finds the first occurrence** — if the text repeats, the highlight lands on the wrong one.
4. Failure → `console.warn`, **no highlight**. The annotation survives in the panel and in the export, just unanchored.

And the verdict (`HANDOFF.md:266`):

> "anchors survive re-renders of the *same* markdown. **Once the document body is edited, the anchors are best-effort** — `originalText` is the real recovery key ... If you build 'comments follow the text through edits' on top of this ..., plan to re-anchor server-side ...; **don't expect these DOM anchors to do it.**"

Plus an honesty note (`:268`) that the failure path "is **not covered by automated tests**."

There *is* an opt-in mitigation added in 0.24.0 (`HANDOFF.md:332`): `useAnnotationHighlighter({ verifyRestoredContent: true, onRestoreMismatch })` re-anchors by text search when a positional restore lands on the wrong text. **Default off.**

**Implication for us:** the "agent revises the document, annotations still point at the right paragraphs" workflow is **not solved by this library.** If our loop is publish → annotate → agent revises → re-publish, we own re-anchoring. Practically this is fine if each revision starts a fresh review (which is Plannotator's own model — it shows a *plan diff* between revisions rather than migrating annotations).

</5b-block-ids-and-anchoring-across-revisions>

<5c-the-format-handed-back-to-the-agent>

**VERIFIED — the earlier finding is correct. It is human-readable Markdown prose, not JSON.**

`packages/ui/utils/parser.ts:775-880`, `exportAnnotations()` builds a string:

```ts
let output = `# ${title}\n\n`;
output += `I've reviewed this ${subject} and have ${annotations.length} piece${...} of feedback:\n\n`;
sortedAnns.forEach((ann, index) => {
  output += `## ${index + 1}. `;
  const lineLabel = lineLabelForAnnotation(blocks, ann);   // "(lines 12–15)"
  if (lineLabel) output += `(${lineLabel}) `;
  switch (ann.type) {
    case 'DELETION':
      output += `Remove this\n\`\`\`\n${ann.originalText}\n\`\`\`\n> I don't want this in the ${subject}.\n`; break;
    case 'COMMENT':
      output += `Feedback on: "${ann.originalText}"\n> ${ann.text}\n`; break;
    case 'GLOBAL_COMMENT':
      output += `General feedback about the ${subject}\n> ${ann.text}\n`; break;
  }
  if (ann.images?.length) { output += `**Attached images:**\n`; /* - [name] `path` */ }
});
```

Producing:

```markdown
# Plan Feedback

I've reviewed this plan and have 2 pieces of feedback:

## 1. (lines 12–15) Remove this
```
the selected text
```
> I don't want this in the plan.

## 2. Feedback on: "some highlighted text"
> This needs more detail about error handling.

---
```

Notable: **source line numbers are cited** (`lineLabelForAnnotation`, `:765-773`), and a **Label Summary** section aggregates quick-label counts (`:864-877`). Images are handed over as **filesystem paths** with an instruction for the agent to `Read` them — which is exactly why images cannot cross a network boundary.

`exportAnnotations` is a **pure function on the supported host-import allowlist** (`HANDOFF.md:188`). We can call it from our own backend with no Plannotator server.

There is a separate compact wire format for sharing — `ShareableAnnotation` (`packages/ui/utils/sharing.ts:19-22`), a positional tuple `['C', originalText, text, author, images?, 1?]` — deliberately terse to keep URLs short. Deserialising it (`fromShareable`, `:90-147`) sets `blockId: ''` and relies entirely on text re-matching.

And there is a **JSON-ish programmatic path**: `GET/POST /api/external-annotations` plus an SSE stream at `/api/external-annotations/stream`, described in the docs as "the stable integration surface ... available in plan, document annotation, and code review sessions and has a tested schema", with the transport exposed as the `externalAnnotationTransport` seam (`HANDOFF.md:91, 110`). That is how an external tool pushes/pulls structured annotations.

</5c-the-format-handed-back-to-the-agent>

</5-annotation-model>

<6-claude-code-integration>

<6a-how-it-plugs-in>

**VERIFIED. A Claude Code *plugin*, whose payload is *hooks* — not MCP.**

`.claude-plugin/marketplace.json` (repo root):
```json
{ "name": "plannotator", "owner": { "name": "backnotprop" },
  "plugins": [ { "name": "plannotator", "source": "./apps/hook", "description": "Interactive Plan Review..." } ] }
```

Install (`README.md:188`): `/plugin marketplace add backnotprop/plannotator` then `/plugin install plannotator@plannotator`.

`apps/hook/hooks/hooks.json`:
```json
{ "hooks": {
    "PreToolUse":        [ { "matcher": "EnterPlanMode", "hooks": [ { "type": "command", "command": "plannotator improve-context", "timeout": 5 } ] } ],
    "PermissionRequest": [ { "matcher": "ExitPlanMode",  "hooks": [ { "type": "command", "command": "plannotator",                "timeout": 345600 } ] } ] } }
```

The `PermissionRequest`/`ExitPlanMode` hook with a **345,600-second (4-day) timeout** is the heart of it: when the agent calls `ExitPlanMode`, Claude Code blocks on the hook; the hook boots the server, opens the browser, and **waits up to four days** for you to submit. Approve → the agent proceeds. Deny → the exported Markdown feedback becomes the denial reason and the agent revises.

Slash commands / skills are installed per harness — for Claude Code at `apps/skills/claude/plannotator-{annotate,last,review}` and `apps/skills/core/...`. **No MCP server exists anywhere in the repo** (verified).

</6a-how-it-plugs-in>

<6b-where-feedback-arrives>

**VERIFIED.** Straight into the agent's turn, via stdout of the hook process. `apps/portal/ANNOTATE.md:112-123`:

> "The `/api/feedback` endpoint resolves a promise that the CLI is awaiting. The server shuts down, and the formatted feedback string is printed to stdout. When invoked via the slash command, Claude Code captures this output and the agent sees a prompt like:
> `## Markdown Annotations` / `<the exported feedback>` / `## Your task` / `Address the annotation feedback above...`"

This is a genuinely excellent design — no polling, no file watching, no context pollution. It is also **the exact thing that does not survive going remote.**

</6b-where-feedback-arrives>

<6c-does-the-integration-work-when-hosted-remotely>

**VERIFIED: no.** The integration is a **blocking local subprocess whose stdout is the return channel.** It requires:
1. The agent and the `plannotator` binary on the same machine (the hook runs a local command).
2. The reviewer's browser able to reach that process's port.
3. The reviewer to finish within the hook timeout.

If the review surface is a cloud app, (2) breaks — and there is no built-in bridge. The pieces that *could* bridge it:
- **`?cb=` callback** (`packages/ui/utils/callback.ts`) — the portal POSTs `{action, token, annotated_url}` to any https URL. Client side done; **no receiver exists in the repo.**
- **`/api/external-annotations`** — a documented, tested, structured push/pull surface (but on the local, unauthenticated server).
- **`writeRemoteShareLink`** (`packages/server/share-url.ts:105`) — already emits a share URL from a remote session.

So the raw materials are there; **the wiring is not.** Concretely, to make the hook work with a hosted reviewer you would need the local hook to (a) publish the document to your cloud service, (b) block on *your* service instead of a local browser, and (c) receive the annotations back over the network. That is a new component, not a config change.

</6c-does-the-integration-work-when-hosted-remotely>

</6-claude-code-integration>

<7-licensing-and-governance>

<7a-the-grant>

**VERIFIED. Dual Apache-2.0 OR MIT, at the recipient's option — the standard Rust-ecosystem dual licence. Maximally permissive for our purposes.**

- `README.md:428-432`: "Copyright 2025-2026 backnotprop. Dual-licensed under [Apache 2.0](LICENSE-APACHE) or [MIT](LICENSE-MIT) **at your option**. Contributions are dual-licensed under the same terms unless you explicitly state otherwise."
- `CONTRIBUTING.md` repeats it verbatim as a DCO-style inbound=outbound clause.
- `package.json:7` — `"license": "MIT OR Apache-2.0"` (SPDX). Same in `apps/opencode-plugin` and `apps/pi-extension`. `apps/vscode-extension` is `"MIT"` alone (a subset — no conflict).
- `LICENSE-MIT` — "Copyright (c) 2025 backnotprop". `LICENSE-APACHE` — standard Apache 2.0 text.

**Both files exist because it is a dual licence, not a split one.** We pick either; MIT is the simpler grant, Apache-2.0 adds an express patent grant. **We may fork, modify, host commercially, and keep changes private** under either. No copyleft, no network clause, no CLA to sign.

**One governance gap (VERIFIED):** `packages/ui/package.json` and `packages/core/package.json` carry **no `license` field**, and the npm registry metadata for `@plannotator/ui@0.28.0` and `@plannotator/core@0.22.0` shows none either. The repo-level dual licence plus `CONTRIBUTING.md` covers the source, so the legal position is clear — but it is a paperwork smell worth an upstream one-line PR if we depend on those packages.

</7a-the-grant>

<7b-is-anything-we-need-gated-behind-a-paid-tier>

**VERIFIED: no — but the roadmap is.**

Everything we would use is in the OSS repo: the annotation UI, the annotation model, the export format, the share portal, the paste service, the Claude Code plugin. `plannotator.ai/workspaces` states "Plannotator core stays free and open source, always."

**Workspaces** is the commercial layer — "a shared team layer for reviewing agent plans and code", currently **private beta with a waitlist**, **no published pricing, no tiers**. Early OSS contributors with meaningful PRs get lifetime free access.

**The strategic read:** `README.md:158` says live collaboration "arrives in Workspaces once the room beta wraps", and `room.plannotator.ai` (the free multiplayer beta) is being sunset — `README.md:144` links a "Beta is ending. Sign up for Workspaces." banner. **The hosted, multi-user, real-time capability is precisely what is being moved behind the commercial product.** That is exactly the capability we want, and it will not be open-sourced. `HANDOFF.md` exists *because* the commercial app consumes the OSS UI packages — the OSS side is the rendering layer, the SaaS side is the backend.

**This cuts both ways, and mostly in our favour:** the maintainer has a direct commercial incentive to keep `@plannotator/ui`'s host seams clean, stable, and documented, because his own paid product depends on them. That is a much better guarantee of API stability than most OSS UI libraries offer.

</7b-is-anything-we-need-gated-behind-a-paid-tier>

<7c-is-a-fork-practical>

Mixed. Cadence and churn are the problem, not the licence or the culture.

- **Culture: welcoming.** `CONTRIBUTING.md` is four lines: fork, change, PR. `HANDOFF.md:227` explicitly invites upstreaming: *"If Workspaces ever wants one of these surfaces, the path is the same as everything else: **add a seam to the module in a Plannotator PR, don't fork the component.**"* Upstreaming a mobile fix is a supported, encouraged path.
- **Cadence: brutal.** 25 releases in 10 weeks. ≥100 commits in 30 days. A fork of the monorepo would need continuous rebasing against a very fast-moving 200k-LOC tree.
- **Tests: substantial but uneven.** 224 test files, including dedicated `*.seam.test.tsx` proving each host seam defaults correctly and routes to an override — the seam contract is genuinely pinned by CI, and there is a `tsconfig.strict-consumer.json` gate that type-checks the published surface under full `strict`. But `HANDOFF.md:268` admits the anchor-failure path is untested, and there is **zero** mobile/touch/viewport coverage.
- **Churn in the parts we care about is real.** `HANDOFF.md:296-311` documents 12 breaking API changes in 0.23.0 alone (the Radix → Base UI migration). Version numbers move fast: `@plannotator/ui` is already 0.28.0 while the newest GitHub release tag is v0.25.0.

</7c-is-a-fork-practical>

</7-licensing-and-governance>

<8-fork-burden>

**Size (VERIFIED, ts/tsx/js/css, tests included):**

| Scope | LOC |
|---|---|
| Whole app+packages tree | ~201,000 |
| What a document-review product actually needs (`packages/ui` + `editor` + `core`) | ~78,800 |
| What is dead weight for us (`review-editor` + `shared` + `server` + `ai` + `marketing` + per-agent plugins) | ~122,000 |

**Toolchain (VERIFIED):** Bun (package manager, test runner, bundler, and the `--compile` target that produces the single binary), TypeScript, React 19, Tailwind v4, Vite, Base UI (`@base-ui/react@^1.6.0` as of 0.23.0), CodeMirror 6 (via `@plannotator/atomic-editor` / `markdown-editor`), `web-highlighter` for the annotation anchoring, KaTeX for math. `moduleResolution: "bundler"`, `allowImportingTsExtensions` — **the npm packages ship TypeScript source, not compiled JS** (`HANDOFF.md:51`), so a consumer needs a TS/TSX-capable bundler.

**Honest cost of maintaining a hard fork of the monorepo:**

Against ~2.5 releases/week and ≥100 commits/month, a fork diverges fast. The bill:
- **Every upstream release is a merge decision.** Take it (rebase our patches, re-test) or skip it (drift). Neither is free.
- **The parts we would touch are the parts that churn.** `packages/editor/App.tsx` is a single 5,015-line file; `packages/ui/components/` is under active redesign (Base UI migration, Vim controls, guided review, resize-handle seams — all in the last two months). Mobile fixes land right in that blast radius.
- **We would be carrying ~122k LOC of code we never run** — code review, git/jj/p4/GitButler integration, PR fetching, nine agent harnesses, a marketing site — all of it generating merge conflicts for zero benefit.
- **Test coverage does not protect our use case.** 224 test files, none of them mobile.

**Rough estimate (UNVERIFIED, judgement):** keeping a monorepo fork current would cost on the order of **half a day to a day of agent-driven merge work per week**, indefinitely, rising whenever upstream does another engine migration. For a project where the owner writes no code himself, that is a recurring tax with no ceiling.

**But the fork is very likely the wrong frame** — see below.

</8-fork-burden>

<the-third-option>

**The option that the adopt / fork / build-fresh framing misses: consume the published packages.**

```bash
npm install @plannotator/ui @plannotator/core
```

Then, per `HANDOFF.md:122-150`:

```ts
import { configurePlannotatorUI } from "@plannotator/ui/configure";
import "@plannotator/ui/styles.css";

configurePlannotatorUI({
  storageBackend,      // localStorage — already synchronous
  identityProvider,    // our auth
  imageSrcResolver,    // our object store (http(s) URLs pass through unchanged)
  uploadTransport,     // our image upload endpoint
  draftTransport,      // our draft store
  externalAnnotationTransport,  // our realtime layer
  serverSync,
  loadSettingsFromBackend: true,
});

import { Viewer } from "@plannotator/ui/components/Viewer";        // the annotatable document
import { CommentPopover } from "@plannotator/ui/components/CommentPopover";
import { AnnotationPanel } from "@plannotator/ui/components/AnnotationPanel";
import { exportAnnotations } from "@plannotator/ui/utils/parser";  // pure — the agent-facing format
```

**What we would get for free:** the entire Markdown renderer, the annotation engine and its anchoring, the comment UI with image attachment, the panel, the theme (precompiled `styles.css`, ~31 KB gzip), the plan-diff view, and the exact export format Claude Code already understands.

**What we would own (which we must own anyway for a cloud product):** auth, storage, the document lifecycle, the agent round-trip, and the deployment.

**What we would lose vs. the local app:** version history, archive, agent-terminal, Ask-AI, Obsidian/Bear export, the file browser — all on the unsupported list (`HANDOFF.md:210-226`) because they hardcode local endpoints. We want approximately none of them.

**Why this beats a fork:** we track a **versioned npm dependency with a CI-pinned contract** instead of rebasing a 200k-LOC tree. Upgrades become "bump 0.28.0 → 0.29.0, read the changelog, run the app" — and `HANDOFF.md` is, in effect, a maintained upgrade guide (every version section since 0.23.0 documents exactly what changed for consumers). Breaking changes are real (12 in 0.23.0) but they are *enumerated*, and we can simply pin and upgrade on our own schedule.

**The catch (be honest about it):** the seams are aimed at a specific first customer — the maintainer's own commercial Workspaces app. `HANDOFF.md:274` flags known rough edges (`AITransport`/`FileTreeBackend` leak raw `Response` objects). If Workspaces changes direction or stalls, the seam contract's maintenance incentive weakens. And **the seam catalog contains no mobile or voice seam** — those still need either upstream PRs or local wrapping.

</the-third-option>

<verdict>

<a-could-the-user-use-plannotator-today-from-his-phone>

**Partially — and not in a form worth adopting.**

**What genuinely works today, unmodified:** the agent can produce a share link (`generateRemoteShareUrl`, or the Export → Share button), you open it on a phone over mobile data at `share.plannotator.ai`, the document renders, and you *can* annotate it. That is more than "local" suggested.

**What blocks it, in order of severity:**

1. **The return trip is manual copy-paste.** The reviewer must regenerate a share URL and get it back to the desktop, where someone pastes it into the Import modal (`ImportModal.tsx`). The agent does not pull anything. The `?cb=` callback (`packages/ui/utils/callback.ts`) would automate this, but **no receiver exists** — that is the missing half.
2. **Pasted images do not survive.** `ImageAttachment` is a filesystem path resolved via `/api/image` (`ImageThumbnail.tsx:13`); the static portal has no such endpoint. Images render broken. This is a stated hard requirement, and it is broken by design in share mode.
3. **No voice.** Zero speech code in the repo.
4. **Mobile annotation is rough.** Tap-to-annotate ("pinpoint") exists and is good, but **defaults to off** (`inputMethod.ts:5`) and its toggle is a 28px button. The default drag-select path collides with the iOS selection callout (no `-webkit-touch-callout` anywhere), the annotation toolbar is unclamped and can render off-screen, the comment popover flips above without a top clamp, and the on-screen keyboard is entirely unhandled (`visualViewport`: 0 occurrences repo-wide).
5. **You cannot simply host the real app instead.** The server is **documented as unauthenticated** and exposes file-write, file-read, and app-launch endpoints. `PLANNOTATOR_REMOTE=1` binds `0.0.0.0` for SSH/devcontainer forwarding — putting that behind a public URL would be genuinely dangerous.
6. **`PLANNOTATOR_REMOTE` solves the wrong problem.** It is about the *agent* being remote (SSH/devcontainer), not the *reviewer*. `remote.ts` does two things: fixed port 19432, bind `0.0.0.0`. Nothing else.

**So: no.** Not as a daily workflow. As a one-off "look at this plan on my phone", yes — with broken images and manual copy-paste.

</a-could-the-user-use-plannotator-today-from-his-phone>

<b-if-we-fork-what-changes-are-needed>

Four changes. Two are trivial, one is medium, one is the actual project.

**1. Mobile input defaults and clamping — SMALL (a day or two).**
- Default `inputMethod` to `'pinpoint'` when `matchMedia('(pointer: coarse)')` — `packages/ui/utils/inputMethod.ts:5`, effectively one line, and it sidesteps the entire native-selection-callout problem by never creating a native selection.
- Auto-engage `CommentPopover`'s existing `mode: 'dialog'` on narrow viewports — `CommentPopover.tsx:100`, one line. The modal is already written and would be excellent at 390px.
- Clamp `AnnotationToolbar` to the viewport (`AnnotationToolbar.tsx:91-121`, add `Math.max`/flip-below) and clamp `CommentPopover` vertically (`:70-76`).
- Add a `visualViewport` resize listener so the composer clears the keyboard (~20 lines; currently zero occurrences repo-wide).
- Bump tap targets from 28px to 44px.

These are all **good upstream PRs**. Given the maintainer's stated "add a seam, don't fork" stance and the lifetime-free-Workspaces incentive for contributors, upstreaming is realistic and would remove them from our maintenance burden entirely.

**2. Voice dictation — SMALL (a day).** One file: `packages/ui/components/CommentPopover.tsx`. A mic button in the existing footer action row (`:446-478`) next to Save / Ask AI / attachments, wired to `SpeechRecognition` or `MediaRecorder` → our transcription endpoint. Modelled on `AttachmentsButton`'s `uploadTransport` seam. No architectural change.

**3. Images that travel — MEDIUM (a few days).** Replace path-based `ImageAttachment` with a resolvable URL. **This is already a seam**: `uploadTransport` (upload to our object store) + `imageSrcResolver` (pass-through for full URLs — `HANDOFF.md:106` confirms `http(s)` URLs pass through unchanged). No forking needed *if* we host; unavoidable rework if we stay on the static share portal.

**4. The cloud round-trip — THE ACTUAL PROJECT (weeks, not days).** A small authenticated service that: accepts a document from the agent, serves a per-review URL to the phone, persists annotations, and hands them back. Plus the agent-side glue — a hook or command that publishes and then blocks on our service instead of on a local browser. The existing `?cb=` client and the `/api/external-annotations` schema are useful prior art, and `exportAnnotations()` (a pure, supported function) gives us the agent-facing format for free.

**Note that (1), (2) and (3) are all changes to `@plannotator/ui` — a published npm package.** None of them requires forking the monorepo.

</b-if-we-fork-what-changes-are-needed>

<c-fork-or-build-fresh>

**Neither. Build a thin cloud app on top of the published `@plannotator/ui` + `@plannotator/core` packages, and upstream the mobile fixes as PRs.**

**Why not fork the monorepo.** ~201k LOC of which ~122k is code we would never run (code review, git/jj/p4/GitButler, PR fetching, nine agent harnesses, a marketing site). Upstream ships ~2.5 releases/week and ≥100 commits/month, and the parts we would patch (`packages/ui/components/`, the 5,015-line `App.tsx`) are the parts churning hardest — a Radix→Base UI migration with 12 breaking changes landed two releases ago. **For an owner who writes no code and delegates to an AI agent, a permanently-rebasing fork is the worst possible shape**: the work is unbounded, invisible, and produces no features. Every week the agent spends resolving merge conflicts is a week it is not building the product.

**Why not build fresh.** The genuinely hard, unglamorous parts are already solved and battle-tested at 7.4k stars: Markdown → annotatable blocks, DOM-range anchoring with a text-search fallback, cross-element selections, the comment/panel/quick-label UX, math and Mermaid rendering, theming — and, critically, **an export format Claude Code already knows how to consume**. Rebuilding that is months, and the result would be worse. `HANDOFF.md:423` records that this was already attempted and failed: *"a prior from-scratch reimplementation of this UI broke the app and was reverted."*

**Why the package route fits this owner specifically.** An AI agent maintaining a **versioned npm dependency with a documented, CI-pinned contract** is a fundamentally easier job than an agent maintaining a fork. Upgrades are "bump the version, read the `HANDOFF.md` section for that release, run the app" — a bounded, legible task that fails loudly and safely. Fork rebases are unbounded, and silent semantic conflicts are exactly the failure mode agents handle worst.

**The strategic risk, stated plainly.** We would be building a small, personal version of **Workspaces** — the maintainer's commercial product — on the same OSS foundation he built it on. That is entirely legal (MIT OR Apache-2.0, no copyleft, no network clause) and it aligns our incentives with his: he is contractually motivated to keep those seams stable because his paid product depends on them. But hosted multi-user review is being deliberately moved *out* of the OSS project and into Workspaces. If our needs stay solo-and-mobile, the packages will serve us well. If we ever want real-time multiplayer, we will be reimplementing what he sells — or paying for it. Given the owner is one person reviewing his own agents' plans from a phone, **solo-and-mobile is the requirement, and the package route is the right bet.**

**Recommended sequence:**
1. Spike: a bare Vite + React page that installs `@plannotator/ui`, calls `configurePlannotatorUI` with `localStorage` + a stub upload transport, renders `<Viewer>` over a hardcoded Markdown string, and calls `exportAnnotations()` on submit. **This is the cheapest possible test of the whole thesis** — it either works in an afternoon or it reveals a blocker, and either outcome is worth far more than more research.
2. Open it on a phone. Confirm the mobile findings above on real hardware — particularly the iOS callout collision and the keyboard behaviour, both of which I could only assess from code.
3. Only then build the auth + storage + agent round-trip service.
4. Upstream the mobile fixes (pinpoint default, dialog fallback, toolbar clamping, `visualViewport`) as PRs — small, self-contained, welcomed, and they permanently reduce our maintenance burden.

</c-fork-or-build-fresh>

</verdict>

<open-questions>

Things I could not settle, flagged so they are not mistaken for conclusions:

1. **Real-device mobile behaviour.** The iOS selection-callout collision, the keyboard/`visualViewport` failure, and the toolbar overflow are all reasoned from code, not observed. They need an actual iPhone. This is cheap to resolve and should be step 2 above.
2. **`web-highlighter`'s event binding.** `node_modules` was not in the extract, so I could not confirm whether it binds `mouseup` or `pointerup`. If `pointerup`, the touch path may partly work without the 400ms bridge — and the bridge would then be double-firing. This changes the severity of the Redline-mid-drag hazard in either direction.
3. **Whether `@plannotator/ui@0.28.0` actually installs and builds cleanly** in a standalone Vite app. `HANDOFF.md:51` claims a CI strict-consumer gate enforces it, and the KaTeX font setup is documented as required manual work (`HANDOFF.md:229-237`) — but I did not run `npm install`.
4. **Workspaces pricing and GA date.** Not published. Relevant only if we later want multiplayer.
5. **Path-traversal exposure of `/api/image?path=` and `/api/source/save`.** I did not audit the handlers. Moot for the recommended architecture (we never run their server), but it would matter for anyone considering exposing the local server.
6. **Missing `license` field** in `packages/ui/package.json` / `packages/core/package.json` and their npm metadata. The repo-level dual licence governs, so the legal position is sound — but a one-line upstream PR would remove the ambiguity if we take a package dependency.

</open-questions>
