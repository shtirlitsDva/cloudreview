# 03 — Stack and hosting research

<meta>
Research date: **2026-07-29**. All pricing and version claims below were fetched
from a live source on that date; anything I could not confirm is marked
**unverified**.

Environment facts established locally (not from memory):

- `dotnet --list-sdks` on this machine: 7.0.410, 8.0.129, 8.0.423, 9.0.316,
  **10.0.302** (current default).
- **.NET 10 is LTS**, GA **2025-11-11**, supported to **2028-11-14**.
  .NET 11 is at **Preview 6** (released 2026-07-14), GA scheduled **2026-11-10**
  as a 2-year STS release. → **Build on .NET 10.**
- **Markdig 1.3.2**, published **2026-06-18** (70.8M total downloads,
  BSD-2-Clause, `github.com/xoofx/markdig`).
- **HtmlSanitizer (Ganss.Xss) 9.1.973** — resolved as latest by `dotnet add package`.

Several conclusions in sections 3 and 4 come from **experiments I actually ran**
against Markdig 1.3.2 on .NET 10, not from documentation reading. Those are
labelled *[measured]*.
</meta>

<1-blazor-hosting-model>

## 1. Blazor hosting model

<the-four-modes>
In a .NET 8+ **Blazor Web App** the hosting model is no longer a project-level
choice; it is a per-component `@rendermode`. Per the
[render modes doc](https://learn.microsoft.com/en-us/aspnet/core/blazor/components/render-modes?view=aspnetcore-10.0)
(page last updated 2026-07-22, monikers now include `aspnetcore-11.0`):

| Mode | Renders | Interactive | Transport |
|---|---|---|---|
| Static Server (SSR) | Server | No | Plain HTTP |
| InteractiveServer | Server | Yes | **SignalR circuit (WebSocket)** |
| InteractiveWebAssembly | Client | Yes | Download .NET runtime + app DLLs |
| InteractiveAuto | Server, then client | Yes | Circuit first, WASM on later visits |

"Interactive Server components handle web UI events using a real-time connection
with the browser called a **circuit**. A circuit and its associated state are
created when a root Interactive Server component is rendered."
</the-four-modes>

<where-blazor-server-hurts>
For *this* app the circuit is a genuine liability, not a theoretical one:

- **Every DOM event is a network round-trip.** Clicking a paragraph, typing in
  the comment box, and every keystroke bound with `@bind`/`oninput` marshals to
  the server and waits for a render diff to come back. Microsoft's own guidance
  and the community consensus is that this "can feel sluggish"; the known
  pathological case is `oninput` binding, which has an open history of poor
  typing experience (see
  [dotnet/aspnetcore#14242](https://github.com/dotnet/aspnetcore/issues/14242),
  [#11586](https://github.com/dotnet/aspnetcore/issues/11586)). The standard
  mitigation is to debounce/throttle — i.e. write bespoke code to work around
  the framework.
- **It requires WebSockets end-to-end.** That eliminates the cheapest Azure
  hosting option outright: **Azure Static Web Apps cannot host Blazor Server at
  all** — its API surface is documented as *"Only HTTP requests are supported for
  APIs. WebSocket, for example, is not supported."* (See section 6.)
  *(Correction to a common belief I initially repeated: App Service **Free F1 on
  Linux does** support WebSockets — up to 5 concurrent, since October 2022. F1 is
  unusable here for unrelated reasons: 0 custom domains and no TLS.)*
- **Scale-to-zero is hostile to circuits.** On Azure Container Apps, an idle
  scale-to-zero kills the circuit; the user gets the "Attempting to
  reconnect…" overlay and loses in-progress UI state. For a review session where
  a human is reading for 20 minutes and then typing, this is exactly the wrong
  failure mode.
- **Server memory per user.** Circuit state is held server-side, which is fine
  for one user but is pure downside with no upside here.
</where-blazor-server-hurts>

<where-js-interop-becomes-unavoidable>
This is the decisive finding for question 1. **Every single interactive feature
this app needs is outside what Blazor can do in C#.**

1. **Clicking a paragraph.** The rendered Markdown arrives as one blob of HTML.
   In Blazor you inject it with `@((MarkupString)html)`. Blazor **cannot attach
   `@onclick` to anything inside a `MarkupString`** — it bypasses the component
   compiler entirely, so `@onclick` inside the string is inert text. This is
   long-standing and won't-fix:
   [dotnet/aspnetcore#18301](https://github.com/dotnet/aspnetcore/issues/18301),
   [#18459](https://github.com/dotnet/aspnetcore/issues/18459).
   The workaround is either (a) hand-build a `RenderFragment` tree from the
   Markdig AST — a large bespoke renderer, exactly the thing we are trying to
   avoid — or (b) JS event delegation. Both defeat the point of using Blazor.
2. **Pasting a screenshot.** Blazor's `ClipboardEventArgs` exposes *no* clipboard
   payload — no `items`, no `files`, no image data. Getting a pasted image
   requires a custom JS event via `Blazor.registerCustomEventType`, or plain JS
   reading `ClipboardEvent.clipboardData`. Confirmed by
   [Meziantou](https://www.meziantou.net/upload-files-with-drag-drop-or-paste-from-clipboard-in-blazor.htm)
   and [Code Maze](https://code-maze.com/upload-files-drag-drop-paste-blazor/).
3. **Microphone / dictation.** `SpeechRecognition` and `getUserMedia` are browser
   APIs with no .NET surface. 100% JS interop.

So the interactive core of cloudreview — click, paste, dictate — is **three JS
modules regardless of whether Blazor is used**. Blazor would sit on top adding a
circuit, a render-mode decision, a reconnection UI, and a WebSocket hosting
requirement, while contributing nothing to the parts that are actually hard.
</where-js-interop-becomes-unavoidable>

<if-blazor-anyway>
If Blazor is used anyway, the only sane configuration is:
**Blazor Web App, global Static SSR, with `InteractiveServer` applied to nothing**,
or a single small island. That is: use Razor components purely as a templating
engine and do the interactivity in JS. But at that point it is Razor Pages with
extra ceremony — which is precisely the recommendation in section 2.

`InteractiveWebAssembly` is also a poor fit: it ships a multi-MB .NET runtime to
render a document that is already HTML on the server, adds a first-load delay,
and still cannot bind events into a `MarkupString`.
</if-blazor-anyway>

**Verdict: no interactive Blazor render mode earns its keep here — every feature this app needs (paragraph click, image paste, dictation) is JS-interop-only, and `MarkupString` blocks C# event binding outright; if Blazor is used at all it must be Static SSR, which is Razor Pages with extra steps.**

</1-blazor-hosting-model>

<2-honest-alternatives>

## 2. Honest alternatives

<three-findings-that-reframe-it>
### Three findings that reframe the whole question

**(a) The "annotation library ecosystem" criterion collapses under inspection.**
I checked every candidate. The libraries that exist — `text-annotator`, Annotorious,
the Hypothesis client — are built for a **different interaction**: drag-select
arbitrary text → highlight → scholarly annotation. cloudreview's spec is "click a
button next to a paragraph". More decisively, **none of them ship a comment editor
UI**. text-annotator's own docs say it emits `createAnnotation` events and *"doesn't
persist data itself"*, with **no built-in comment popup components**.

So the comment box, the image paste, the dictation, and the storage are bespoke in
**every** option. This criterion cannot separate the frameworks — which is itself a
strong argument for the simplest stack.

**(b) Two browser platform features have quietly deleted most of the bespoke UI code.**
- **Popover API** reached **Baseline Widely Available in April 2025** (Chrome/Edge
  114+, Firefox 132+, Safari 17.4+). `<button popovertarget="c7">` +
  `<div popover id="c7">` gives top-layer rendering, light-dismiss on outside click,
  and Escape-to-close **for free**.
- **CSS anchor positioning** is at **81.67% global** (Chrome/Edge 125+, Firefox 147+,
  Safari 26.0+). Positions the popup beside its paragraph with **zero JavaScript**.

Together, "pop an inline editor next to a paragraph" — historically the fiddliest
part — is now pure HTML + CSS. No Floating UI, no React portal, no framework.

**(c) The Azure Static Web Apps premise in the brief was wrong, but the real
problem is worse.** Hybrid Next.js does *not* require Standard — the tutorial says
*"A managed backend is automatically available for every hybrid Next.js deployment
in all plans"* and walks through selecting **Free**. The actual problem is that
hybrid Next.js on SWA is **still labelled "(preview)"** with `ms.date: 2024-04-25`
— roughly **two years in preview** — and the Free plan has **no SLA**.
</three-findings-that-reframe-it>

<aspnet-htmx>
### ASP.NET Core Razor Pages + htmx

htmx stable is **2.0.10** (npm, 2026-04-21); repo 48,811★, last push 2026-07-25 —
very actively maintained. `Htmx` / `Htmx.TagHelpers` (Khalid Abuhakmeh) at
**1.12.0** (2026-03-02), MIT, 754.5K downloads; description claims htmx 1.x/2.x
support only.

Coverage of the three hard interactions:

| Need | htmx |
|---|---|
| Pop inline editor next to a paragraph | ✅ Literally the documented [click-to-edit](https://htmx.org/examples/click-to-edit/) pattern: `hx-get` + `hx-target="this"` + `hx-swap="outerHTML"` |
| Partial updates | ✅ Its whole reason to exist |
| **Image paste** | ❌ Out of scope — you write a `paste` listener (~25 LOC JS) |
| **Dictation** | ❌ Out of scope everywhere (~20 LOC JS) |

**⚠ The htmx 4.0 problem — the biggest finding against htmx.** There is no 3.x.
The npm `next` tag is **`4.0.0-beta6`** (2026-07-23), with betas monthly since May
2026. htmx 4 is not a polish release: attribute inheritance flips from **implicit to
explicit** (you must add `:inherited` to parent `hx-target`/`hx-swap`), XHR → `fetch()`,
4xx/5xx now swap by default, `hx-disable` → `hx-ignore`, and the `hx-on:click.prevent`
shorthand is replaced by a new arrow grammar.

**For an AI-agent maintainer this is the worst possible shape of churn**: the training
corpus is overwhelmingly htmx 1.x/2.x, and htmx 4 silently changes the *semantics* of
attributes that still parse fine. An agent will confidently write 2.x-shaped code that
runs and misbehaves. Mitigation: **pin `htmx.org@2.0.10` exactly**, with a comment
saying why.
</aspnet-htmx>

<nextjs-react>
### Next.js / React on Azure Static Web Apps

Next.js **16.2.12** (2026-07-25), React **19.2.8** (2026-07-21). Next.js has
**3,867 published versions** — release velocity is itself a maintenance cost.

SWA specifics: Free plan works; hybrid is **preview**; **no SLA** on Free; app size
cap **250 MB**; and **SWA CLI local emulation is unsupported for hybrid Next.js** —
so an AI agent cannot faithfully run it locally, which is a real problem for an
agent-driven workflow.

The React annotation ecosystem, examined concretely:

| Library | Version (date) | Status |
|---|---|---|
| `react-text-annotate` | 0.3.0 (**2020-04-08**) | ☠️ **Dead** — 6 years stale |
| `@recogito/recogito-js` | 1.8.4 (2023-12-16) | ☠️ **Dead** — repo `archived: true`, npm-deprecated |
| `@annotorious/annotorious` | 3.8.8 (2026-07-02) | ✅ Alive — but **images only** after the v3 rewrite |
| `@recogito/text-annotator` | 4.2.5 (2026-06-17) | ✅ Alive but tiny — 96★, **bus factor ~2**, no comment UI, no persistence |

Comment-box editors: **TipTap 3.29.2** (2026-07-28, 15.5M weekly) is the
best-maintained, and `@tiptap/extension-file-handler` is now MIT with `onPaste`/`onDrop`
— but **`@tiptap-pro/extension-comments` is 404 on public npm**: commenting is a
**paid private-registry feature**. Lexical (**0.48.0**) and Slate (**0.126.0**) are
still 0.x after years.

**Maximum ecosystem, minimum *applicable* ecosystem.**
</nextjs-react>

<sveltekit>
### SvelteKit

Svelte **5.56.8** (2026-07-24), `@sveltejs/kit` **2.70.1** (2026-07-19) — both healthy.

**The Azure story is the disqualifier.** There is **no official
`@sveltejs/adapter-azure-swa`** (404 on npm). The only option is the community
`svelte-adapter-azure-swa`: **0.22.1, published 2025-11-26 (8 months stale)**, 21 open
issues, **1,858 weekly downloads**, still 0.x, README notes the build fails under pnpm.
A single-maintainer, pre-1.0, stale adapter is exactly the dependency that strands an
agent in three years.

**Agent maintainability is the second disqualifier.** Svelte is ~31× smaller than React
by downloads (5.2M vs 162.7M weekly). Worse, **Svelte 5's runes (`$state`, `$derived`,
`$effect`) are a hard break from Svelte 4 stores**, and most of any model's Svelte corpus
is Svelte 3/4 — agents reliably regress to `export let` and `$:`. Same failure mode as
the htmx 2→4 hazard, but **already happening** rather than pending.
</sveltekit>

<existing-tools>
### Existing self-hostable tools — adapt instead of build?

The decisive column turned out to be **image paste**, which almost nothing supports.

| Tool | Version (date) | ★ | Licence | Paragraph annotation | **Paste screenshot** | API to read back |
|---|---|---|---|---|---|---|
| **Hypothesis** | Docker 20260710 | 3,171 | BSD-2 | ✅ Best in class | ❌ **No** — images must be *"already hosted on the web"* | ✅ Excellent |
| **Outline** | v1.9.2 (2026-07-21) | 39,893 | ⚠️ **BSL 1.1** (not OSS) | ✅ ProseMirror comment mark | ✅ **Yes** — paste + drop | ✅ Cleanest |
| **Docmost** | v0.95.0 (2026-07-03) | 21,168 | AGPL-3.0 | ✅ Yjs relative positions | ❌ No Image node in comment editor | ⚠️ API keys are **Enterprise-only** |
| **BookStack** | v26.05.2 (2026-07-02) | 18,947 | **MIT** | ✅ `content_ref` anchors | ❌ No | ✅ Exposes the anchor |
| **AFFiNE** | v0.27.3 (2026-07-23) | 70,895 | ⚠️ backend Enterprise-licensed | ✅ | ✅ Yes | ⚠️ Content is an opaque BlockSuite JSON blob |
| **HedgeDoc** | 1.11.1 (2026-07-24) | 7,329 | AGPL-3.0 | ❌ **No comments at all** — issue #657 open since 2021-01-05 | — | — |
| **CodiMD** | 2.6.1 (2025-10-01) | 10,118 | AGPL-3.0 | ❌ | — | — |
| **AppFlowy** | 0.13.0 | 74,492 | AGPL-3.0 | ⚠️ claimed | ? | ❌ Cloud backend has **no inline-comment API**; no release since 2025-07-04 |
| **markupmarkdown** | 1.0.0 | **12** | MIT | ✅ + MCP server | ? unverified | MCP only |

Hypothesis has the best anchoring engine in existence and is blocked outright by
"images must already be hosted elsewhere" — plus it needs Postgres + **Elasticsearch
7.10** + RabbitMQ, with no supported self-host path. `markupmarkdown` is eerily close
to cloudreview's exact spec (drag-select comments on rendered markdown, MCP server,
MIT) but has **12 stars, one contributor, created 2026-06-03, no tagged releases**.
**Watch it; don't build on it.**
</existing-tools>

<github-pr-review>
### GitHub PR review as a zero-build option — evaluated seriously

The make-or-break question was **verified empirically**. Taking a real image URL from
a public-repo comment: `https://github.com/user-attachments/assets/…` returns
**HTTP 200, image/png, 141,167 bytes, anonymously**, via a 300-second signed S3
redirect that is re-minted on each request.

So **pasting a screenshot into a GitHub comment works, and an unauthenticated agent
can fetch it back.** Reading comments is well-trodden:
`GET /repos/{owner}/{repo}/pulls/{n}/comments` gives `path`, `line`, `start_line`,
`side`, `diff_hunk`, `body`.

**Genuine downsides, all confirmed:**
- **No dictation.** The Web Speech API is *your page's* to call. On GitHub you get
  OS-level dictation (Win+H) at best. This is an explicit requirement, lost.
- **Anchoring is on the Markdown *source*, not the rendered output.** GitHub's rich-diff
  view is read-only. The reviewer comments on `*   some **bolded** text`, not on the
  rendered paragraph — a meaningful UX downgrade for prose.
- Comments land on diff lines, so each document needs its own branch + PR (one repo,
  many branches — not a repo per doc, but still ceremony).

**Gitea v1.27.1** (MIT, 57,094★) and **Forgejo v16.0.1** replicate the model, but you'd
pay for a VM + DB to get strictly less than GitHub gives free. No reason to self-host a forge.
</github-pr-review>

<vanilla-floor>
### Plain ASP.NET Core + a little vanilla JS — and the honest line count

| Piece | How | LOC |
|---|---|---|
| POST endpoint (markdown + images) | Minimal API + `IFormFileCollection` | ~30 C# |
| Render MD → HTML with stable ids | Markdig AST walk (section 3) | **~6 C#** |
| Comment button + popup by each paragraph | `popovertarget` + `popover` + CSS anchor positioning | **0 JS**, ~20 CSS |
| Image paste → upload → insert `![](url)` | `paste` listener + `clipboardData.files` + `FormData` | ~25 JS |
| Dictation | `MediaRecorder` → POST → transcribe | ~20 JS |
| Save comment | `<form method=post>` | ~5 |
| GET annotations back | Minimal API returning JSON | ~15 C# |
| Storage | Blob + Table SDK | ~40 C# |
| **Total bespoke** | | **≈ 160–200 lines** |

`clipboardData.files` on a `<textarea>` is Baseline **"widely available since July
2015"**. Popover and anchor positioning as above.

**Crucially, ~180 LOC is the floor for *every* custom-build option**, because no
library supplies the paragraph-comment UI. React/Next/SvelteKit sit *on top of* that
floor and add build config, hydration, adapter plumbing, and dependency churn. htmx
removes maybe 15 lines of glue in exchange for a library heading into a
semantics-breaking major release.
</vanilla-floor>

<scoring>
### Scoring (1–5, higher is better)

**(a)** least bespoke code · **(b)** *applicable* library ecosystem · **(c)** Azure cost ·
**(d)** long-term maintainability by an AI agent.

| Option | (a) | (b) | (c) | (d) | **Total** | Meets all hard reqs?¹ |
|---|---|---|---|---|---|---|
| **ASP.NET Core + vanilla JS** | **5** | 3 | **5** | **5** | **18** | ✅ |
| Razor Pages + htmx 2.0.x (pinned) | 5 | 3 | **5** | 3 | 16 | ✅ |
| GitHub PR review | **5** | **5** | **5** | **5** | 20² | ❌ **no dictation; source-level anchors** |
| BookStack (self-host) | 4 | 4 | 3 | 4 | 15 | ❌ no image paste |
| Outline (self-host) | 4 | 4 | 3 | 3 | 14 | ⚠️ ❌ paragraph-button UX, ❌ dictation |
| Docmost (self-host) | 4 | 4 | 3 | 3 | 14 | ❌ no image paste |
| Next.js on SWA | 2 | 2 | 4 | 2 | 10 | ✅ (with work) |
| Hypothesis (self-host) | 3 | 4 | 1 | 2 | 10 | ❌ no image paste |
| SvelteKit | 3 | 2 | 3 | **1** | 9 | ✅ (with work) |
| HedgeDoc / CodiMD | — | — | — | — | — | ❌ **no comments feature** |

¹ Paragraph-level annotation on **rendered** output + **image paste** + **dictation** +
**API to read annotations back**.
² Scores highest numerically but **fails two hard requirements** — which is why it is
not the recommendation, only the fallback worth prototyping.
</scoring>

**Verdict: the framework question is largely a red herring — no framework reduces the ~180-line floor, because the annotation-library ecosystem targets a different interaction (drag-select highlighting, not paragraph comments) and ships no comment UI at all. Choose on what still works unattended in 2029: plain ASP.NET Core + Markdig + Baseline browser APIs, with htmx 2.0.x pinned as an optional convenience. Next.js on SWA (2-year preview, no SLA, no local emulation) and SvelteKit (stale single-maintainer adapter + the Svelte 5 runes corpus break) are both worse bets. Prototype the GitHub-PR fallback for an afternoon first — if source-level anchoring and no dictation turn out to be tolerable, build nothing.**

</2-honest-alternatives>

<3-markdown-rendering>

## 3. Markdown rendering in .NET, and stable per-block ids

<library-survey>
- **Markdig 1.3.2** (2026-06-18, BSD-2-Clause, 70.8M downloads) — the only
  serious choice in .NET. CommonMark-compliant (600+ spec tests), 20+ extensions,
  and critically it exposes a **mutable AST with source positions**. This is what
  makes stable anchoring possible.
- **Markdown.ColorCode 3.0.1** — a Markdig extension that server-side
  syntax-highlights fenced code blocks to inline-styled HTML. Nice-to-have, zero
  JS, no client library. Optional.
- **Blazored.Markdown — does not exist.** `nuget.org/packages/Blazored.Markdown`
  returns **HTTP 404** and it does not appear in package search. It should be
  struck from the brief. The real Blazor-flavoured wrappers are
  `Blazorise.Markdown` (2.1.0, 2026-04-15), `MudBlazor.Markdown` (9.0.0), and
  `PSC.Blazor.Components.MarkdownEditor` (10.0.9) — all are *editor/viewer*
  components that wrap a JS editor, none of them help with per-block anchoring,
  and all of them add a UI-framework dependency. **Do not use any of them.**
- **JS side:** markdown-it 14.x, remark 15.x / unified 11.x, marked 12.x.
  `remark`+`rehype`+`rehype-sanitize` is genuinely excellent and has real
  position data (`node.position.start.line`). But adopting it means the render
  pipeline lives in Node/browser while the API lives in .NET — two ecosystems for
  one small app. **Markdig is strictly better here** because rendering happens
  once, server-side, at publish time.
</library-survey>

<the-anchoring-problem>
### The critical sub-question: stable, re-render-safe block ids

Markdig's `MarkdownObject` exposes three position members:
`Span` (`SourceSpan`, start/end char offsets), `Line` (0-based), `Column`.
`UsePreciseSourceLocation()` makes these exact. Every node also supports
attached HTML attributes via `GetAttributes()` (`Markdig.Renderers.Html`).

The working recipe is **parse → mutate AST → render**, not string post-processing:

```csharp
var pipeline = new MarkdownPipelineBuilder()
    .UseAdvancedExtensions()
    .UsePreciseSourceLocation()   // required for exact Span
    .Build();

var doc = Markdown.Parse(src, pipeline);

foreach (var b in doc.Descendants<Block>())
{
    if (b is MarkdownDocument) continue;
    var slice = src.Substring(b.Span.Start, b.Span.End - b.Span.Start + 1);
    var norm  = string.Join(" ", slice.Split((char[]?)null,
                    StringSplitOptions.RemoveEmptyEntries));   // whitespace-normalise
    var hash  = Convert.ToHexString(
                    SHA256.HashData(Encoding.UTF8.GetBytes(b.GetType().Name + "|" + norm)))
                [..10].ToLowerInvariant();
    b.GetAttributes().Id = $"b-{hash}";      // + "-{n}" ordinal for duplicates
}

var w = new StringWriter();
var renderer = new Markdig.Renderers.HtmlRenderer(w);
pipeline.Setup(renderer);      // must call Setup or extensions are lost
renderer.Render(doc);
var html = w.ToString();
```

*[measured]* Actual output from this code on Markdig 1.3.2:

```html
<p id="b-9601105348">First paragraph of the doc.</p>
<p id="b-69943eb845">Second paragraph here.</p>
<ul id="b-3e8e4e5761">
<li id="b-4f01cd84dc">item one</li>
<li id="b-8460d81a5d">item two</li>
</ul>
<pre><code id="b-2b94243df4" class="language-csharp">var x = 1;
</code></pre>
<blockquote id="b-2e013740b5">
<p id="b-3984f2b36f">a quote</p>
</blockquote>
```

Notes from the experiment:

- Ids land on `ParagraphBlock`, `ListBlock`, `ListItemBlock`, `FencedCodeBlock`,
  `QuoteBlock`. For annotation purposes iterate **top-level blocks only**
  (`foreach (var b in doc)`) rather than `Descendants<Block>()`, otherwise you get
  nested ids on both `<blockquote>` and its inner `<p>`.
- **`HtmlBlock` silently ignores the id** — the HTML renderer writes raw HTML
  blocks verbatim and does not emit attributes. So raw-HTML blocks cannot be
  annotated. This matters for the XML headings in section 4 and is another reason
  to normalise them away.
</the-anchoring-problem>

<line-vs-hash>
### Why `Block.Line` is the wrong id and a content hash is the right one

*[measured]* I rendered three versions of a 3-paragraph document and printed
`line:contenthash` for each paragraph:

| Version | Paragraph ids produced |
|---|---|
| v0 baseline | `L0:D10672`  `L2:CB47AD`  `L4:6B566B` |
| v1 — middle paragraph edited | `L0:D10672`  `L2:932AF8`  `L4:6B566B` |
| v2 — new paragraph inserted at top | `L0:B4FAFA`  `L2:D10672`  `L4:CB47AD`  `L6:6B566B` |

Read the hashes, not the line numbers:

- Editing paragraph 2 changed **only** paragraph 2's hash. Annotations on
  paragraphs 1 and 3 survive.
- Inserting at the top shifted **every** line number (`L0→L2`, `L2→L4`, `L4→L6`)
  while **every content hash was preserved**. A line-number anchor would have
  silently re-pointed all three annotations to the wrong paragraphs. A content
  hash re-attached all three correctly.

So: **id = short SHA-256 of (block type + whitespace-normalised source slice),
with a `-1`, `-2` ordinal suffix to disambiguate genuinely identical blocks.**
Persist the id with the annotation. If a document is republished and an id
disappears, mark that annotation "orphaned" rather than guessing — with
`Block.Line` you cannot even detect the problem.
</line-vs-hash>

**Verdict: Markdig 1.3.2 with `UsePreciseSourceLocation()`, assigning ids by mutating `GetAttributes().Id` on top-level blocks, keyed on a truncated SHA-256 of the whitespace-normalised source slice — measured to survive both edits and insertions, which `Block.Line` demonstrably does not. Skip Markdown.ColorCode unless code highlighting is wanted; ignore all Blazor markdown wrapper packages, and note `Blazored.Markdown` does not exist.**

</3-markdown-rendering>

<4-xml-style-headings>

## 4. XML-style headings (`<section-name>` … `</section-name>`)

<what-actually-happens>
Markdig passes raw HTML through by default — **confirmed** — but the naive
assumption that this "just works" is **wrong**, and the failure is silent and
ugly.

*[measured]* Input written the way the owner actually writes it (no blank lines):

```markdown
<section-name>
This is a paragraph with **bold**.

Second paragraph.
</section-name>
```

Markdig parses this as `HtmlBlock(Line=0, Span=0..48)` + `ParagraphBlock(Line=3)`
and emits:

```html
<section-name>
This is a paragraph with **bold**.
<p>Second paragraph.
</section-name></p>
```

Two serious defects: the first paragraph's `**bold**` **was never parsed as
Markdown** (it got swallowed into the raw HTML block), and the output is
**malformed** — `</section-name></p>` closes the tags in the wrong order.

This is correct CommonMark behaviour, not a Markdig bug: `<section-name>` is an
unknown tag, so it starts an **HTML block type 7**, which runs until the next
blank line and consumes everything in between verbatim.

*[measured]* With blank lines around the tags it behaves properly:

```markdown
<section-name>

This is a paragraph with **bold**.

Second paragraph.

</section-name>
```
→ `HtmlBlock`, `ParagraphBlock`, `ParagraphBlock`, `HtmlBlock`, rendering
correct `<p>…<strong>bold</strong>…</p>`. Nesting (`<outer>`/`<inner>`) also
works when blank-line-separated.

So the behaviour depends entirely on blank-line placement that the author is not
thinking about. Relying on it is fragile.
</what-actually-happens>

<recommended-fix>
### Normalise the tags to real headings in a pre-pass

Rather than depending on the author's blank lines, run a ~6-line regex pre-pass
that converts XML-style section tags into ATX headings before parsing. This also
solves the "HtmlBlock can't carry an id" problem from section 3, and makes the
sections real, linkable, annotatable headings.

```csharp
static string NormalizeXmlHeadings(string md)
{
    // a line containing ONLY a custom (hyphenated) open or close tag
    var open  = new Regex(@"^[ \t]*<([a-zA-Z][a-zA-Z0-9]*(?:-[a-zA-Z0-9]+)+)>[ \t]*$",  RegexOptions.Multiline);
    var close = new Regex(@"^[ \t]*</([a-zA-Z][a-zA-Z0-9]*(?:-[a-zA-Z0-9]+)+)>[ \t]*$", RegexOptions.Multiline);
    md = close.Replace(md, "");                                       // drop closers
    md = open.Replace(md, m => "## " + m.Groups[1].Value.Replace('-', ' '));
    return md;
}
```

*[measured]* Applied to the broken no-blank-line input above, with
`.UseAutoIdentifiers()` on the pipeline:

```html
<h2 id="section-name">section name</h2>
<p>This is a paragraph with <strong>bold</strong>.</p>
<p>Second paragraph.</p>
<h2 id="another-section">another section</h2>
<p>Content of another.</p>
```

Correct, well-formed, `**bold**` parsed, and headings get slug ids for free. The
regex deliberately requires a hyphen in the tag name, so it targets custom
section tags and leaves legitimate HTML (`<div>`, `<br>`, `<img>`) alone.
</recommended-fix>

<sanitisation>
### HTML sanitisation — mandatory, and it has a trap

*[measured]* Markdig's raw-HTML passthrough is a live XSS hole. Given
agent-authored input:

```html
<p>Normal <strong>text</strong>.</p>
<script>alert('pwned')</script>
<img src=x onerror="alert(1)">
<p><a href="javascript:alert(1)">link</a></p>
```

All of it renders verbatim. `<script>` executes. `onerror` executes.
`javascript:` URLs are live.

Two mitigations, both measured:

1. `.DisableHtml()` on the pipeline — escapes all HTML. But it also escapes the
   XML section tags into visible `&lt;section-name&gt;` text, so only use it
   *after* the normalisation pre-pass above.
2. **HtmlSanitizer (Ganss.Xss) 9.1.973** on the rendered output. Defaults are
   strong: `<script>` removed entirely, `onerror` stripped, `javascript:` href
   stripped (leaving a bare `<a>`), unknown tags unwrapped while keeping their
   children, relative image `src` preserved.

**The trap** *[measured]*: HtmlSanitizer's default allowlist does **not** include
`id` or `class`.

```text
id allowed by default?    False
class allowed by default? False
Sanitize("<p id=\"b-abc123\" class=\"x\" data-block-id=\"b-abc123\">hi</p>")
  → <p>hi</p>
```

Sanitising after assigning block ids **silently destroys the entire anchoring
scheme**. You must explicitly allow the attributes you rely on:

```csharp
var s = new HtmlSanitizer();
s.AllowedAttributes.Add("id");
s.AllowedAttributes.Add("class");           // for language-* on code blocks
s.AllowedAttributes.Add("data-block-id");   // measured: survives once allowed
```

Order of operations: **normalise → parse → assign ids → render → sanitise (with
`id`/`class`/`data-block-id` allowed)**. Prefer a `data-block-id` attribute over
`id` so that an author who injects raw `<p id="…">` cannot collide with the
anchor namespace.
</sanitisation>

**Verdict: Markdig does pass raw HTML through, but XML-style headings without surrounding blank lines produce unparsed Markdown and malformed HTML — normalise the tags to ATX headings with a small regex pre-pass, then sanitise the rendered output with HtmlSanitizer 9.1.973, explicitly allow-listing `id`/`class`/`data-block-id` or the anchoring silently breaks.**

</4-xml-style-headings>

<5-voice-dictation>

## 5. Voice dictation

<web-speech-api>
### Web Speech API (`SpeechRecognition`)

Support, from **MDN browser-compat-data** (`api/SpeechRecognition.json`, fetched
2026-07-29):

| Browser | Unprefixed `SpeechRecognition` | `webkitSpeechRecognition` |
|---|---|---|
| Chrome desktop | **139** | 33+ |
| Edge (Windows) | mirrors Chrome → **139** | 79+ |
| Firefox | 142, but behind `media.webspeech.recognition.enable` — **off by default** | no |
| Safari | 14.1 | 14.1 (prefixed) |
| Chrome Android | yes | yes |

The unprefixed name only arrived in **Chrome/Edge 139 (stable 2025-08-05)**, so
the defensive line is still needed:
`const SR = window.SpeechRecognition || window.webkitSpeechRecognition;`

Note: **caniuse.com is stale here** — it still lists Edge as "Not supported" and
has no Chrome 139 entry. MDN BCD and Microsoft's own Edge docs contradict it.

**Does it send audio to the cloud? Yes, by default — still true in 2026.** MDN:
*"On some browsers, like Chrome, using Speech Recognition on a web page involves
a server-based recognition engine. Your audio is sent to a web service for
recognition processing, so it won't work offline."* Edge's implementation is
cloud-backed by Azure.

**On-device recognition exists but is narrower than hoped.** It shipped in
**Chrome 139**, not 138, and the API is `SpeechRecognition.processLocally`
(instance property) plus static `SpeechRecognition.available({langs,
processLocally, quality})` and `SpeechRecognition.install(...)` — the names
`availableOnDevice()` / `installOnDevice()` never shipped. `quality: "dictation"`
needs a heavier model and can return `unavailable` on weaker hardware.
**Edge on-device is not ready**: per learn.microsoft.com (2026-06-01) it requires
**Edge Canary/Dev 150.0.4076+** behind an `edge://flags` toggle. On Edge stable,
audio goes to Microsoft's cloud.

Crucially for the owner's complaint: **on-device mode will not improve quality.**
Chrome's on-device SODA model is *smaller* than the cloud model — `processLocally`
trades accuracy for privacy, not the reverse.

**Cost:** free. But note a known **~60-second single-utterance timeout** in Chrome
with no documented extension; mitigate with `continuous = true` and an `onend`
restart or long dictations silently truncate. The "no quota" claim traces to an
older chromium-dev post and is **unverified for 2026**.

**Quality:** the owner's instinct is directionally right but I found **no credible
head-to-head WER benchmark of browser Web Speech vs Whisper/Azure** — it is absent
from Artificial Analysis and every other leaderboard checked. Percentage claims
floating around (90–95% accuracy) come from vendor-adjacent blogs and are
**unverified**. The *structural* argument is solid though: Web Speech gives you no
punctuation control, no vocabulary biasing before Chrome 142's `phrases`, and no
confidence tuning — so dictated prose reads worse even at comparable raw WER.
</web-speech-api>

<azure-ai-speech>
### Azure AI Speech

The public pricing page renders prices client-side as `$-` and is unfetchable.
Figures below come from the **Azure Retail Prices API**
(`prices.azure.com/api/retail/prices`), the authoritative billing source —
region **westeurope**, `productName eq 'Azure Speech'`:

| Meter | Price | Unit |
|---|---|---|
| **S1 Speech To Text** (standard real-time) | **$1.00** | 1 hour |
| **Fast Transcription** | **$0.36** | 1 hour |
| **S1 Speech to Text Batch** | **$0.18** | 1 hour |
| S1 Custom Speech To Text (real-time) | $1.20 | 1 hour |
| Free Speech To Text | $0.00 | 1 hour |

(The meters are named **S1** internally even though the portal SKU is **S0**
pay-as-you-go. Same thing.)

**Free tier F0:** "Real-time Transcription: 5 audio hours free per month". Per
[quotas and limits](https://learn.microsoft.com/en-us/azure/ai-services/speech-service/speech-services-quotas-and-limits)
(updated 2026-07-15), **F0 allows 1 concurrent real-time request**, and Fast
Transcription and Batch are **not available on F0 at all**. Whether the 5 hours
is shared across standard+custom is **unverified**.

**SDKs:** NuGet `Microsoft.CognitiveServices.Speech` **1.51.1** (2026-07-25);
npm `microsoft-cognitiveservices-speech-sdk` **1.51.0** (MIT).

**Keeping the key out of the browser.** Server POSTs to
`https://REGION.api.cognitive.microsoft.com/sts/v1.0/issueToken` with
`Ocp-Apim-Subscription-Key`, gets a JWT valid **10 minutes**, hands it to the
browser, which calls `SpeechConfig.fromAuthorizationToken(token, region)`.
**The classic bug:** you must refresh by setting `recognizer.authorizationToken`
on the *live recognizer* — setting it on the config only affects recognizers
created afterwards. Otherwise recognition dies after 10 minutes.

**Managed Identity** works for Speech but requires an irreversible **custom
subdomain** (regional endpoints don't support Entra auth) plus the *Cognitive
Services Speech User* role — and the docs have **no JavaScript pivot**. Practical
conclusion: use Managed Identity on the .NET backend, never Entra from the browser.

**Quality:** Artificial Analysis lists Microsoft's **MAI-Transcribe-1 at 2.6% WER**
— second only to ElevenLabs Scribe v2 (2.2%) and ahead of every OpenAI model. But
that is *not* the classic $1/hr `S1 Speech To Text` meter, and its Azure pricing
and availability could not be confirmed from a primary source — **unverified**.
</azure-ai-speech>

<openai-whisper>
### OpenAI transcription

From [developers.openai.com/api/docs/pricing](https://developers.openai.com/api/docs/pricing):

| Model | Price |
|---|---|
| **`gpt-transcribe`** (current flagship batch) | **$0.0045 / min** |
| `gpt-live-transcribe` (streaming) | $0.017 / min |
| `gpt-4o-transcribe` | $0.006 / min |
| `gpt-4o-mini-transcribe` | $0.003 / min |
| `whisper-1` | $0.006 / min |

**`whisper-1` is now strictly dominated** — `gpt-transcribe` is both cheaper and
more accurate. Do not use `whisper-1`.

WER from the [Artificial Analysis speech-to-text index](https://artificialanalysis.ai/speech-to-text)
(fetched 2026-07-29, lower is better): Scribe v2 2.2% · MAI-Transcribe-1 2.6% ·
Gemini 3.1 Pro 2.8% · **GPT Transcribe 3.3%** · GPT-4o Transcribe 4.0% ·
Whisper Large v2 4.1% · GPT-4o Mini Transcribe 4.5% · Deepgram Nova-3 5.2%.

Key handling: for short comments **no ephemeral token is needed**. Record with
`MediaRecorder`, POST the blob to a .NET endpoint, server calls
`/v1/audio/transcriptions`. The key never leaves the server. Ephemeral tokens
(`/v1/realtime/client_secrets`, default 600s, range 10–7200s) are only needed for
live streaming via WebRTC.

**Azure OpenAI transcription pricing is unverified** — every figure on the Azure
OpenAI pricing page renders as a `$-` placeholder, and the Retail Prices API
doesn't surface them. Azure OpenAI also does not appear to carry the newer
`gpt-transcribe` / `gpt-live-transcribe` models yet.
</openai-whisper>

<cost-at-realistic-volume>
### Real cost at 60 minutes of dictation per month

| Option | Monthly cost |
|---|---|
| Web Speech API | **$0.00** |
| Azure STT real-time, F0 free tier | **$0.00** (within 5 free hrs) |
| **OpenAI `gpt-transcribe`** | **$0.27** |
| Azure Fast Transcription | $0.36 |
| Azure STT real-time, S0 | $1.00 |
| OpenAI `gpt-live-transcribe` | $1.02 |

At this volume **cost is not a decision variable** — every option rounds to free.
Decide on quality and implementation effort.
</cost-at-realistic-volume>

**Verdict: OpenAI `gpt-transcribe` ($0.0045/min, 3.3% WER) via a server-side proxy — `MediaRecorder` → POST blob → `/v1/audio/transcriptions`, key never leaves the server, no token-refresh bug, ~$0.27/month. It directly answers the owner's "Google voice typing isn't very good" complaint, which Chrome's on-device mode would make *worse*, not better. Keep Web Speech API as a free live-preview fallback; if single-vendor Azure billing matters more, use Fast Transcription at $0.36/hr from the .NET backend — never the browser SDK in front of a key.**

</5-voice-dictation>

<6-azure-hosting>

## 6. Azure hosting

<pricing-method-note>
All figures fetched **2026-07-29**, USD pay-as-you-go retail. Figures marked **[API]**
come from the **Azure Retail Prices API** (`prices.azure.com/api/retail/prices`) — the
HTML pricing pages render client-side and serve `$-` placeholders to a fetcher, so the
API and the pages' embedded regional JSON are the only quotable sources. Monthly =
hourly × 730.
</pricing-method-note>

<static-web-apps>
### Azure Static Web Apps

| | Free | Standard |
|---|---|---|
| App | **$0** | **$9.00 / app / month** [API] |
| Included bandwidth | 100 GB/mo per subscription | 100 GB/mo |
| Bandwidth overage | **none — site stops being served** | $0.20/GB |
| Max app size | 250 MB | 500 MB |
| Custom domains | 2 | 5–6 |
| **Free managed TLS** | ✅ | ✅ |
| SLA | **None** | Yes |

SWA Free is in the **"Always" free column**, not the 12-month trial column. The
Dedicated plan was **retired 2025-10-31**.

**Can SWA host Blazor Server or any server-rendered .NET? NO.** Three confirmations:

> "Only HTTP requests are supported for APIs. **WebSocket, for example, is not
> supported.**" — [apis-overview](https://learn.microsoft.com/en-us/azure/static-web-apps/apis-overview)

> "Static web apps are commonly built using libraries and web frameworks like Angular,
> React, Svelte, Vue, or Blazor **where server side rendering isn't required**… or using
> **Blazor to create WebAssembly applications**, with an Azure Functions back-end."

**Blazor WASM is fully supported.** Blazor Web App with `InteractiveServer` or
`InteractiveAuto` is impossible — no circuit.

**Managed Functions runtime ceiling: .NET 9 isolated. `dotnet-isolated:10.0` is NOT
listed.** Since .NET 9 dies 2026-11-10, SWA's managed backend is a dead end for new
server-side .NET work.
</static-web-apps>

<app-service>
### Azure App Service (Linux)

| SKU | West Europe | East US |
|---|---|---|
| F1 Free | $0.00/hr | $0.00/hr |
| **B1 Basic Linux** | **$0.018/hr ≈ $13.14/mo** | **$0.017/hr ≈ $12.41/mo** |
| B2 Basic Linux | $0.036/hr ≈ $26.28/mo | ≈ $24.82/mo |

**Windows B1 is $0.075/hr ≈ $54.75/mo — 4× Linux. Use Linux.**

| | F1 (Free) | B1 (Basic) |
|---|---|---|
| CPU time/day | **60 minutes** | Unlimited |
| Bandwidth | **165 MB** | Unlimited |
| **WebSockets/instance (Linux)** | **5** | ~50,000 |
| **Always On** | **Not available** | Available |
| **Custom domains** | **0** | 500 |
| **Free managed certificate** | **No** | **Yes** |
| SLA | None | 99.95% |

**Does F1 support WebSockets? YES — the common belief that it doesn't is out of date.**
Per the [Linux App Service FAQ](https://learn.microsoft.com/en-us/troubleshoot/azure/app-service/faqs-app-service-linux-new)
(doc date 2026-01-20): *"Web Sockets are always enabled for Linux"* and *"Web Sockets are
now supported for Linux apps on Free App Service plans. We support **up to five web
socket connections**… Exceeding this limit results in an **HTTP 429**."* Changed
October 2022.

**But F1 is still not viable** — for a different reason: **0 custom domains and no
custom-domain TLS at all**, plus 60 CPU-min/day and 165 MB bandwidth.

**Cold start / idle unload**, confirmed from
[configure-common](https://learn.microsoft.com/en-us/azure/app-service/configure-common)
(2026-04-13):

> "When **Always On** is turned off (default), **the app is unloaded after 20 minutes
> without any incoming requests.**"

Always On is **off by default and available only on Basic and above** — so F1 cannot
avoid the 20-minute unload; B1 can. Free managed certificates likewise require
**Basic or above**.

For Blazor Server on B1: enable WebSockets and **session affinity (ARR)**;
*"A Blazor app on Azure App Service doesn't require Azure SignalR Service."*
</app-service>

<container-apps>
### Azure Container Apps (Consumption)

| Meter [API] | West Europe | East US |
|---|---|---|
| vCPU **active** | $0.000034 /vCPU-s | **$0.000024 /vCPU-s** |
| vCPU **idle** | $0.000004 /vCPU-s | $0.000003 |
| Memory active **and idle** | $0.000004 /GiB-s | $0.000003 |
| Requests | $0.56 / 1M | $0.40 / 1M |

Note: **memory has no idle discount** — only vCPU gets the ~8.75× idle rate.

**Free monthly grant, confirmed verbatim** from [billing](https://learn.microsoft.com/en-us/azure/container-apps/billing):

> "The following resources are free during each calendar month, **per subscription**:
> The first **180,000 vCPU-seconds** · The first **360,000 GiB-seconds** · The first
> **2 million HTTP requests**."

Consumption-only environments carry **no management fee**. Health-probe requests are
not billable.

**Scale-to-zero:** defaults `minReplicas: 0`, cool-down **300 s** → ~5 minutes after
the last request. *"When a revision is scaled to zero replicas, no resource consumption
charges are incurred."*

**Cold start:** Microsoft's own [cold-start doc](https://learn.microsoft.com/en-us/azure/container-apps/cold-start)
gives **zero numbers**. The one rigorous public benchmark (Oct 2025, DevTools-timed
after 15–60 min idle) measured **15–37 seconds** at 0.25 vCPU/512 MB for a *Quarkus
native* image versus ~140 ms with min-scale 1. A JIT .NET app will not be faster.
**Expect tens of seconds.** Widely-circulated "5–10 s for .NET" claims have no
published methodology — **unverified**.

**Min-replica cost** (0.25 vCPU / 0.5 GiB, the smallest valid combo — ACA enforces
1:2 vCPU:GiB):

| Scenario | West Europe | East US |
|---|---|---|
| `min=0`, single-user traffic | **$0** (inside grant) | **$0** |
| `min=1`, idle 730 h/mo, after grant | ≈ **$5.72/mo** | ≈ **$4.29/mo** |

**WebSockets/SignalR: supported.** Ingress gives *"Support for WebSocket and gRPC"*
plus session affinity; *"The [Azure SignalR] service isn't required for Blazor apps
hosted in Azure App Service or Azure Container Apps."* **Two mandatory steps:**
enable session affinity, and **persist ASP.NET Core Data Protection keys to Blob
Storage** — the docs state flatly *"You need to enable data protection for all .NET
apps on Azure Container Apps."* Ingress idle timeout defaults to 4 minutes (raising it
needs Premium ingress → a Dedicated workload profile → economically absurd here), but
SignalR's 15 s `KeepAliveInterval` makes that a non-issue.

**Custom domains + free managed TLS: yes**, auto-renewing.

**Unverified:** what happens to an in-flight WebSocket when ACA scales to zero — no
Microsoft doc addresses it. Certain: replica termination closes the socket and an
unrecovered Blazor circuit means a reload and total loss of component state.
</container-apps>

<functions>
### Azure Functions

| Plan | Free grant | Above grant |
|---|---|---|
| **Consumption** | **1M executions + 400,000 GB-s** | $0.20/M exec; $0.000016/GB-s |
| Flex Consumption | 250,000 exec + 100,000 GB-s | $0.40/M exec; $0.000026/GB-s |

Flex is **62% more expensive per GB-s with a 4× smaller grant** — you pay for the
sub-2-second cold start.

**Can Functions host a Blazor Server app? No.** The ASP.NET Core integration exists,
but [the docs](https://learn.microsoft.com/en-us/azure/azure-functions/dotnet-isolated-process-guide) state:

> "**This model doesn't expose all features of ASP.NET Core. Specifically, it doesn't
> provide access to the ASP.NET Core middleware pipeline and routing capabilities.**"

That is disqualifying — Blazor Server needs the full pipeline and a persistent
`/_blazor` hub. Functions fits only as an **API behind a Blazor WASM frontend**.
Two retirements: the **in-process model dies 2026-11-10**; **runtime 1.x dies
2026-09-14**.
</functions>

<dotnet-10-availability>
### .NET 10 runtime availability

| Version | Type | GA | End of support |
|---|---|---|---|
| **.NET 10** | **LTS** | 2025-11-11 | **2028-11-14** |
| .NET 9 | STS | 2024-11-12 | **2026-11-10** |
| .NET 8 | LTS | 2023-11-14 | **2026-11-10** |

**Both .NET 9 and .NET 8 die 2026-11-10 — about 3.5 months from now. Do not start on
either.**

- **Container Apps: yes, trivially** — `mcr.microsoft.com/dotnet/aspnet:10.0` verified
  present in MCR (plus `10.0-alpine`, `10.0-alpine-aot`). No version gate; it's a container.
- **App Service Linux: yes** (`DOTNETCORE|10.0`), preview announced 2025-08-26.
  **Unverified:** whether the portal still labels it "(Preview)" as of 2026-07. Escape
  hatch: self-contained deployment or a custom container, both of which bypass the
  platform runtime.
- **Azure Functions:** .NET 10 isolated only, and **not on Linux Consumption** — Flex required.
- **SWA managed Functions: .NET 10 NOT available** (ceiling `dotnet-isolated:9.0`).
</dotnet-10-availability>

<hosting-ranking>
### Cheapest for near-zero traffic

| Rank | Option | ~$/mo (WE) | Notes |
|---|---|---|---|
| **1** | **SWA Free + Blazor WASM** | **$0** | Free custom domains + TLS, global CDN, no cold start. Capped at 250 MB; managed Functions stuck on .NET 9. |
| **2** | **ACA Consumption, `min=1`, 0.25 vCPU** | **$0–5.72** | Often $0 inside the grant ($4.29 in East US). Free TLS, .NET 10, no runtime gate. Needs session affinity + Data Protection to Blob. |
| 3 | ACA Consumption, `min=0` | ~$0 | Cheapest on paper; tens of seconds of cold start. Bad trade for interactive UI. |
| 4 | **App Service B1 Linux** | **$13.14** | The boring, low-risk choice: Always On, ~50k WebSockets, 99.95% SLA, free cert, no Data Protection wiring at one instance. |
| 5 | SWA Standard | $9 | Only for bring-your-own backend / SLA / >2 domains. |
| — | App Service F1 | $0 | **Not viable** — 0 custom domains, no TLS, 60 CPU-min/day. |
| — | Azure Functions | $0 grant | API only; **cannot host Blazor Server**. |

**Region:** compute favours **East US** (ACA active vCPU 42% cheaper, requests 29%
cheaper, B1 6% cheaper); blob storage favours **West Europe**. At these volumes the
deltas are cents — choose on latency and data residency, not price.
</hosting-ranking>

**Verdict: Azure Container Apps Consumption with `minReplicas=1` at 0.25 vCPU / 0.5 GiB — realistically $0/month inside the free grant (180k vCPU-s + 360k GiB-s + 2M requests per subscription), free custom domain and managed TLS, unrestricted .NET 10, and no cold-start cliff. `minReplicas=0` saves nothing further and costs tens of seconds of cold start. App Service B1 at $13.14/mo is the low-configuration fallback if ACA's session-affinity + Data-Protection wiring proves annoying. SWA Free is the winner *only* if the app is Blazor WASM — it cannot host server-rendered .NET at all, and its managed Functions backend is capped at .NET 9.**

</6-azure-hosting>

<7-storage>

## 7. Storage

<blob>
### Azure Blob Storage (Standard GPv2, LRS)

Capacity per GB/month, first 50 TB [API + pricing-page JSON, in exact agreement]:

| Tier | West Europe | East US | Min retention |
|---|---|---|---|
| **Hot** | **$0.0196** | $0.0208 | none |
| Cool | $0.0100 | $0.0152 | 30 days |
| Cold | $0.0045 | $0.0036 | 90 days |
| Archive | $0.0018 | $0.00099 | 180 days |

Note **East US hot *and* cool are more expensive than West Europe** — there is no
blanket "US is cheaper" rule for storage.

Transactions per 10,000, West Europe (East US):

| Operation | Hot | Cool |
|---|---|---|
| Write | $0.054 ($0.05) | $0.10 |
| List / Create Container | $0.054 ($0.05) | $0.054 |
| **Read** | **$0.0043** ($0.004) | $0.010 |
| **Delete** | **Free** | Free |

Egress: first **100 GB/month free, always** (not a trial benefit), then $0.087/GB.

**There is no always-free Blob or Table allowance.** The free account gives 5 GB hot
block blob + 20,000 reads + 10,000 writes **for 12 months only**; Table Storage appears
nowhere in the free list.
</blob>

<table-storage>
### Azure Table Storage

**$0.045 / GB / month (LRS)** and **$0.00036 per 10,000 transactions** — all operation
types, identical in West Europe and East US.

**Trap worth flagging:** the second table on the Tables pricing page is for
**account-scoped encryption** (customer-managed keys) at $0.0585/GB and **$0.117/10k
for scan and list — a ~325× multiplier**. Use the default service-scoped encryption
(already AES-256 at rest) and don't enable encryption scopes without a compliance reason.

**Key asymmetry:** Table is ~2.3× Blob-hot per GB, but **~12× cheaper per read and
~150× cheaper per write**. Bulk bytes → Blob; many small records → Table.
</table-storage>

<cosmos-and-sql>
### Cosmos DB serverless and Azure SQL serverless

**Cosmos serverless:** $0.305 per 1M RUs (WE) / $0.25 (EUS), plus $0.25/GB/month.
**No minimum charge and no hourly floor** — an idle account is ~$0, so ~$0.56/mo at
1M RU + 1 GB. **But the Cosmos free tier (1000 RU/s + 25 GB, lifetime, one per
subscription) is alive** (doc updated 2026-04-27) **and is explicitly unavailable on
serverless accounts**: *"Free tier is currently not available for serverless accounts."*
The two are mutually exclusive. Documents cap at 2 MB, so images need Blob regardless.
The 30-day *"Try Cosmos DB free"* sandbox appears retired — **unverified**, don't plan
around it.

**Azure SQL serverless:** GP Gen5 compute **$0.573934/vCore-hour** (WE), storage
$0.13685/GB/mo. **Auto-pause delay minimum is 15 minutes — 60 is merely the default.**
The real billing floor is **0.673 vCores, not 0.5** (billing takes
`max(min vCores, used, min memory GB ÷ 3, …)`; 0.5 min vCores implies 2.02 GB → 0.673).
Realistic minimum (1 h/day usage, 15-min auto-pause, 5 GB): **≈ $15.38/month** (WE).
Leaving the 60-minute default instead: **$24.21/month** — a 57% swing from one setting.
Resume takes ~1 minute and the first connection fails with **error 40613**, so retry
logic is mandatory.

**The Azure SQL free offer still exists and is perpetual:** *"The database is free
forever, with monthly limits. There's no time limit."* — 100,000 vCore-seconds + 32 GB
data per database, up to 10 GP databases per subscription. The catch: the 15-minute
auto-pause config consumes 92,225 vCore-s (~8% headroom, **free**), while the
**60-minute default consumes 147,560 — 48% over**. Default overage behaviour is
auto-pause until next month; switching to "continue for additional charges" is a
**one-way door**.
</cosmos-and-sql>

<sqlite-danger>
### SQLite on a mounted volume — **NOT SAFE. Don't.**

Microsoft says so directly, twice:

> "**We don't recommend that you use storage mounts for local databases, such as
> SQLite**, or for any other applications and components that rely on file handles and
> locks." — [configure-connect-to-azure-storage](https://learn.microsoft.com/en-us/azure/app-service/configure-connect-to-azure-storage)

> "The file system of your application is **a mounted network share**… **Unfortunately
> this blocks the use of file-based database providers like SQLite since it's not
> possible to acquire exclusive locks on the database file.**"

Three independent failure mechanisms:

1. **Advisory locks may be silently unenforced.** SQLite's own docs: *"POSIX advisory
   locking is known to be buggy or even unimplemented on many NFS implementations…
   **Your best defense is to not use SQLite for files on a network filesystem.**"*
2. **`nobrl` makes it strictly worse — and Microsoft recommends `nobrl`.** The
   recommended SMB `mountOptions` that Container Apps links to include
   `nobrl  # disable sending byte range lock requests to the server`. With `nobrl`,
   **SQLite's locking becomes a complete no-op.**
3. **WAL mode does not work over network filesystems.** *"All processes using a
   database must be on the same host computer; WAL does not work over a network
   filesystem."* The usual "just turn on WAL" fix is an **anti-pattern** here: it makes
   `SQLITE_BUSY` disappear while making the real hazard worse.

Failure mode is **silent corruption discovered later**, not a clean error at write
time. **"Single user" does not protect you** — the dangerous concurrency comes from the
*deployment system* (rolling revision overlap starts a new replica before tearing down
the old), not from users. Container Apps has **no block-storage option at all**; Azure
Files is the only persistent mount, and NFS requires premium/SSD with a **100 GiB
minimum ≈ $19.20/month floor** for a 10 MB database.

If SQLite is genuinely wanted: container-local **ephemeral** disk (correct POSIX
locking, WAL works normally) + **Litestream** replicating to Azure Blob (`abs://`,
Managed Identity in v0.5.0+), `minReplicas=1, maxReplicas=1`, restore-on-start —
accepting async replication lag as the RPO.
</sqlite-danger>

<storage-recommendation>
### Cheapest adequate option

**Blob (hot, LRS) for Markdown + screenshots; Table Storage for annotation records.**
One storage account, one managed identity, no connection string.

| Line | Cost |
|---|---|
| Blob 2.05 GB @ $0.0196 | $0.0402 |
| Table 0.0098 GB @ $0.045 | $0.0004 |
| 5,500 Table transactions @ $0.00036/10k | $0.0002 |
| Egress (under 100 GB) | $0 |
| **Total** | **≈ $0.041 / month** |

**$0.00 for the first 12 months** on a new subscription. All-Blob would be $0.045 —
the split saves nothing at this volume, so choose it for the *shape* fit (opaque bytes
vs keyed records), not the money. It pays off decisively only if annotation traffic
ever grows orders of magnitude, where Table's 12×-cheaper reads and ~150×-cheaper
writes dominate.

The three data shapes map cleanly onto two primitives: Markdown and images are opaque
blobs addressed by key; annotations are small keyed records with a natural
`PartitionKey` (document id) / `RowKey` (paragraph anchor id from section 3).
**Don't add a database unless you need queries you can't express as a partition scan**
— and note images must live in Blob under *every* option anyway (Cosmos caps documents
at 2 MB, Table entities at 1 MB), so a database only ever adds a second system beside
Blob rather than replacing it.
</storage-recommendation>

**Verdict: Azure Blob Storage (hot, LRS) for Markdown and images + Azure Table Storage for annotations — ≈ $0.04/month, and $0.00 for the first 12 months. Cosmos serverless (~$0.56/mo) and Azure SQL serverless (~$15.38/mo real floor, or free-forever if auto-pause is set to 15 minutes rather than the 60-minute default) are both defensible but add a second system beside the Blob you need anyway. SQLite on a mounted Azure Files share is actively unsafe — Microsoft explicitly recommends against it, the recommended `nobrl` mount option turns SQLite's locking into a no-op, and WAL mode does not work over a network filesystem.**

</7-storage>

<8-secrets-without-leaks>

## 8. Secrets without leaks (public repo)

<the-headline>
**The end state is that there are no secrets at all** — not "secrets stored
safely". OIDC replaces the deploy credential; Managed Identity + RBAC replaces
the storage connection string; and `allowSharedKeyAccess: false` makes the
eliminated key *non-existent* rather than merely unused. The deliverable is
`Settings → Secrets and variables → Actions` containing **zero secrets**.
</the-headline>

<timing-landmine>
### A landmine specific to starting this repo today

**GitHub changed the OIDC subject claim format on 2026-07-15.** Repos created
*after* that date emit **immutable subject claims** embedding numeric IDs:

```text
OLD:  repo:octo-org/octo-repo:ref:refs/heads/main
NEW:  repo:octo-org@123456/octo-repo@456789:ref:refs/heads/main
```

([changelog, 2026-04-23](https://github.blog/changelog/2026-04-23-immutable-subject-claims-for-github-actions-oidc-tokens/))

cloudreview's repo already exists, so it likely uses the **old** format — but
this must be *checked, not assumed*, because Microsoft documents that a wrong
subject **fails silently**:

> "If you accidentally add the incorrect external workload information in the
> *subject* setting the federated identity credential is created successfully
> without error. The error does not become apparent until the token exchange
> fails."

**Rule: do not hand-type the subject.** Use GitHub's OIDC settings preview
endpoint, or run the workflow once and read the actual `sub` claim out of the
failure. Wildcards are not supported in any FIC property.
</timing-landmine>

<github-oidc>
### GitHub OIDC federated credentials

Current action versions: **`azure/login@v3`** (v3.0.0, Node 20→24),
`azure/webapps-deploy@v3`, `azure/container-apps-deploy-action@v1`,
`Azure/setup-azd@v2.3.0`. Note **learn.microsoft.com still shows `azure/login@v2`**
in every OIDC article — the docs lag the action; v3 is current.

`permissions: id-token: write` is what creates `ACTIONS_ID_TOKEN_REQUEST_TOKEN`
on the runner. Default audience `api://AzureADTokenExchange`. Azure CLI ≥ 2.30.

Use a **user-assigned managed identity**, not an app registration —
**FIC is not supported on system-assigned identities**, and a UAMI cannot
accidentally have a client secret added to it later.

```bash
az identity create --name id-github-deploy --resource-group $RG --location $LOC

az identity federated-credential create \
  --name gh-env-prod \
  --identity-name id-github-deploy \
  --resource-group $RG \
  --issuer 'https://token.actions.githubusercontent.com' \
  --subject 'repo:OWNER/REPO:environment:prod' \
  --audiences 'api://AzureADTokenExchange'
```

Limits: max 20 FICs per identity, issuer+subject ≤ 600 chars, exactly one
audience, no wildcards.

**Variables vs secrets.** Microsoft Learn says use secrets; Microsoft's own
azd-generated workflow uses `vars.AZURE_CLIENT_ID` etc. Both are defensible —
client/tenant/subscription IDs are **identifiers, not credentials**; possessing
them grants nothing without a matching FIC. Use `vars`; the FIC subject is the
actual security boundary.
</github-oidc>

<public-repo-risk>
### The public-repo risk — the part that actually matters

**Can a fork PR mint your Azure token? No, by default — but there are two ways
to break that.**

Why it's safe by default, mechanism 1 — the permissions downgrade
([workflow syntax docs](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax)):

> "if the workflow was triggered by a pull request event other than
> `pull_request_target` from a forked repository, and the **Send write tokens to
> workflows from pull requests** setting is not selected, the permissions are
> adjusted to change any write permissions to read only."

`id-token` is the **only** scope whose values are `write | none` — there is no
`id-token: read`. So on a fork PR it degrades to `none` and the request token
env var is never set. Mechanism 2: with the exception of `GITHUB_TOKEN`, secrets
are simply not passed to fork-triggered runs.

**Danger 1 — `pull_request_target` (the "pwn request").** It grants read/write
`GITHUB_TOKEN` *even from a public fork*. A `pull_request_target` workflow that
checks out `github.event.pull_request.head.sha` and runs anything from it (build
script, npm postinstall, MSBuild target) executes attacker code with
`id-token: write` available → attacker mints an Azure token. **This is the one
realistic path to a secret-free-but-still-compromised subscription.** Recent
hardening helps but does not excuse using it: `actions/checkout` **v7**
(2026-06-18, backported 2026-07-20) now *refuses* fork-PR checkout under
`pull_request_target` unless you pass `allow-unsafe-pr-checkout`.

**Danger 2 — the repo setting "Send write tokens to workflows from pull
requests."** If enabled, the downgrade above does not happen. **Must stay OFF.**

**The safe pattern:**

```yaml
on:
  push: { branches: [main] }     # NOT pull_request_target
  workflow_dispatch:

permissions:
  contents: read                 # file-level default: no id-token

jobs:
  deploy:
    environment: prod            # required reviewers live here
    permissions:
      id-token: write            # granted ONLY in this job
      contents: read
```

1. Never use `pull_request_target` in this repo.
2. Never grant `id-token: write` at workflow level — per-job only.
3. **Bind the FIC to `environment:prod`, not a branch**, and put required
   reviewers on that environment. A token can only be minted after a human
   approves. Strongest available control, free.
4. Do not create a `repo:OWNER/REPO:pull_request` FIC.
5. Separate CI (PR builds, no `id-token`) from CD.
6. Pin third-party actions by **full 40-char SHA** — Microsoft's own OIDC doc
   does this.
7. Scope narrowly: `Website Contributor` on the single site, not `Contributor`
   on the subscription.

Also kill the *other* long-lived App Service credential:

```bash
az resource update -g $RG --name scm --namespace Microsoft.Web \
  --resource-type basicPublishingCredentialsPolicies --parent sites/$APP --set properties.allow=false
az resource update -g $RG --name ftp --namespace Microsoft.Web \
  --resource-type basicPublishingCredentialsPolicies --parent sites/$APP --set properties.allow=false
```
</public-repo-risk>

<managed-identity-storage>
### Managed Identity + RBAC — eliminating connection strings entirely

**`Azure.Identity` 1.21.0** (published 2026-04-11). `DefaultAzureCredential`
chain: Environment → Workload Identity → **Managed Identity** → Visual Studio →
VS Code → **Azure CLI** → Azure PowerShell → azd. So `az login` locally and
managed identity in Azure run **identical code with zero config difference and
zero stored secrets**.

```csharp
builder.Services.AddAzureClients(clients =>
{
    clients.AddBlobServiceClient(new Uri($"https://{account}.blob.core.windows.net"));
    clients.AddTableServiceClient(new Uri($"https://{account}.table.core.windows.net"));
    clients.UseCredential(new DefaultAzureCredential());
});
```

Set `AZURE_TOKEN_CREDENTIALS=prod` as an app setting in Azure (skips dev-tool
credentials, avoids a wasted round-trip per startup; requires Azure.Identity ≥ 1.15.0).

Exact roles needed (note the **data-plane vs control-plane split** — `Contributor`
on the account does **not** grant blob/table data access; this trips up everyone once):

| Role | GUID |
|---|---|
| **Storage Blob Data Contributor** | `ba92f5b4-2d11-453d-a403-e96b0029c9fe` |
| **Storage Table Data Contributor** | `0a9a7e1f-b9d0-4cc4-a60d-0319b160aaa3` |

Then prove it — turn the keys off entirely:

```bash
az storage account update -n $SA -g $RG --allow-shared-key-access false
```

> "When you are confident that you can safely reject requests that are authorized
> with Shared Key, you can set the `AllowSharedKeyAccess` property for the storage
> account to false… For optimal security, Microsoft recommends using Microsoft
> Entra ID with managed identities."

**This flag is the single highest-value anxiety-killer in this document: with it
set, a leaked connection string is not merely unlikely — it is inert.**
</managed-identity-storage>

<key-vault-verdict>
### Key Vault — recommended **against**, initially

Syntax for reference: App Service uses
`@Microsoft.KeyVault(SecretUri=https://vault.vault.azure.net/secrets/name)` or
`@Microsoft.KeyVault(VaultName=v;SecretName=s)`; Container Apps uses a different
form, `name=keyvaultref:<uri>,identityref:<mi-id>`. Both need
*Key Vault Secrets User*. Unversioned references refresh within 24 hours.

Pricing: Microsoft's pricing page renders `$-` placeholders client-side and is
**unverifiable from the primary source**; third-party trackers consistently
report ~$0.03 per 10,000 operations, no per-secret storage fee — treat as
approximate.

**Cost is not the reason to skip it** (fractions of a cent/month here). The
reason is that **it solves a problem this design doesn't have.** Key Vault is a
secret store, and the whole thesis above is that there are no secrets. An empty
Key Vault is pure operational surface — one more resource, one more RBAC grant,
one more identity-propagation failure mode, one more way for an app setting to
resolve to the literal string `@Microsoft.KeyVault(...)` at 3am.

**Clear trigger to revisit:** add it the day a secret arrives that genuinely
cannot be replaced by an identity — a third-party API key (e.g. **the OpenAI key
from section 5**), or a SQL password if not using Entra auth. Since section 5
recommends OpenAI `gpt-transcribe`, that day is probably day one — so plan for
Key Vault *or* simply hold the OpenAI key as a Container Apps secret / App
Service app setting, which is adequate for a single-user app and keeps the
resource count down.
</key-vault-verdict>

<local-dev>
### Local dev

`dotnet user-secrets` stores plaintext JSON at
`%APPDATA%\Microsoft\UserSecrets\<id>\secrets.json` — outside the project tree,
never in source control. But note the docs' own warning: *"Secret Manager doesn't
encrypt the stored secrets and shouldn't be treated as a trusted store."* It keeps
secrets out of **git**; it protects them from nothing on the machine.

`dotnet user-secrets init` adds `<UserSecretsId>` to the `.csproj`; the provider
is auto-registered only when `EnvironmentName == "Development"`.

**Best outcome: you never call `user-secrets set` at all** — with `az login` +
`DefaultAzureCredential`, local dev needs zero stored secrets. Run `init` anyway
so there's an obvious right place for the first third-party API key.
</local-dev>

<azd-and-iac>
### azd, and Bicep vs Terraform

**azd 1.28.1** (released 2026-07-23). `azd init` / `azd up` / `azd down` /
`azd infra generate` (`azd infra synth` is now a deprecated alias).

**`azd pipeline config` does OIDC by default** — you don't even need
`--auth-type federated`. Per [the docs](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/pipeline-github-actions)
(ms.date 2026-07-20): *"By default, `azd pipeline config` configures OpenID
Connect (OIDC), also called federated credentials… OIDC/federated credentials are
**not** supported for Terraform."*

Caveats before running it: it is **in beta**; it defaults to **subscription-scoped
`Contributor`** (over-privileged — narrow it after, or pre-create the UAMI); it
creates an app registration rather than a UAMI; and it offers to create a
*private* repo, so point it at the existing public one.

Templates: `hello-azd` (a Blazor app, with the full pipeline walkthrough),
`Azure-Samples/todo-csharp-sql` (C# API + Bicep on App Service), 300+ in the
[awesome-azd gallery](https://azure.github.io/awesome-azd/). Bicep is the default
IaC provider; Terraform is still beta.

**Use Bicep. Not a close call:**

1. **Terraform breaks the zero-secret goal outright** — azd cannot configure OIDC
   for Terraform, putting you back on a client secret. This one fact ends it.
2. **No state file** — Bicep deploys against ARM, which *is* the state. Terraform
   needs a remote backend, a lock, and a state file that itself contains sensitive
   values: a whole second infrastructure problem and a classic leak vector.
3. **No provider version drift**, no `terraform init`, fewer moving parts an agent
   can break unattended.
4. **For an AI agent specifically:** no plan/apply/state lifecycle to mis-sequence.
   `az deployment group what-if` gives you plan preview without state machinery.

Terraform wins on multi-cloud and large module ecosystems. This project has neither.
</azd-and-iac>

<push-protection>
### Push protection and .gitignore

Secret scanning + push protection are **free on public repos and default-on since
2024-03-11 — but only for repos owned by *personal accounts*, and only *new* ones.**
Existing repos and org-owned repos are **not** covered. **Verify, don't assume:**
Settings → Code security → Push protection. Note an admin can disable it and a
pusher can bypass a block with a reason — it is a safety net, not a wall. Add a
local gitleaks/trufflehog pre-commit hook too, since push protection only blocks
*at the remote*, by which point the secret has left the machine.

`dotnet new gitignore` does **not** exclude `appsettings.*.json`. Add:

```gitignore
appsettings.Development.json
appsettings.Local.json
**/secrets.json
*.pubxml
*.publishsettings
.env
.env.*
!.env.example
.azure/            # azd local env state — contains subscription IDs
```

`.gitignore` does nothing for already-tracked files —
`git rm --cached <file>` if needed. **Better: don't rely on gitignore at all.**
Commit an `appsettings.Development.json` containing only non-secret dev settings,
and keep every real secret out of the tree. Defence that depends on remembering a
gitignore line eventually fails.
</push-protection>

<verify-dont-trust>
### Verify the claims rather than trusting them

The step that actually answers the owner's anxiety: **open a PR from a fork of
your own repo with a job requesting `id-token: write`, and confirm it fails with
`ACTIONS_ID_TOKEN_REQUEST_TOKEN` unset.** Then confirm the storage account rejects
a shared-key request, confirm the Actions secrets list is empty, and confirm a
merge to `main` *pauses for approval* before any Azure token is minted.
</verify-dont-trust>

<doc-smells>
### Documentation smells found (flagged, not acted on)

1. **`learn.microsoft.com/azure/container-apps/github-actions`** (updated
   2026-05-08) still teaches `az ad sp create-for-rbac --json-auth` →
   `AZURE_CREDENTIALS` repo secret → `azure/login@v1`. That is a long-lived client
   secret, in current Microsoft documentation, with a two-major-versions-stale
   action. **Do not follow it.**
2. All learn.microsoft.com OIDC articles pin `azure/login@v2` while the action
   repo ships v3.
3. `azd pipeline config` defaults to subscription-scoped `Contributor`.
4. Microsoft's Key Vault pricing page renders `$-` placeholders — no pricing claim
   from it is machine-verifiable, including the one above.
5. Parallel ARM creation of multiple FICs on one identity throws 409; Microsoft's
   workaround is `dependsOn` chaining. Relevant the moment a second environment
   is added.
</doc-smells>

**Verdict: user-assigned managed identity + GitHub OIDC federated credential bound to a `prod` environment with required reviewers, `azure/login@v3` with IDs in repo `vars`, `DefaultAzureCredential` + Storage RBAC data roles, and `allowSharedKeyAccess: false` so the eliminated connection string is inert. Bicep, not Terraform (Terraform cannot use OIDC under azd). No Key Vault until the OpenAI key forces it. Never `pull_request_target`; never `id-token: write` above job level; verify by attempting a fork-PR token grab.**

</8-secrets-without-leaks>

<recommended-stack>

## Recommended stack

<the-stack>
| Layer | Choice | Two-line justification |
|---|---|---|
| **Runtime** | **.NET 10 (LTS)** | GA 2025-11-11, supported to 2028-11-14. .NET 8 and 9 both die 2026-11-10, ~3.5 months away — starting on either would mean an immediate forced upgrade. |
| **Web framework** | **ASP.NET Core Razor Pages + Minimal APIs**, server-rendered | Keeps the owner in C#/.NET where he is strongest, with no build step, no hydration model, and no JS framework to rot. The page is 95% static HTML; only three small interactions are dynamic. |
| **Interactivity** | **~70 lines of vanilla JS** + Popover API + CSS anchor positioning | Popover reached Baseline in April 2025 and anchor positioning is at 81.67% — the paragraph-side editor is now pure HTML+CSS, zero JS. The remaining JS is one paste handler and one recorder. |
| *(optional)* | **htmx 2.0.10, pinned exactly** | Only for the documented click-to-edit idiom; saves ~15 lines. **Pin it** — htmx 4 beta flips attribute inheritance from implicit to explicit, silently changing the meaning of code that still parses. |
| **Markdown** | **Markdig 1.3.2** + `UsePreciseSourceLocation()` | The only serious .NET Markdown library, post-1.0, BSD-2, and the only one exposing a mutable AST with source spans — which is what makes stable annotation anchoring possible at all. |
| **Anchoring** | **`data-block-id` = truncated SHA-256 of the whitespace-normalised block source** | *Measured*: survives both edits elsewhere and insertions at the top of the document. `Block.Line` silently re-points every annotation when a paragraph is inserted. |
| **XML headings** | **Regex pre-pass → ATX headings**, then `UseAutoIdentifiers()` | *Measured*: without it, tags with no surrounding blank lines swallow the following paragraph unparsed and emit malformed `</section-name></p>`. The pre-pass is ~6 lines. |
| **Sanitisation** | **HtmlSanitizer (Ganss.Xss) 9.1.973**, allow-listing `id`/`class`/`data-block-id` | Markdig passes `<script>` and `javascript:` straight through — *measured*. And the sanitizer's defaults strip `id`/`class`, which would silently destroy the anchoring scheme. |
| **Dictation** | **OpenAI `gpt-transcribe`** via a server-side proxy | 3.3% WER at $0.0045/min ≈ $0.27/month — directly answers "Google voice typing isn't very good". `MediaRecorder` → POST blob → server; the key never reaches the browser. |
| **Hosting** | **Azure Container Apps, Consumption, `minReplicas=1`, 0.25 vCPU / 0.5 GiB** | Realistically $0/month inside the free grant (180k vCPU-s + 360k GiB-s + 2M requests per subscription), with free custom domain and managed TLS and unrestricted .NET 10. `min=1` avoids a measured 15–37 s cold start for no meaningful saving. |
| **Storage** | **Blob (hot, LRS) + Table Storage**, one account | ≈ $0.04/month, and $0.00 for the first 12 months. Markdown and images are opaque blobs; annotations are keyed records with a natural document/paragraph partition — no database earns its keep. |
| **Identity** | **User-assigned Managed Identity + `DefaultAzureCredential` 1.21.0** | `az login` locally and managed identity in Azure run identical code with zero stored secrets. Storage data-plane RBAC roles remove the connection string entirely. |
| **Deploy** | **GitHub OIDC federated credential bound to a `prod` environment with required reviewers**, `azure/login@v3` | No publish profile, no client secret, nothing in Actions secrets. The environment gate means no Azure token can be minted without a human approving. |
| **IaC** | **Bicep** (optionally via `azd`) | Terraform cannot use OIDC under `azd` — it would reintroduce the exact client secret we are eliminating. Bicep also has no state file to secure or leak. |
| **Hardening** | **`allowSharedKeyAccess: false`** on the storage account | The single highest-value setting in the whole design: it makes a leaked connection string not merely unlikely but **inert**. |
| **Not included** | Key Vault, Cosmos/SQL, SQLite, any JS framework, any Blazor interactivity | Each was evaluated and rejected on evidence above, not omitted by oversight. Add Key Vault only when the OpenAI key makes it worthwhile. |
</the-stack>

<overruling-blazor>
### Where I am overruling the owner's Blazor preference — and where I am not

**I am overruling interactive Blazor. The reason is specific and, I think, decisive:
every single interactive feature cloudreview needs is one Blazor cannot express in C#.**

1. The rendered Markdown is one blob of HTML injected as a `MarkupString`. **Blazor
   cannot attach `@onclick` to anything inside a `MarkupString`** — it bypasses the
   component compiler, so the directive is inert text
   ([dotnet/aspnetcore#18301](https://github.com/dotnet/aspnetcore/issues/18301),
   [#18459](https://github.com/dotnet/aspnetcore/issues/18459)). The alternatives are
   to hand-build a `RenderFragment` tree from the Markdig AST — a large bespoke
   renderer, precisely what we are trying to avoid — or JS event delegation.
2. **Pasting a screenshot:** Blazor's `ClipboardEventArgs` exposes no clipboard payload
   at all. JS interop is mandatory.
3. **Dictation:** no .NET surface exists. JS interop is mandatory.

So the interactive core — click, paste, dictate — is the same three JS modules whether
or not Blazor is present. Blazor would sit on top contributing a SignalR circuit, a
reconnection UI, a WebSocket hosting requirement, and a render-mode decision, while
adding nothing to the parts that are actually hard. That is a worse deal than writing
70 lines of JS directly.

**But I am not overruling .NET, and that matters more than it sounds.** The
recommendation keeps the owner in C#, Razor syntax, NuGet, and Visual Studio — the
whole ecosystem he knows. If he prefers the `.razor` file format to `.cshtml`, he can
even build this as a **Blazor Web App with global Static SSR and no interactive render
mode anywhere**: the components become a templating engine, which is a perfectly
reasonable way to write server-rendered HTML. That is a genuine, honest partial win for
his preference and costs nothing.

The thing to reject is not Blazor-the-syntax. It is **Blazor-the-interactivity-model**,
which for this particular app buys a circuit and pays for it in latency, hosting
constraints, and workarounds — while the three features that justify the app still have
to be written in JavaScript regardless.

**One more honest note:** if the owner is willing to give up dictation and accept
commenting on Markdown *source* rather than rendered output, the **GitHub PR review**
fallback (section 2) is ~40 lines of glue and nothing to maintain — image paste works
and an agent can fetch the screenshots anonymously, both verified. It is worth
prototyping for an afternoon before committing to building anything at all. If it turns
out to be tolerable, the cheapest correct answer is to build nothing.
</overruling-blazor>

<open-questions>
### Unverified items, carried forward honestly

1. Whether App Service Linux still labels **.NET 10 as "(Preview)"** in the portal.
   Irrelevant to the recommendation (Container Apps has no runtime gate), but relevant
   if the B1 fallback is taken.
2. What happens to an **in-flight WebSocket when Container Apps scales to zero** — no
   Microsoft doc addresses it. Moot under the recommended `minReplicas=1`.
3. **.NET-specific ACA cold-start latency** — the only rigorous public benchmark used a
   Quarkus native image. Also moot under `minReplicas=1`.
4. Whether the **Web Speech API still has no quota** in 2026 — the "no limit" claim
   traces to an older chromium-dev post. Moot if OpenAI transcription is used.
5. **Azure OpenAI transcription pricing** — the pricing page renders `$-` placeholders
   and the Retail Prices API does not surface the meters.
6. **Key Vault pricing** — same `$-` placeholder problem; the ~$0.03/10k operations
   figure comes from third-party trackers, not Microsoft.
</open-questions>

</recommended-stack>
