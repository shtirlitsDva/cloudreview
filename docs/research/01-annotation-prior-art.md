# cloudreview — Annotation Prior Art

<meta>
Research date: 2026-07-29
Question: What already exists for "render a document, let a human annotate it inline, then export the annotations machine-readably"?
Method: WebSearch + WebFetch against primary sources (spec text, official API docs, source repos). Licences verified from repo LICENSE files / npm metadata where stated; anything unconfirmed is explicitly marked **unverified**.
</meta>

<executive-summary>
The single most important finding: **this product already exists, at least three times over.** Plannotator (Apache-2.0/MIT, open source), Moat (SaaS), and Markloop (SaaS) are all shipping in 2026 and do very nearly what cloudreview proposes — agent publishes a document, human annotates it in a browser, structured feedback returns to the agent. Plannotator is the closest and is permissively licensed, so it can be read, borrowed from, or forked outright.

The second most important finding: across the best-specified prior art (OOXML, Review Board, Gerrit, Hypothesis, Markloop), the designs **converge on the same answer** — anchor redundantly (stable block id + source range + quoted text), keep comment bodies out-of-band in a sidecar keyed by anchor id, and batch the review into one atomic publish event.

The third: **voice comments and pasted screenshots have essentially no prior art anywhere.** Every system surveyed is text comments plus, at most, file attachments (Plannotator supports image attachment; nothing supports dictation). That part is genuinely novel and is where differentiation lives — not in the annotation mechanics.
</executive-summary>

---

<section-1-agent-native-review-tools>
# 1. Purpose-built for agent-authored documents (the closest prior art of all)

This category did not meaningfully exist 18 months ago. It does now. Treat these as the competitive set.

<plannotator>
## Plannotator — THE closest prior art

- **What:** A local, browser-based review surface for AI coding agents. Agent produces a plan / Markdown doc / HTML / code diff → local server opens it in the browser → human annotates → structured feedback is pushed back into the originating agent session.
- **Link:** https://github.com/backnotprop/plannotator · https://docs.plannotator.ai/open-source/ · https://plannotator.ai/
- **Licence:** **Dual Apache-2.0 / MIT at your option** (stated in README). TypeScript. 100% free, no paid tier.
- **Library or product:** Both — it is a product, but the licence means we can read, vendor, or fork it.
- **Architecture:** `plannotator` binary runs a temporary local server on a random port (fixed 19432 in remote mode), opens the browser. Local data in `~/.plannotator`. Small plans are **encoded entirely in the URL hash — no server involved**. Large shared plans go through a short-link service, **AES-256-GCM encrypted in the browser**.
- **Agent integration:** Claude Code, Codex, Copilot CLI, Gemini CLI, OpenCode, Kiro, Droid, Amp, Pi. Wired through each harness's hooks. Flow for Claude Code: `ExitPlanMode` → `PermissionRequest` hook fires → local server reads the plan from hook input → browser opens. Manual commands: `/plannotator-review`, `/plannotator-annotate`, `/plannotator-last`.
- **Anchoring model (verified from docs):** **block-based**, not DOM/XPath.
  ```typescript
  interface Block {
    id: string;
    type: 'paragraph' | 'heading' | 'blockquote' | 'list-item' | 'code' | 'hr' | 'table';
    content: string;
    startLine: number;
  }
  ```
  Selection is associated with a `blockId` and the `originalText` of the selection is preserved. Headings additionally get slug-derived anchor ids; code blocks carry language info.
- **Annotation types:** `DELETION`, `INSERTION`, `REPLACEMENT`, `COMMENT`, `GLOBAL_COMMENT` (file-level). Image attachments supported.
- **Export format — the interesting part.** It does **not** hand the agent JSON annotations. `exportAnnotations` generates **human-readable Markdown** aimed at the LLM:
  ```markdown
  ### Line 42: "Installation" (Heading)

  **DELETION**: Remove redundant section

  ### Line 58: "The API uses REST principles" (Paragraph)

  **COMMENT**: Add link to REST documentation

  **REPLACEMENT**: Original: "The API uses REST principles"
  Suggested: "The API follows REST architectural principles..."
  ```
  Images appear as `[proposed-layout] /tmp/uploads/image-abc123.png`. Global comments cluster at the top.
- **Transport contract (verified):** hook-native mode `--hook` emits `{"decision":"block","reason":"<feedback>"}` when annotations are submitted; empty output on approve or close. Structured mode `--gate --json` emits `{"decision":"annotated","feedback":"<feedback>"}` / `{"decision":"approved"}` / `{"decision":"approved","feedback":"<notes>"}` / `{"decision":"dismissed"}`. With `--require-approval`, `approved` exits 0, `annotated` and `dismissed` exit 1. `--result-file` writes stdout first, then atomically publishes an owner-only file.
- **VERDICT: ADOPT / FORK / AT MINIMUM READ END-TO-END BEFORE WRITING A LINE.** This is cloudreview with a local server instead of a hosted page. The block-id anchoring, the five annotation types, the Markdown-not-JSON export, and the hook decision contract are all directly reusable and permissively licensed. The honest question this raises is **why cloudreview should exist at all** — the defensible answers are (a) *hosted* rather than localhost, so a reviewer on a phone or a non-technical stakeholder can review, and (b) **voice dictation**, which Plannotator does not have.
</plannotator>

<moat>
## Moat (moat.so)

- **What:** "Review layer for agent-written docs." Agent publishes a Markdown artifact (PRD, implementation plan, research report, launch analysis) → reviewer annotates in the browser → feedback syncs back, source file is revised, review is finished.
- **Link:** https://moat.so/
- **Licence:** **Proprietary SaaS.** (Direct fetch returned HTTP 403; details below come from search-result summaries — **the detailed feature claims are unverified against the primary site.**)
- **Design stance worth stealing (their explicit framing):** *"Moat stores the review surface, not the source of truth. The local Markdown stays canonical and Moat is just the browser review layer."* Safety defaults: ask before publishing, warn on likely secrets, never upload folders, keep the canonical file local.
- **Anchoring model / export format:** **unverified** — could not fetch primary source.
- **VERDICT: BORROW THE PRODUCT STANCE.** "The hosted page is a review surface, never the source of truth; local file stays canonical" is exactly the right architectural boundary for cloudreview, and the secret-scanning-before-publish default is a real safety requirement we would otherwise forget.
</moat>

<markloop>
## Markloop (markloop.io)

- **What:** Review platform for **agent-made HTML documents**. Agent pushes HTML artifacts via **MCP** (or human uploads); each upload creates a version in a chain; reviewers open a browser link and pin comments; agent pulls comments back over MCP.
- **Link:** https://markloop.io/
- **Licence:** **Proprietary SaaS.** Solo $19/mo (1 creator seat), Team $49/mo (5 seats). Reviewers always free. 14-day trial, no perpetual free tier. © 2026 Markloop.
- **Anchoring model — dual, and this is the notable bit:**
  - **CSS selector** (e.g. `h2#payments`)
  - **exact text quote** (e.g. `"Apple Pay"`) **with occurrence tracking** (nth match disambiguation)
- **Export format fields:** status (`OPEN` / `RESOLVED`), version number, target (CSS selector), quote (+ occurrence), note (reviewer intent), section context (e.g. `§2 Payments`). Their pitch line: *"Every comment exports with its selector, quote and context — ready to apply in one pass."*
- **Versioning:** comments stay attached to the version they were made on; new revisions "check off what's done."
- **Notable feature we should copy:** **embedded questions** — the agent can ask reviewers questions *inside* the document, and answers come back as structured data alongside the comments. This is precisely the "questionnaire" use case in the cloudreview brief, already shipped.
- **VERDICT: BORROW HEAVILY (the model), IGNORE (the product).** Selector + quote + occurrence + version is a well-judged anchor tuple. Embedded questions validate the questionnaire mode. But it's closed, paid, and HTML-only.
</markloop>

<pair-review>
## pair-review (in-the-loop-labs/pair-review)

- **What:** Local web app for keeping humans in the loop with AI coding agents. Node.js + Express, vanilla JS frontend, SQLite, git worktrees for clean PR checkout. CLI-adapter pattern for Claude, Antigravity, Codex, Copilot, OpenCode, Cursor, Pi.
- **Link:** https://github.com/in-the-loop-labs/pair-review
- **Licence:** **Apache-2.0.**
- **Anchoring:** file + line. **Code only — not Markdown, not specs.**
- **Feedback return:** three paths — Markdown export to paste into chat; **an MCP server exposing `get_user_comments` and `get_ai_suggestions`**; and three Claude Code plugins (`pair-loop`, `code-critic`, `pair-review`).
- **VERDICT: BORROW ONE IDEA.** Wrong document type, but the **MCP-server-as-readback-channel** (`get_user_comments` as a tool the agent calls) is a cleaner integration than hook exit codes and is worth copying.
</pair-review>
</section-1-agent-native-review-tools>

---

<section-2-w3c-and-hypothesis>
# 2. W3C Web Annotation + Hypothesis — the standards question

<w3c-data-model>
## W3C Web Annotation Data Model (Recommendation)

- **Link:** https://www.w3.org/TR/annotation-model/
- **Licence:** W3C Recommendation — free to implement, no royalties.
- **Core shape (§3.1):** JSON-LD with `@context: "http://www.w3.org/ns/anno.jsonld"`, `id`, `type: "Annotation"`, required `target`, optional `body`. `motivation` (§3.3.5) is a controlled vocabulary: assessing, bookmarking, classifying, **commenting**, describing, editing, highlighting, identifying, linking, moderating, **questioning**, replying, tagging. Embedded comment text uses `TextualBody` (§3.2.4) with `value` / `format` / `language`.
- **Selectors (§4.2) — the part that matters for us:**

  | Selector | §  | Fields |
  |---|---|---|
  | `FragmentSelector` | 4.2.1 | `value`, `conformsTo` |
  | `CssSelector` | 4.2.2 | `value` |
  | `XPathSelector` | 4.2.3 | `value` |
  | `TextQuoteSelector` | 4.2.4 | `exact` (required), `prefix`, `suffix` |
  | `TextPositionSelector` | 4.2.5 | `start`, `end` (exclusive) |
  | `DataPositionSelector` | 4.2.6 | `start`, `end` (bytes) |
  | `SvgSelector` | 4.2.7 | `value` |
  | `RangeSelector` | 4.2.8 | `startSelector`, `endSelector` |

  `refinedBy` (§4.2.9 / §4.3.3) chains selectors — "each refines the results of the previous one."

  TextQuoteSelector, spec Example 23:
  ```json
  {
    "@context": "http://www.w3.org/ns/anno.jsonld",
    "id": "http://example.org/anno23",
    "type": "Annotation",
    "body": "http://example.org/comment1",
    "target": {
      "source": "http://example.org/page1",
      "selector": {
        "type": "TextQuoteSelector",
        "exact": "anotation",
        "prefix": "this is an ",
        "suffix": " that has some"
      }
    }
  }
  ```
  RangeSelector, spec Example 28:
  ```json
  {
    "type": "RangeSelector",
    "startSelector": { "type": "XPathSelector", "value": "//table[1]/tr[1]/td[2]" },
    "endSelector":   { "type": "XPathSelector", "value": "//table[1]/tr[1]/td[4]" }
  }
  ```
- **`target.selector` may be an ARRAY** — multiple selectors describing the *same* target, so a consumer can try each. This is the single most useful property of the model for us.
- **VERDICT: BORROW THE SELECTOR VOCABULARY, DO NOT ADOPT JSON-LD WHOLESALE.** Use the *names and semantics* of `TextQuoteSelector` (`exact`/`prefix`/`suffix`) and `TextPositionSelector` (`start`/`end`) so the payload is legible to anyone who knows the standard and convertible later. Do **not** take on `@context`, IRI-valued ids, RDF semantics, or JSON-LD processing — the agent consuming this is an LLM reading JSON, not a triple store, and the ceremony buys nothing.
</w3c-data-model>

<w3c-protocol>
## W3C Web Annotation Protocol

- **Link:** https://www.w3.org/TR/annotation-protocol/
- **What:** HTTP CRUD over **LDP Containers** that double as Activity Streams `OrderedCollection`s, with `AnnotationPage` pagination. Required media type `application/ld+json;profile="http://www.w3.org/ns/anno.jsonld"`. Servers advertise constraints via `Link rel="http://www.w3.org/ns/ldp#constrainedBy"`; clients discover containers via `Link rel="http://www.w3.org/ns/oa#annotationService"`. ETags for update/delete.
- **VERDICT: IGNORE.** It requires LDP + Activity Streams + JSON-LD content negotiation to serve what is, for cloudreview, a single document with a single reviewer and one "Finish" button. Pure ceremony. The Data Model is worth reading; the Protocol is not worth implementing.
</w3c-protocol>

<hypothesis>
## Hypothesis (hypothes.is)

- **Link:** https://github.com/hypothesis/client · https://web.hypothes.is/blog/fuzzy-anchoring/
- **Licence:** **BSD-2-Clause**, verified from the repo LICENSE file. (GitHub reports NOASSERTION only because subcomponent notices are appended — one of them for the MIT/GPL Annotator.js code it descends from.) Actively maintained (pushed 2026-07-28, funded org).
- **Library or product:** both, but the client is heavy — sidebar iframe, cross-frame RPC, tied to their service. The *anchoring module* is the reusable part.
- **Anchoring — this is the best implementation of this problem in existence.** It stores **all three selectors simultaneously** on every annotation:
  1. `RangeSelector` — XPath pair + string offsets
  2. `TextPositionSelector` — character offsets in the document text
  3. `TextQuoteSelector` — `exact` + **32 chars of prefix** + **32 chars of suffix**

  And re-anchors with a **progressive fallback chain**:
  1. **Range** (fastest) — apply the XPaths; fails if DOM structure changed
  2. **Position** — character offsets; works if structure changed but content didn't
  3. **Context-first fuzzy** — fuzzy-match the prefix/suffix; survives both structural and content change
  4. **Selector-only fuzzy** — "last-ditch": fuzzy search for the exact text alone

  Source: `src/annotator/anchoring/types.ts` implements `RangeAnchor`, `TextPositionAnchor`, `TextQuoteAnchor`, `MediaTimeAnchor`; `match-quote.ts` does approximate string matching with a position `hint`.
- **Known failure mode:** annotations that match nothing become **orphans**. Hypothesis has no automatic change detection when a document is edited outside its control.
- **Export:** genuine W3C Web Annotation JSON with a `selectors[]` array.
- **VERDICT: BORROW THE ALGORITHM — highest-value technical item in this whole report.** "Emit all three selector types; resolve with progressive fuzzy fallback" is the design decision that determines whether annotations survive the agent republishing a revised document. Do not embed the client; steal the strategy.
- **Related, cleaner source for the same matchers:** `apache/incubator-annotator` — **Apache-2.0**, spec-faithful, zero UI, pure matcher functions (`text-quote/`, `text-position/`, `range/`, `refinable.ts`). **Retired from the Apache Incubator 2025-08-11**, repo archived, last publish 0.2.0 (2021-09-03). It is *finished* rather than abandoned — **vendor the code, do not take a runtime dependency.**
</hypothesis>
</section-2-w3c-and-hypothesis>

---

<section-3-github-and-code-review>
# 3. GitHub PR review + code-review tools — the batching model

<github>
## GitHub pull request reviews

- **Link:** https://docs.github.com/en/rest/pulls/comments · https://docs.github.com/en/rest/pulls/reviews
- **Licence:** proprietary product. **Imitate, do not embed.**
- **Anchoring fields (verified from the REST reference):**
  - `path` (required) — relative file path
  - `commit_id` (required) — SHA of the commit being commented on
  - `diff_hunk` — the diff section for context
  - `subject_type` — `line` or `file`
  - **Legacy:** `position` — position *in the diff*, counting after the first `@@` hunk header. Explicitly *not* the file line number.
  - **Modern:** `line` (line in the diff), `side` (`LEFT` = deletion, `RIGHT` = addition/context, default `RIGHT`)
  - **Multi-line:** `start_line` + `start_side` (first line) with `line` + `side` (last line)
  - **Threading:** `in_reply_to_id` references a top-level comment. **Replies to replies are not supported** — threads are exactly two levels deep. When `in_reply_to` is used, all body params other than `body` are ignored.
- **The batching model — this is what cloudreview should copy precisely:**
  1. **"Start a review"** → creates a `PENDING` review.
  2. Reviewer adds N comments. They are **drafts, private to the author**, invisible to everyone else. The reviewer can edit or delete them freely.
  3. **"Submit review"** → one atomic event carrying an overall `body`, an `event` (`APPROVE` / `REQUEST_CHANGES` / `COMMENT`), and all N comments at once. Notifications fire once.
- **Why this matters for us:** the agent must be woken **once**, with the complete annotation set, not on every keystroke or every comment. The pending-review state also gives the human permission to think — they can revise annotation #2 after writing #7 without the agent having already acted on #2.
- **VERDICT: ADOPT THE BATCHING MODEL VERBATIM.** Draft → accumulate → single submit with a top-level verdict (`APPROVE` / `REQUEST_CHANGES` / `COMMENT`). Note that Plannotator independently arrived at the same three-valued outcome (`approved` / `annotated` / `dismissed`).
</github>

<gerrit>
## Gerrit

- **Link:** https://gerrit-review.googlesource.com/Documentation/rest-api-changes.html
- **Licence:** **Apache-2.0** (verified).
- **Anchoring — `CommentInfo`:** `patch_set`, `id`, `path`, `side` (`REVISION`/`PARENT`), `parent`, `line`, `range`, `in_reply_to`, `message`, `updated`, `author`, `unresolved`. `CommentRange` = **`start_line`, `start_character`, `end_line`, `end_character`** — line + character precision, the correct shape for a text document.
- **Batching:** drafts are private, managed via `GET /changes/.../drafts`, published together through `ReviewInput` when the review is posted. Drafts have no `author` field and are freely editable pre-publication.
- **VERDICT: ADOPT THE RANGE SHAPE.** `{start_line, start_character, end_line, end_character}` maps directly onto Markdown source coordinates and is more precise than GitHub's diff-position hack. Best-documented draft/publish model of the lot.
</gerrit>

<review-board>
## Review Board — `MarkdownReviewUI` (surprise finding, near-exact prior art)

- **Link:** https://www.reviewboard.org/ · source `reviewboard/reviews/ui/markdownui.py`, `reviewboard/reviews/ui/text.py`
- **Licence:** **MIT** (verified via GitHub licence API, `COPYING`). The proprietary "Power Pack" is separate — **Markdown review is in the MIT core.**
- **What it does (from the class docstring):** `class MarkdownReviewUI(TextBasedReviewUI)` — *"This renders the markdown to HTML, and allows users to comment on each top-level block (header, paragraph, list, code block, etc)."* `supported_mimetypes = ['text/markdown', 'text/x-markdown']`, `can_render_text = True`.
- **UX:** **Rendered** and **Source** tabs; comments can be added in either. Hovering highlights a top-level block in grey; clicking opens the comment dialog. Blue flags in the left gutter show per-block comment counts.
- **Anchoring — `SerializedTextComment`:** `beginLineNum: int`, `endLineNum: int`, **`viewMode: str`** (*"either 'source' or 'rendered'"*). Stored in `comment.extra_data`; grouped by the key `f'{beginLineNum}-{endLineNum}'`. Rendered blocks get line numbers via `iter_markdown_lines()` over the rendered HTML.
- **Export:** REST API 2.0 JSON. Base comment fields: `id, extra_data, issue_opened, issue_status, public, text, text_type, timestamp, user`. `extra_data` is exposed and writable.
- **The non-obvious insight to steal:** the **`viewMode` discriminator**. The same line range means different things against the Markdown source versus the rendered HTML, and if you don't record which surface the human was looking at, you cannot faithfully replay the annotation.
- **VERDICT: STUDY CLOSELY / BORROW THE SCHEMA.** MIT, paragraph-level anchoring on rendered Markdown, dual rendered/source addressing, JSON readback. `markdownui.py` + `text.py` are about 400 lines and already solved the anchor-schema problem. Don't embed the Django app; copy the schema.
</review-board>

<reviewable>
## Reviewable

- **Link:** https://reviewable.io · https://docs.reviewable.io
- **Licence:** **proprietary, commercial subscription.** Not embeddable.
- **Two ideas worth taking:**
  1. *"Map line comments across file revisions and keep them around until resolved, not just until changes are pushed."* — **anchor survival across revisions.** When the agent republishes v2, comments must re-anchor, not vanish.
  2. **Dispositions** — per-participant stance per discussion, richer than a resolved boolean. An agent can act differently on "reviewer disagrees" versus "reviewer resolved."
- **Export:** no public comment-export API documented — **unverified.**
- **VERDICT: BORROW TWO IDEAS, IGNORE THE PRODUCT.**
</reviewable>
</section-3-github-and-code-review>

---

<section-4-office-document-models>
# 4. Google Docs / Microsoft Word — how the incumbents anchor

<ooxml>
## Microsoft Word / OOXML — the best-specified model here

- **Link:** https://learn.microsoft.com/en-us/dotnet/api/documentformat.openxml.wordprocessing.commentrangestart (quotes ECMA-376 / ISO-IEC 29500-1 verbatim)
- **Licence:** ECMA-376 is a free open standard. The .NET `DocumentFormat.OpenXml` SDK is **MIT** — genuinely embeddable if we ever need it.
- **Model — inline sentinels + separate part, joined by an integer id.** Spec sections: `commentRangeStart` §17.13.4.1, `commentRangeEnd` §17.13.4.3, `commentReference` §17.13.4.5, `comment` §17.13.4.2. Spec's own example:
  ```xml
  <w:p>
    <w:r><w:t xml:space="preserve">Some </w:t></w:r>
    <w:commentRangeStart w:id="0"/>
    <w:r><w:t>text.</w:t></w:r>
    <w:commentRangeEnd w:id="0"/>
    <w:r><w:commentReference w:id="0"/></w:r>
  </w:p>
  ```
  Comment **bodies live in a separate part** (`word/comments.xml`). Conformance: a `commentRangeStart` with no matching end is a single anchor point; a missing matching `commentReference` makes the document non-conformant.
- **VERDICT: BORROW THE ARCHITECTURE.** The separation — *anchors are inline markers in the content stream; comment content is a separate keyed part; joined by id* — maps exactly onto "inject `<span data-anchor-id>` into the rendered Markdown, keep comments in sidecar JSON." Don't adopt OOXML; adopt its separation of concerns.
</ooxml>

<google-docs>
## Google Docs / Drive comments — weaker than expected

- **Links:** https://developers.google.com/workspace/drive/api/reference/rest/v3/comments · https://developers.google.com/workspace/drive/api/v3/manage-comments
- **Correction to a common assumption:** **the Google Docs API v1 has no comment model at all.** Verified against the live discovery document (`https://docs.googleapis.com/$discovery/rest?version=v1`, revision 20260722): the string `comment` appears **zero times**. `documents.get` accepts only `includeTabsContent` and `suggestionsViewMode`. The **Drive API v3 `comments` resource is the only comment surface.**
- **Anchoring:** the `anchor` field is documented verbatim as *"A region of the document represented as a JSON string."* **Its internal structure is not published.** The only shape shown anywhere in Google's docs is a Python sample: `{'region': {'kind': 'drive#commentRegion', 'line': N, 'rev': 'head'}}`. The full field set is **unverified — Google does not document it.** The docs themselves warn: *"anchors are immutable, and their position relative to the content of a document cannot be guaranteed between revisions."*
- **The one good idea:** `quotedFileContent {mimeType, value}` — the quoted text carried alongside the opaque anchor, as plain text. A resilient fallback for when the anchor rots. Plus `resolved` and `replies[]`.
- **Export:** Docs export to `.docx`, `.odt`, `.rtf`, `.pdf`, `.txt`, HTML-zip, `.epub`, `text/markdown`. **Whether comments survive export is not stated in any primary Google source — unverified.** Markdown/plain-text export structurally cannot carry them.
- **VERDICT: BORROW ONE IDEA, IGNORE THE FORMAT.** Take `quotedFileContent` (text quote as self-healing anchor). The anchor blob is undocumented and admits its own brittleness across revisions — a cautionary tale, not a model.
</google-docs>
</section-4-office-document-models>

---

<section-5-markdown-native-syntaxes>
# 5. CriticMarkup and Markdown-native annotation

<criticmarkup>
## CriticMarkup

- **What:** "Basic editorial change tracking in plain text files," syntax-compatible with Markdown, MultiMarkdown and HTML.
- **Link:** http://criticmarkup.com (**note: the site was unreachable at research time — ECONNREFUSED on 216.40.34.41**; details taken from the GitHub toolkit repo). https://github.com/CriticMarkup/CriticMarkup-toolkit
- **Licence:** **Apache-2.0.** Copyright 2013 Gabe Weatherhead and Erik Hess.
- **Syntax:**

  | Operation | Syntax |
  |---|---|
  | Addition | `{++ ++}` |
  | Deletion | `{-- --}` |
  | Substitution | `{~~ ~> ~~}` |
  | Comment | `{>> <<}` |
  | Highlight | `{== ==}{>> <<}` |

- **Maintenance:** 840 stars, 77 commits; **the repo shows no explicit maintenance statement and current activity could not be confirmed — treat as dormant (unverified).** The 2013 copyright and the dead website are not encouraging.
- **VERDICT: IGNORE AS A TRANSPORT, BORROW AS A DISPLAY/OUTPUT FORM.** In-band markup is the wrong choice for cloudreview's payload: it mutates the document (so offsets shift as annotations accumulate), it has no place for author, timestamp, screenshot, or audio, and it cannot express threads. **But** its operation vocabulary — addition / deletion / substitution / comment / highlight — is exactly the five-way split Plannotator independently reinvented (`INSERTION` / `DELETION` / `REPLACEMENT` / `COMMENT` / `GLOBAL_COMMENT`). Two independent designs landing on the same five verbs is a strong signal that **this is the right annotation taxonomy.** It may also be a useful *rendering* of the feedback when handing it to the LLM, since it is compact and self-explanatory.
</criticmarkup>
</section-5-markdown-native-syntaxes>

---

<section-6-embeddable-libraries>
# 6. Embeddable JS annotation libraries

<recogito-text-annotator>
## Recogito Text Annotator — the strongest embeddable candidate

- **Link:** https://github.com/recogito/text-annotator-js · npm `@recogito/text-annotator`
- **Licence:** **BSD-3-Clause** (verified: repo LICENSE metadata + npm package metadata). Commercially safe.
- **Library, embeddable.** One call — `createTextAnnotator(document.getElementById('content'))` — against a **plain, non-contenteditable div**. That is precisely our situation: agent renders Markdown → HTML → human annotates.
- **Anchoring (internal model):** character offsets + quote, deliberately simplified from W3C for performance:
  ```json
  { "id": "uuid", "bodies": [],
    "target": { "selector": [{ "quote": "Tell me, O muse", "start": 48, "end": 63 }],
                "creator": {...}, "created": "...", "updated": "..." } }
  ```
  `bodies` is explicitly the application-specific payload slot — our typed comment, voice transcript, or screenshot reference goes there.
- **W3C:** *"aligns closely with the W3C Web Annotation Data Model… with a few key differences,"* and ships a real adapter at `packages/text-annotator/src/model/w3c/` (`w3c-text-format-adapter.ts`, `w3c-text-annotation.ts`) defining proper `TextQuoteSelector` and `TextPositionSelector`. So conformant JSON on export is available.
- **Rendering — important:** two renderers, `renderer-css-highlight` (**CSS Custom Highlight API**) and `renderer-spans`. The CSS Highlight path paints highlights **without mutating the DOM**, so the published HTML stays byte-identical and offsets stay valid.
- **Readback:** `getAnnotations()`, `loadAnnotations(url)`, events `createAnnotation` / `updateAnnotation` / `deleteAnnotation` / `selectionChanged`.
- **Maintenance:** active — `@recogito/text-annotator` **4.2.5 published 2026-06-17**; repo pushed 2026-07-01. But only ~96 stars and effectively **one maintainer (bus factor ~1)**; BSD-3 means we could fork if that fails.
- **VERDICT: ADOPT (primary library candidate)** if we decide to support arbitrary text-range selection. Nothing else maintained, permissive, and embeddable matches this closely.
- **Siblings:** `@annotorious/annotorious` v3 (BSD-3, active, **images/regions only** — shelve for diagram annotation). `recogito/recogito-js` **archived 2025-09-30, deprecated**. `recogito/pdf-annotator-js` **archived 2026-04-30**. **Recogito Studio** (`recogito-client`/`recogito-server`) is **AGPL-3.0** — study the UX, **do not link against it**; AGPL is viral over a network service.
</recogito-text-annotator>

<editor-comment-systems>
## Editor comment plugins (ProseMirror / TipTap / Lexical / CKEditor / Milkdown)

All of these require adopting a **rich-text editor** the reviewer does not need — the human is annotating, not editing. Ideas only.

- **ProseMirror** — `prosemirror-collab` is **MIT** but the repo is **archived** (last push 2026-04-01; complete rather than abandoned) and has nothing to do with comments. The canonical comment reference is the official collab demo `ProseMirror/website` `src/collab/client/comment.js` (**MIT**, ~120 lines): comments are `Decoration.inline(from, to, {class:"comment"}, {comment})` in a `DecorationSet`, remapped on every transaction via `decos.map(tr.mapping, tr.doc)`, with an `unsentEvents()` queue shipping `{type:"create", id, from, to, text}`. **No `prosemirror-comments` package exists on npm (404 verified).** Nearest maintained relative: `@remirror/extension-annotation` (**MIT**, 3.0.2, 2025-08-02).
  **Marks vs decorations:** marks live in the document (in history, sync free via Yjs) but are schema-constrained and pollute exported content; decorations are view-layer, span multiple nodes, keep the document clean, but must be persisted and re-mapped manually. **For a review workflow where the reviewer must not mutate the agent's document, decorations are correct.** *Verdict: borrow the position-remapping technique; ignore the dependency.*
- **TipTap Comments** — `@tiptap-pro/extension-comments`. **PAID AND CLOSED**: verified **HTTP 404 on registry.npmjs.org**; published to Tiptap's private authenticated registry, requires a subscription, and requires Tiptap Cloud or their on-prem Document server. (Core `ueberdosis/tiptap` is MIT and very active — the MIT licence does **not** extend to Pro extensions.) *One idea worth stealing:* the **`inlineThread` (mark) / `blockThread` (node) duality** — the cleanest answer to "comment on a whole paragraph *and* on a phrase," which cloudreview will need. *Verdict: ignore the dependency, borrow the duality.*
- **Lexical CommentPlugin** — `facebook/lexical`, **MIT** (verified), very active (pushed 2026-07-29). Anchors via `MarkNode` (`$wrapSelectionInMarkNode(selection, isBackward, id)`); `MarkNode extends ElementNode` holding **`__ids: readonly string[]` — an array**, so overlapping comments accumulate ids and `registerNestedElementResolver` merges nested marks. `CommentStore` holds `Thread`/`Comment` and **stores the quote alongside**; sync uses a **separate `'comments'` Y.Doc** from the document doc. Caveat: it is **unpublished playground demo code** (~900 lines of React), not a package. *Verdict: borrow two ideas — `ids: string[]` for overlap, and the separate comments doc.*
- **CKEditor 5 Comments / Track Changes** — **COMMERCIAL.** Base editor is dual **GPL-2.0-or-later / commercial**; `@ckeditor/ckeditor5-comments` npm metadata reports `license: "SEE LICENSE IN LICENSE.md"` — a paid premium add-on. Anchors via CKEditor markers; exported HTML carries `<comment-start>`/`<comment-end>` tags inline and `data-comment-start-before`/`data-comment-end-after` on blocks. **Deliberately, only thread IDs go into the markup, never comment text** — their stated rationale is to prevent users editing comment content through the document data. *Verdict: ignore the dependency; **adopt that integrity rule** — the HTML is the anchor substrate, the JSON is the truth.*
- **Milkdown** — **MIT**, very active (11.7k stars, `@milkdown/core` 7.21.3, 2026-07-12). **It has no comment or annotation plugin.** Verified by enumerating `packages/plugins/` via the GitHub API: automd, block, clipboard, collab, cursor, diff, emoji, highlight (syntax/search, not annotation), history, indent, listener, prism, slash, streaming, tooltip, trailing, upload. *Verdict: ignore — right licence, wrong feature set.* (Aside: `plugin-diff` and `plugin-streaming` are interesting if we ever want the agent to stream a revised document into the same view.)
</editor-comment-systems>

<annotator-lineage>
## The Annotator.js lineage and the "highlight arbitrary HTML" libraries

- **Annotator.js** (`openannotation/annotator`) — **dual MIT / GPL-3.0** (verified from `LICENSE-MIT` + `LICENSE-GPL`; GitHub reports NOASSERTION because of the dual layout). Anchors via serialized **XPath ranges** (`xpath-range`), wraps matches in `<span class="annotator-hl">`. **No quote or fuzzy fallback — brittle.** **Effectively dead:** last npm publish `2.0.0-alpha.3` on **2015-07-03**, `package.json` still pins `"engines": {"node": ">=0.10 <0.12"}` and jQuery 1.x. A hobby revival is in progress (`REBOOT.md`/`REVIVAL-PLAN.md` added 2026-04) whose own text admits the repo barely builds; no new release. *Verdict: **ignore the code, borrow the decomposition*** — adder / editor / viewer / highlighter / storage / authz is the component split we will end up with. Its real successor is Hypothesis.
- **`apache/incubator-annotator`** — **Apache-2.0**, spec-faithful matchers, no UI. **Retired from the Incubator 2025-08-11**, archived. *Verdict: **vendor the code**, don't depend on it.*
- **`alienzhou/web-highlighter`** — MIT, 981 stars, but anchors via `{parentTagName, parentIndex, textOffset}` resolved as `getElementsByTagName(tag)[index]` — i.e. **"the Nth `<p>` in the document."** Any change in preceding element count silently re-anchors to the *wrong text*. Mutates the DOM. npm 0.7.4 (2021-06-24). *Verdict: **ignore** — this is precisely the failure mode we cannot afford.*
- **`julkue/mark.js`** — MIT, keyword/regex highlighting only; a renderer, not an annotator. No persistence, no selector model, no selection capture. npm 8.11.1 published **2018-01-11**, last real commit 2019. *Verdict: ignore — use the native **CSS Custom Highlight API** instead, no library and no DOM mutation.*
- **`timdown/rangy`** — MIT (from the LICENSE file; npm metadata omits it). A cross-browser Range/Selection shim whose reason to exist was old-IE normalization. Feature-frozen. *Verdict: ignore — modern `Range`, `Selection`, `CSS.highlights` cover this.*
- **`mir3z/texthighlighter`** — MIT, repo description literally begins **"-- NO LONGER MAINTAINED --"**, last push 2018-05-10. *Verdict: ignore.*
</annotator-lineage>
</section-6-embeddable-libraries>

---

<section-7-other-products>
# 7. Other doc/comment products surveyed

- **Notion comments** — proprietary. Comment `parent` may be **`page_id` or `block_id` only**; `discussion_id` groups threads. The docs state plainly: *"The public API can create comments attached to whole blocks with `parent.block_id`. It does not support creating a new discussion anchored to a selected range of text inside a block."* You can *reply* into a text-anchored discussion Notion's own UI created, but **resolved comments cannot be retrieved at all.** *Verdict: ignore, but note the cautionary tale — the capability we need is exactly the one they withhold from the API. It does validate block-level granularity as a legitimate product choice.*
- **Figma comments** — proprietary; **public OpenAPI spec** at https://github.com/figma/rest-api-spec. `Comment.client_meta` is a `oneOf` over `Vector` (canvas x/y), `FrameOffset` (`node_id` + `node_offset`), `Region`, `FrameOffsetRegion`. Threading via flat `parent_id`. *Verdict: ignore the spatial anchoring (wrong for flowing text); note that **`node_id` + offset** is the right instinct, and flat `parent_id` threading is simpler than Notion's `discussion_id` indirection.*
- **PDF annotations (ISO 32000-1:2008)** — §12.5.6.10 Text Markup Annotations, Table 179. `Subtype` ∈ `Highlight`/`Underline`/`Squiggly`/`StrikeOut`; **`QuadPoints`** = 8×n numbers, *"Each quadrilateral shall encompass a word or group of contiguous words."* Comment text in `Contents`; author in `T`. **Threading is native:** `IRT` (in-reply-to) + `RT` ∈ `R`|`Group`. **And so is disposition** (§12.5.6.3, Table 171): state models `Marked` (Marked/Unmarked) and **`Review` (Accepted / Rejected / Cancelled / Completed / None)**. *Verdict: ignore QuadPoints (geometric, useless for reflowing HTML); **steal the `Review` state model** — it is the most mature annotation-disposition vocabulary found anywhere in this survey and is directly actionable by an agent. A 40-year-old standard answers this better than any modern SaaS here.*
- **PDF.js** — `mozilla/pdf.js`, **Apache-2.0** (verified). `src/display/editor/highlight.js` defines `HighlightEditor extends AnnotationEditor`; `#serializeBoxes()` emits Acrobat-format quadPoints. It can **create and serialize**, not just render. *Verdict: ignore for our document type.*
- **HedgeDoc** — **AGPL-3.0** (verified). **Does not support inline comments.** Long-standing open requests (#657, #4351, #5450); maintainers said *not in 2.0, earliest 2.1*, with a stated design direction of **storing comments outside the document and using annotations**. 1.x is maintenance-only and accepts no feature PRs. *Verdict: ignore (also AGPL is a hazard for a hosted product) — but note that HedgeDoc's own unbuilt design independently converges on the OOXML/Review Board sidecar architecture. **Three independent designs landing on the same answer is a strong signal.***
- **HackMD** — commercial SaaS. Has commenting with a permission toggle (Forbidden / Owners / Signed-in / Everyone). **Whether it is per-paragraph, and whether any comment-read API exists: unverified.** Its AGPL-3.0 ancestor **CodiMD** does not carry the commercial comment feature.
- **utterances / giscus / Docusaurus / Docsify** — utterances (**MIT**, GitHub Issues-backed) and giscus (**MIT**, GitHub Discussions-backed; its LICENSE even carries utterances' 2018 copyright) map a page to a thread via `pathname` / `url` / `title` / `og:title` / specific term / discussion number. Docusaurus and Docsify are both **MIT** and have no native comment layer — these widgets are dropped in. **Granularity is strictly PER-PAGE, never per-paragraph** — a hard architectural limit, not a config gap. *Verdict: ignore as-is. **But the storage idea is genuinely good**: use GitHub Discussions as the comment database, write no backend, and get a machine-readable API for free. Could be applied at paragraph granularity via one discussion per document with the anchor encoded in the comment body — a cheap path to a working v1.*
- **Medium highlights** — closed source, no SDK. **Anchoring model unverified** (Medium stores content as structured Mobiledoc, so block-id + offset is plausible but unconfirmed). *Verdict: borrow the **UX** — select text → floating toolbar → margin marker → sidebar thread is the interaction reviewers now expect. The **structural** lesson matters more: because Medium owns a structured document model, anchors are block-id + offset rather than DOM XPath. **We are in the same position** — our agent authors the Markdown, so it can emit stable per-block ids.*
- **Genius / annotate.js** — closed source; **no annotator repo exists in the Genius GitHub org** (verified by enumerating its 30 most-recently-pushed repos — only Ruby/infra forks). The `genius.it` proxy-annotator peaked in 2016, drew backlash for annotating personal blogs without consent, and rotted without a formal shutdown. **Current status and anchoring model unverified.** *Verdict: ignore — and take the cautionary tale: proxy-and-rewrite-someone-else's-HTML is exactly the fragility we avoid by controlling the published page.*
</section-7-other-products>

---

<convergence>
# What the prior art agrees on

Five systems designed independently — OOXML (2006), PDF (1993/2008), Gerrit, Review Board, Hypothesis, Markloop, Plannotator — converge on the same five decisions. That convergence is the most reliable signal in this document.

1. **Anchor redundantly.** Never one selector. Stable block/element id **+** source range **+** quoted text (with occurrence index). Hypothesis stores three and falls back progressively; Markloop stores selector + quote + occurrence; Drive carries `quotedFileContent` next to its opaque anchor precisely because the anchor rots.
2. **Comment bodies live out-of-band**, in a sidecar keyed by anchor id. OOXML (`comments.xml`), CKEditor (ids-only in markup, by design, to stop users editing comment content through document data), Review Board (`extra_data`), and even HedgeDoc's unbuilt design all landed here.
3. **Draft → accumulate → single atomic publish.** GitHub pending reviews, Gerrit drafts, Reviewable's drafts panel. The agent wakes once with the complete set.
4. **A disposition vocabulary, not a resolved boolean.** PDF's `Review` state model (Accepted / Rejected / Cancelled / Completed / None) is the most mature; GitHub's `APPROVE`/`REQUEST_CHANGES`/`COMMENT` and Plannotator's `approved`/`annotated`/`dismissed` are the practical minimum.
5. **A five-verb annotation taxonomy.** CriticMarkup (2013) and Plannotator (2026) independently produced the same set: insertion, deletion, replacement, comment, document-level comment.

One more, from Review Board alone but non-obvious and correct: **record which surface the human was looking at** (`viewMode: 'source' | 'rendered'`). The same range means different things against the Markdown source and the rendered HTML.
</convergence>

---

<gaps>
# Where there is no prior art

- **Voice / dictated comments.** Nothing in this survey supports it. Not Hypothesis, not Plannotator, not Markloop, not any editor plugin. This is genuinely open ground.
- **Pasted screenshots as annotation bodies.** Only Plannotator does it, and crudely — a filesystem path (`[proposed-layout] /tmp/uploads/image-abc123.png`) inlined into a Markdown export. No prior art for a well-modelled image body that an agent can actually consume.
- **Questionnaire mode** (agent asks embedded questions, human answers in place). Only Markloop has this, and it is closed. The W3C model has a `questioning` motivation but no answer-binding.
- Both gaps sit on the *body* side of the annotation, not the *anchor* side. **The anchoring problem is solved prior art; the multimodal body is the unexplored part.**
</gaps>

---

<recommendation>
## Recommendation

**Build on these three, in this order:**

1. **Plannotator (Apache-2.0 / MIT) — read it end to end before designing anything.** It is cloudreview, minus hosting and minus voice. Its block-based anchor (`Block { id, type, content, startLine }` + `blockId` + preserved `originalText`), its five annotation types, its decision contract (`approved` / `annotated` / `dismissed`), and its choice to hand the LLM **human-readable Markdown rather than JSON** are all directly reusable and permissively licensed. Copying a working design is not a compromise here — it is the fastest path to a correct one. Be honest about the consequence: if cloudreview is not *hosted* (phone, non-technical stakeholder, no localhost) and does not do *voice*, it has no reason to exist alongside Plannotator.

2. **Hypothesis's multi-selector fallback algorithm (BSD-2-Clause), with Apache Annotator (Apache-2.0) as the vendorable clean-room implementation.** Persist block id **+** `TextPositionSelector` **+** `TextQuoteSelector` (`exact`/`prefix`/`suffix`, 32 chars each side) on every annotation, and resolve with the progressive fallback chain. Use the **W3C selector names and semantics** for legibility and future convertibility, but **not** JSON-LD, `@context`, IRIs, or the Web Annotation Protocol — that ceremony buys nothing when the consumer is an LLM reading JSON.

3. **GitHub's / Gerrit's draft-then-publish batching, plus Review Board's `MarkdownReviewUI` schema (MIT).** Annotations are private drafts until one atomic "Finish" event delivers the whole set with a top-level verdict. Steal Review Board's `{beginLineNum, endLineNum, viewMode}` — especially the **`viewMode` discriminator** — and Gerrit's `{start_line, start_character, end_line, end_character}` for sub-block precision.

**If we need a text-selection library rather than pure block-level clicking**, `@recogito/text-annotator` (BSD-3-Clause, v4.2.5, 2026-06-17) is the only maintained, permissive, embeddable option that works on plain non-contenteditable HTML — and its **CSS Custom Highlight API** renderer paints highlights *without mutating the DOM*, which keeps offsets valid. Accept the bus-factor-1 risk; BSD-3 means we can fork.

**The single biggest trap to avoid: anchoring to the rendered DOM.**

Every failure mode in this survey traces to the same root — anchoring to something the publisher did not control. web-highlighter resolves "the Nth `<p>` in the document" and silently re-anchors to the *wrong text* when a paragraph is inserted above. Annotator.js stores raw XPath and breaks on any structural change. Google's Drive anchor is so brittle its own documentation warns that position "cannot be guaranteed between revisions." Genius proxied and rewrote other people's HTML and rotted as the web changed.

**cloudreview does not have this problem unless it invents it.** The agent authors the Markdown, so the agent can emit stable per-block ids at publish time — and those ids can be *carried forward* when the agent republishes a revision. That is Medium's structural advantage, available to us for free. Anchor to `(stable-block-id, startOffset, endOffset, quote, occurrence)`; never to XPath, never to CSS selectors derived from the render, never to element position.

The corollary trap, and the one that will bite on the second document rather than the first: **the agent will republish a revised version, and annotations must survive it.** Design the version chain on day one (Markloop and Reviewable both treat this as first-class; Hypothesis, which does not control its documents, is stuck with orphans). If block ids are minted by the agent and preserved across revisions, re-anchoring is trivial; if they are minted by the renderer at publish time, every revision orphans every annotation.
</recommendation>

---

<verification-notes>
# Explicitly unverified

- Moat's anchoring model, export format, and pricing — moat.so returned HTTP 403 to direct fetch; all Moat detail here comes from search-result summaries.
- Whether Google Docs `.docx` export preserves comments — no primary Google source states this either way.
- The full internal structure of the Drive API `anchor` JSON string — Google does not publish it.
- CriticMarkup's current maintenance status — criticmarkup.com was unreachable (ECONNREFUSED); the GitHub repo carries no maintenance statement.
- Medium's highlight anchoring model — closed source, no primary source found.
- Genius web annotator's current live status and anchoring model — closed source, site not fetchable from this environment.
- HackMD's comment granularity and whether any comment-read API exists — not documented on the public features page.
- Reviewable's comment-export API — none documented publicly.
</verification-notes>
