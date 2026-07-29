# Agent → Human → Agent Round-Trip: Prior Art

<meta>
Research date: 2026-07-29
Question: How do existing systems get an AI agent to hand work to a human and receive a
structured answer back — and how do they make that round-trip token-efficient?
Method: primary sources (spec text, official docs, source repos) via WebSearch/WebFetch.
Anything I could not confirm from a primary source is explicitly marked **unverified**.
</meta>

<headline-findings>
Five things that change the design, up front:

1. **Claude Code supports MCP elicitation — including URL mode.** The official docs have a
   section "Respond to MCP elicitation requests" describing both *form mode* and *URL mode*
   ("Claude Code opens a browser URL for authentication or approval. Complete the flow in the
   browser, then confirm in the CLI"). URL-mode elicitation is almost exactly cloudreview's
   shape. Note this contradicts several secondary sources — see
   `<contradiction-claude-code-elicitation>`.

2. **URL-mode elicitation deliberately does not carry data back through the client.** The spec
   is explicit: `action: "accept"` means *the user consented to open the URL*, not that the
   interaction finished. The annotations must come back through the **server's own channel**
   and be returned as the eventual tool result. For cloudreview this is a feature, not a
   limitation — the server hosts the page, so it already has the annotations.

3. **MCP 2026-07-28 made this a breaking change.** Server-initiated requests are gone,
   replaced by *Multi Round-Trip Requests* (MRTR): the server returns an `InputRequiredResult`
   with an opaque `requestState`, and the **client retries the whole call**. Plan for retry
   semantics, not a held-open stream. Sampling and Roots are **deprecated** in the same release.

4. **The return trip has a hard ceiling.** Claude Code warns above **10,000 tokens** of MCP
   tool output and truncates at **25,000 tokens** by default (`MAX_MCP_OUTPUT_TOKENS`). A
   naive "return the annotated document" design will hit this on a long review.

5. **The biggest token win is refusing to echo the document back.** The agent wrote the doc; it
   is already in context. A referential return (`anchor-id → comment`) costs roughly
   **~25 tokens per annotation** versus ~8,000+ tokens to re-send a 6,000-word document. That
   is an order-of-magnitude difference and it is the single most important design decision.

6. **No existing system models "many annotations anchored within one long artifact."** Fourteen
   systems surveyed; every one models a *point decision on a single item*. The annotation schema
   is cloudreview's to invent regardless of transport — that is the product, and the transport
   is a solved problem. See `<the-gap-nobody-fills>`.
</headline-findings>

<verdict-table>
| Candidate | Kind | Verdict | One-line why |
|---|---|---|---|
| MCP elicitation, URL mode | Protocol to speak | **Adopt** | Native Claude Code support; purpose-built for "go do this in a browser" |
| MCP MRTR (`InputRequiredResult`) | Protocol to speak | **Adopt** | The mandatory 2026-07-28 mechanism for any elicitation |
| MCP elicitation, form mode | Protocol to speak | **Borrow (fallback)** | Flat primitives only — cannot express 40 annotations |
| MCP sampling | Protocol | **Ignore** | Deprecated 2026-07-28; Claude Code never supported it |
| MCP resources / prompts | Protocol to speak | **Borrow** | Nice for `@`-mention of a past review; not the main path |
| MCP Apps (SEP-1865) | Protocol | **Ignore (for Claude Code)** | Inline iframe UI; a terminal cannot render it |
| Claude Code `Elicitation` hook | Integration point | **Borrow** | Enables headless/CI auto-answer; good escape hatch |
| Claude Code `AskUserQuestion` | Built-in tool | **Borrow (idea)** | Proves structured-choice UX; wrong shape for long docs |
| Claude Code Skills / slash commands | Integration point | **Adopt** | The natural `/cloudreview` entry point |
| **Trigger.dev wait tokens** | Library + pattern | **Borrow (heavily)** | Closest analogue; arbitrary JSON payload + browser-safe scoped token |
| **AG-UI `responseSchema`** | Draft protocol | **Borrow (the idea)** | Only mechanism where the agent declares the answer's schema; draft, unshipped |
| HumanLayer | Library + API | **Ignore (deprecated)** | Repo README self-declares deprecated; PyPI frozen at 0.7.9 (Jun 2025) |
| LangGraph `interrupt()` | Library | **Borrow (idea)** | Best-articulated pause/resume model; wrong ecosystem |
| LangSmith Annotation Queues | Product | **Ignore** | Offline eval tooling; cannot resume a paused run |
| Agent Inbox | Library | **Borrow (schema flags)** | Right silhouette, but one verdict per interrupt; LangGraph-locked |
| CrewAI human input | Library | **Ignore** | Plain-string feedback; OSS path not durable |
| AutoGen `UserProxyAgent` | Library | **Ignore** | `Awaitable[str]`, not durable; docs say terminate rather than wait |
| Temporal Signals / Updates | Library | **Borrow (semantics)** | Strongest durability; `@workflow.update` = typed in, typed ack out |
| OpenAI Agents SDK `RunState` | Library | **Borrow (mechanic)** | Serialize-pause-resume across processes; approve/reject too coarse |
| Inngest `step.waitForEvent()` | Library | **Ignore (weaker sibling)** | Same shape as Trigger.dev, weaker correlation, no browser credential |
| Braintrust human review | Product | **Ignore** | No pause primitive; no documented read-back API |
| Humanloop | Product | **Ignore** | Acquired by Anthropic 2025, platform sunsetting |
| GitHub PR review API | Idea to borrow | **Borrow** | The canonical batched-annotation payload shape |
| W3C Web Annotation selectors | Standard to borrow | **Adopt (subset)** | The correct answer to anchor stability |
| Hypothesis fuzzy anchoring | Algorithm to borrow | **Adopt** | Proven 4-strategy re-anchoring fallback |
| LSP position encoding | Idea to borrow | **Borrow (cautionary)** | Shows why raw line/char offsets are a trap |
| JSON Patch (RFC 6902) | Format | **Ignore for comments / Borrow for edits** | Wrong shape for commentary; fine for text edits |
| Magic links / signed URLs | Auth pattern | **Adopt** | Lightest safe thing for a one-session page |
</verdict-table>

<mcp-protocol>

<mcp-elicitation-current-spec>
**Source:** https://modelcontextprotocol.io/specification/2026-07-28/client/elicitation
(also the superseded https://modelcontextprotocol.io/specification/2025-06-18/client/elicitation)

Elicitation is the MCP capability by which **a server asks a human a question through the
client**. As of the `2026-07-28` spec it has two modes.

**Capability declaration.** Clients declare it per-request in `_meta`:

```json
{
  "_meta": {
    "io.modelcontextprotocol/clientCapabilities": {
      "elicitation": { "form": {}, "url": {} }
    }
  }
}
```

An empty `"elicitation": {}` means **form mode only** (backwards compatibility). Servers
**MUST NOT** send a mode the client did not declare.

**Form mode** collects structured data through the client UI. Its schema is deliberately
crippled: *"limited to flat objects with primitive properties only"* — string, number/integer,
boolean, enum (single- and multi-select). String formats are limited to `email`, `uri`, `date`,
`date-time`. The spec states plainly:

> Note that complex nested structures, arrays of objects (beyond enums), and other advanced
> JSON Schema features are intentionally not supported to simplify client user experience.

**This is the decisive limit for cloudreview.** You cannot express "40 annotations, each with
an anchor id, a comment body, and an optional image" as a form-mode `requestedSchema`. Form
mode can carry a *handful of scalar answers*, nothing more.

**URL mode** (introduced in `2025-11-25`) is the interesting one:

```json
{
  "method": "elicitation/create",
  "params": {
    "mode": "url",
    "url": "https://mcp.example.com/ui/set_api_key",
    "message": "Please provide your API key to continue."
  }
}
```

Client result:

```json
{ "action": "accept" }
```

And the spec is emphatic about what that means:

> The response with `action: "accept"` indicates that the user has consented to the
> interaction. It does not mean that the interaction is complete. The interaction occurs out
> of band and the client is not directly informed of the outcome. When the client retries
> the original request, the server determines from the echoed `requestState` (or its own
> stored state) whether the out-of-band interaction has completed, and either returns the
> final result or responds with another `InputRequiredResult`.

Response actions are `accept` / `decline` / `cancel` in both modes.

**Verdict: adopt.** URL mode is the mechanism cloudreview should speak. The data path
(annotations) stays server-side and lands in the tool result; only the URL and a consent
signal cross the client.
</mcp-elicitation-current-spec>

<mcp-mrtr>
**Source:** https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/mrtr

MRTR is how elicitation is *delivered* as of 2026-07-28, and it is a breaking change:

> Servers **MUST** send server-to-client requests (such as `roots/list`,
> `sampling/createMessage`, or `elicitation/create`) using the MRTR pattern. The previous
> pattern of server-initiated requests is no longer supported. This is a breaking change.

The flow: client calls `tools/call` (id 1) → server replies with an `InputRequiredResult`
carrying `inputRequests` (a **map** of server-assigned keys → request objects) and an opaque
`requestState` → client gathers input → client **retries the original call** (id 2) with
`inputResponses` and the echoed `requestState`.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "resultType": "input_required",
    "inputRequests": {
      "github_login": {
        "method": "elicitation/create",
        "params": { "mode": "form", "message": "...", "requestedSchema": { } }
      }
    },
    "requestState": "AEAD-protected blob"
  }
}
```

Rules that matter for implementation:

- `InputRequiredResult` is allowed **only** on `tools/call`, `prompts/get`, `resources/read`.
- The JSON-RPC `id` **MUST** differ between the initial call and the retry.
- Clients **MUST** echo `requestState` byte-for-byte and **MUST NOT** parse it.
- Servers **MUST** treat `requestState` as attacker-controlled and integrity-protect it
  (HMAC/AEAD), and **SHOULD** embed the authenticated principal, a short TTL, and an
  identifier for the originating request to bound replay.
- Servers **MUST NOT** assume the client will retry at all.
- A server may return `InputRequiredResult` **repeatedly** — this is the documented way to
  poll for an out-of-band interaction that has not finished yet.

The design goal is statelessness: *"without requiring a shared storage layer across server
instances or requiring stateful load balancing."*

**Verdict: adopt.** Not optional — it is the only conformant way to elicit on the current spec.

**Design consequence for cloudreview:** the "wait for the human" loop is naturally expressed as
*repeat `InputRequiredResult` until the review is submitted*. First response: URL-mode
elicitation pointing at the review page. Subsequent retries: either the final result (if the
human clicked Finish) or another `InputRequiredResult` with no `inputRequests` (which the
client **MAY** retry immediately). That gives a clean poll loop without holding a socket open.
</mcp-mrtr>

<mcp-deprecations>
**Source:** https://modelcontextprotocol.io/docs/learn/client-concepts

- **Sampling** — *"deprecated as of protocol version `2026-07-28`... New implementations should
  integrate directly with LLM provider APIs instead."* Claude Code never supported it anyway
  (see the capability matrix below). **Ignore.**
- **Roots** — *"deprecated as of protocol version `2026-07-28` and scheduled for removal."*
  **Ignore.**

Do not build on either.
</mcp-deprecations>

<mcp-resources-and-prompts>
Claude Code exposes MCP **resources** via `@`-mention (`@server:protocol://resource/path`, e.g.
`@github:issue://123`) and MCP **prompts** as slash commands. Claude Code also supports
`list_changed` notifications so a server can refresh its advertised tools/prompts/resources
live.

Useful secondary surface for cloudreview: expose past reviews as resources
(`@cloudreview:review://2026-07-29-auth-spec`) so the human or agent can pull an old review back
into context on demand rather than eagerly. **Borrow** — not the main path, cheap to add later.
</mcp-resources-and-prompts>

<mcp-apps>
**Source:** https://modelcontextprotocol.io/seps/1865-mcp-apps-interactive-user-interfaces-for-mcp

MCP Apps (SEP-1865, co-authored by Anthropic and OpenAI) lets a server register a `ui://`
resource containing HTML, reference it from a tool via `_meta.ui.resourceUri`, and have the host
render it in a **sandboxed iframe** communicating over JSON-RPC-over-`postMessage`. Reported
adopters are Claude Desktop, ChatGPT enterprise tiers, and Cursor.

**Verdict: ignore, for the Claude Code target.** Claude Code is a terminal; it cannot render an
iframe, and its MCP documentation makes no mention of MCP Apps or `ui://` (I grepped the full
page). If cloudreview later targets Claude Desktop or Cursor, MCP Apps becomes the better
in-client surface. **Unverified:** whether Claude Code has any MCP Apps support at all — absence
from the docs is strong evidence but not proof.
</mcp-apps>

</mcp-protocol>

<claude-code-integration>

<contradiction-claude-code-elicitation>
This needs stating clearly because the sources disagree.

**Evidence that Claude Code DOES support elicitation (primary, current):**
The official Claude Code MCP documentation (https://code.claude.com/docs/en/mcp) contains a
section titled *"Respond to MCP elicitation requests"*:

> MCP servers can request structured input from you mid-task using elicitation. When a server
> needs information it can't get on its own, Claude Code displays an interactive dialog and
> passes your response back to the server. No configuration is required on your side:
> elicitation dialogs appear automatically when a server requests them.
>
> Servers can request input in two ways:
> - **Form mode**: Claude Code shows a dialog with form fields defined by the server...
> - **URL mode**: Claude Code opens a browser URL for authentication or approval. Complete the
>   flow in the browser, then confirm in the CLI.

The same page separately references elicitation in its backgrounding rules ("A call waiting on
an open elicitation dialog isn't backgrounded while the dialog is open"), and the hooks
documentation defines `Elicitation` and `ElicitationResult` hook events. Three independent
places in the official docs — this is solid.

**Evidence that it does NOT (secondary, stale):**
- `github.com/apify/mcp-client-capabilities` lists Claude Code as Elicitation ❌ (form) ❌ (url).
- `anthropics/claude-code` issue #2799 (opened 2025-07-01) and #7108 (opened 2025-09-04,
  closed as *not planned / duplicate*) are feature requests for elicitation.

**Resolution:** the official documentation is current and specific; the capability matrix and
the 2025 issues predate the feature. I am treating elicitation (form + URL) as **supported in
current Claude Code**. **Action item: verify empirically before committing the architecture** —
run `github.com/mcp-use/mcp-elicitation-demo` against the installed Claude Code version. That
repo is a minimal server built precisely to test form-mode and URL-mode elicitation support
(it does not itself publish a client matrix).

**Unverified:** whether current Claude Code implements the *2026-07-28 MRTR* delivery of
elicitation or the older server-initiated `elicitation/create`. Anthropic's blog
("MCP 2026-07-28 spec: stateless core, coming to Claude") says only *"Support is being rolled
out across Claude products soon"* without naming Claude Code or a date. **Build against the SDK
rather than hand-rolling JSON-RPC**, so this is the SDK's problem and not yours.
</contradiction-claude-code-elicitation>

<claude-code-operational-limits>
These are hard numbers from https://code.claude.com/docs/en/mcp and they constrain the design.

**Output size — the binding constraint on the return trip:**
> Claude Code displays a warning when MCP tool output exceeds 10,000 tokens and limits output
> to 25,000 tokens by default. To raise the limit, set the `MAX_MCP_OUTPUT_TOKENS` environment
> variable...; the warning threshold is fixed.

A tool may instead declare `anthropic/maxResultSizeChars`, which takes precedence for text
content. **Budget the annotation payload to stay under 10,000 tokens** to avoid even the
warning. See `<token-efficiency>`.

**Timeouts — the binding constraint on how long a human may take:**

| Timer | Default | Notes |
|---|---|---|
| Wall-clock per tool call | `MCP_TOOL_TIMEOUT`, **~28 hours** when unset | Per-server `timeout` in `.mcp.json` overrides. Progress notifications do **not** extend it. |
| Idle timeout | **5 min** (HTTP/SSE/WebSocket/connector), **30 min** (stdio) | Aborts if the server sends *no response and no progress notification* in the window. `CLAUDE_CODE_MCP_TOOL_IDLE_TIMEOUT`; `0` disables. |
| Per-request (first byte), HTTP/SSE/connector only | **60 s** | Raised by setting per-server `timeout` or `MCP_TOOL_TIMEOUT` to ≥60 s. Stdio/WebSocket exempt. |
| Auto-backgrounding | **2 min** | Main-conversation calls move to a background task; Claude keeps working and gets a task notification later. `CLAUDE_CODE_MCP_AUTO_BACKGROUND_MS`. |

Three consequences:

1. **A remote HTTP cloudreview server must emit progress notifications at least every ~5
   minutes** while the human reviews, or the call is aborted for idleness. This is the single
   most likely production failure mode.
2. **Auto-backgrounding is a gift.** After 2 minutes the review call becomes a background task,
   Claude Code carries on, and the result arrives as a notification. The human is not blocking
   the session. Caveat: *subagent* calls are never backgrounded, and background tasks do not
   survive exiting the session.
3. **Elicitation dialogs are explicitly excluded from backgrounding** while open — Claude Code
   "defers the move until the dialog closes." So the consent dialog blocks briefly; the long
   wait afterwards does not.

With MRTR's retry model, points 1 and 3 mostly dissolve: each round trip is short, and the
waiting is expressed as repeated `InputRequiredResult` rather than one long-held call. This is
another argument for MRTR over a held-open call.
</claude-code-operational-limits>

<claude-code-hooks>
**Source:** https://code.claude.com/docs/en/hooks

Two directly relevant events:

- **`Elicitation`** — fires when an MCP server requests user input. Matched by **MCP server
  name**. Receives `session_id`, `server_name`, `tool_name`, `elicitation_id`, `form_fields[]`.
  Returns `hookSpecificOutput` with `action: accept|decline|cancel` and a `content` object to
  **auto-respond without showing a dialog**.
- **`ElicitationResult`** — fires after the user responds, and can **validate or transform** the
  response before it goes back to the server.

**Verdict: borrow.** Not the primary mechanism, but two real uses: (a) headless/CI runs where
no human exists — a hook auto-declines or supplies a canned response so pipelines don't hang;
(b) local policy, e.g. auto-accepting the "open the review URL" consent for a trusted server so
the human isn't double-prompted.
</claude-code-hooks>

<claude-code-askuserquestion>
Claude Code ships a built-in `AskUserQuestion` tool: 1–8 questions per call, 2–4 options each,
`{label, description}` option objects, `multiSelect` boolean, an auto-added "Other…" free-text
escape, and a side-by-side preview panel.

**Verdict: borrow the idea, do not try to use it.** It is the in-terminal analogue of what
cloudreview does out-of-band, and it validates the core premise (structured choices beat
free-form prose for both the human and the token budget). But it is a *built-in* tool an MCP
server cannot invoke, and its shape — a handful of multiple-choice questions — is exactly the
shape that stops working at "40 annotations on a 6,000-word document," which is cloudreview's
reason to exist. Worth borrowing directly: **offering constrained choices where possible**
(`accept` / `revise` / `reject` per section) so most annotations cost ~3 tokens instead of ~25.

**Note (secondary sources only, mark unverified):** the option schema details above come from
community documentation and third-party reimplementations, not an official Anthropic tool-schema
page. The tool's existence and general behaviour are well attested; exact field names are
**unverified**.
</claude-code-askuserquestion>

<claude-code-skills-and-commands>
**Source:** https://code.claude.com/docs/en/slash-commands (now titled "Extend Claude with skills")

> **Custom commands have been merged into skills.** A file at `.claude/commands/deploy.md` and
> a skill at `.claude/skills/deploy/SKILL.md` both create `/deploy` and work the same way.

Skills follow the Agent Skills open standard (agentskills.io); Claude Code adds invocation
control, subagent execution, and dynamic context injection. MCP **prompts** also surface as
slash commands.

**Verdict: adopt as the entry point.** Ship a `cloudreview` skill that tells the agent *when*
to publish a document for review and *how to interpret* the returned annotation format. This is
where the return-format contract should be documented — it costs nothing until invoked, and it
is what turns a raw tool into a workflow. **Output styles** are "stored system prompts" and are
not relevant here.
</claude-code-skills-and-commands>

<mcp-vs-http-verdict>
**Both, layered — but the MCP server is the thin part.**

Reasons MCP is the right *control* surface for Claude Code:
- Native discovery and invocation, no bespoke CLI or credential plumbing for the user.
- URL-mode elicitation gives a spec-blessed, consent-gated way to send the human to a browser,
  with the client rendering the domain and requiring explicit approval.
- Auto-backgrounding means a long human review does not block the session.
- OAuth 2.0 is supported natively for remote HTTP servers, including dynamic headers via
  `headersHelper`.

Reasons the HTTP API must exist anyway:
- The review page itself is a web app; it needs a real backend regardless.
- URL-mode elicitation *requires* the payload to travel out-of-band — the server must own that
  channel by design.
- Non-Claude-Code consumers (CI, other agents, the Agent SDK, a plain `curl`) need a way in.
- MCP is young and moving fast (a breaking change in the current release). An HTTP core with a
  thin MCP adapter is the resilient factoring.

So: **HTTP API is the product; the MCP server is a ~200-line adapter over it.** Do not put
business logic in the MCP layer.
</mcp-vs-http-verdict>

</claude-code-integration>

<token-efficiency>

This is the part most prior art gets wrong, so it is worth being concrete.

<the-core-insight>
**The agent already has the document in context. Any byte of the document that comes back is
waste.** Every system surveyed that returns "the annotated artifact" pays this cost; the ones
that return *references plus deltas* do not.

Design rule: **the return payload should be referential, not reproductive.**
</the-core-insight>

<quantified-comparison>
Assume a realistic case: a 6,000-word review document (~8,000 tokens) with 60 addressable
sections, of which the human annotates 40. Average comment ~18 words (~24 tokens).

Estimates below use the standard ~0.75 words/token heuristic and count JSON punctuation
explicitly. Treat as **order-of-magnitude estimates, not measurements.**

| Strategy | Payload | Est. tokens | vs. best |
|---|---|---|---|
| A. Return whole annotated document | full text + inline comments | ~8,000 + ~1,000 = **~9,000** | 9× |
| B. Return whole doc as unified diff (human edited 10 sections) | 10 hunks × (header + 3 lines context each side + changes) | **~1,100** | 1.4× |
| C. JSON array of `{anchor, comment}` objects | `{"a":"q3","c":"..."}` ×40 | ~40 × 34 = **~1,360** | 1.7× |
| D. **Compact line format** `anchor > comment` | `q3 > too vague, split this` ×40 | ~40 × 27 = **~1,080** | 1.05× |
| E. D + verdict shorthand for unchanged sections | 30 comments + 10 × `q4 ok` | ~30×27 + 10×4 = **~850** | 1.0× |

**Per-annotation cost, format overhead only (excluding the comment text itself):**

| Format | Overhead/annotation |
|---|---|
| JSON object in an array | ~10–12 tokens |
| YAML mapping entry | ~5–7 tokens |
| `anchor > comment` line | **~3–4 tokens** |
| TSV | ~2–3 tokens |

**Conclusions:**

1. Re-sending the document (A) is the only genuinely bad option — and it is the obvious one.
   Avoiding it is worth ~8× on its own; everything after that is a rounding error by comparison.
2. **JSON costs ~25–30% more than a compact line format** for this payload, purely in braces,
   quotes, and repeated key names. For 40 annotations that is ~300 tokens — real but modest.
   Trade it deliberately: JSON is more robustly parseable and less likely to be mangled if a
   comment contains the delimiter. **Recommendation: compact line format with a JSON escape
   hatch** — plain lines for simple comments, and a JSON block only for annotations that carry
   structure (images, ranges, multi-line code).
3. **The cheapest annotation is one with no prose.** Encourage per-section verdicts
   (`ok` / `cut` / `expand`) — ~3–4 tokens each versus ~27. On a questionnaire where the human
   accepts most sections, this dominates the savings. This is the `AskUserQuestion` lesson
   applied to the return trip.
4. **Silence must be free.** Un-annotated sections must cost **zero** tokens. Never emit
   `q12: (no comment)` — 40 annotations out of 60 sections should transmit 40 records, not 60.
</quantified-comparison>

<how-github-does-it>
**Source:** https://docs.github.com/en/rest/pulls/reviews

GitHub's batched review is the canonical shape and it is worth copying almost exactly:

```json
{
  "commit_id": "ecdd80bb57125d7ba9641ffaa4d7d2c19d3f3091",
  "body": "This is close to perfect! Please address the suggested inline change.",
  "event": "REQUEST_CHANGES",
  "comments": [
    { "path": "file.md", "position": 6, "body": "Please add more information here." }
  ]
}
```

What to copy:
- **One call carries the whole review.** Not 40 round trips — one batch, submitted atomically
  when the human clicks Finish. Exactly cloudreview's model.
- **An overall verdict alongside per-item comments** (`APPROVE` / `REQUEST_CHANGES` / `COMMENT`),
  plus a review-level `body`. cloudreview should have the same two-level structure: a global
  disposition plus anchored annotations.
- **A pending state**: leaving `event` blank creates a draft review submitted later. That is
  the "human is still working" state, and it maps onto MRTR's repeat-`InputRequiredResult` loop.
- **`commit_id` pins the review to a document version.** cloudreview needs the exact analogue:
  a document version/content hash on every returned annotation set, so the agent can detect
  that it edited the doc while the human was reviewing it.

What **not** to copy: `position` (offset from the first `@@` hunk header) is deprecated even at
GitHub, and `line`/`side`/`start_line` are still diff-coordinate based. Both are unstable under
edits — see `<anchor-stability>`.
</how-github-does-it>

<json-patch-and-diffs>
**Sources:** RFC 6902; "JSON Whisperer: Efficient JSON Editing with LLMs"
(https://arxiv.org/html/2510.04717v1, EMNLP 2025 Industry Track)

Reported findings: patch generation reduces token usage by **~31%** versus full regeneration
while keeping edit quality within 5%; at 100 entities, RFC 6902 patching used **~5× fewer
tokens** than full regeneration.

The paper's most useful result for cloudreview is a *failure* mode, not the savings:

> Two key challenges in patch-based editing are that LLMs often miss related updates when
> generating isolated patches, and array manipulations require tracking index shifts across
> operations, which LLMs handle poorly.

Their fix (EASE — "Explicitly Addressed Sequence Encoding") is to **transform arrays into
dictionaries with stable keys, eliminating index arithmetic**. That is precisely the argument
for **stable anchor ids over line numbers or array indices**, arrived at independently.

**Verdict: ignore JSON Patch for annotations, borrow it for edits.** A comment is not a mutation
of the document — modelling `{"op":"add","path":"/sections/3/comments/-","value":{...}}` is
pure ceremony over `q3 > comment`. But if cloudreview later lets the human *rewrite* a passage,
a unified diff or a JSON Patch over the section is the right return for that specific case.
</json-patch-and-diffs>

<lsp-cautionary-tale>
**Source:** https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/

LSP positions are `{line, character}` zero-based offsets. The instructive part is the mess
around encoding: until 3.17 the character offset was **always UTF-16 code units**, so in the
string `a𐐀b` the offset of `b` is **3**, not 2. LSP 3.17 added negotiation
(`general.positionEncodings` client capability, `capabilities.positionEncoding` in the
initialize result), and *"the only mandatory encoding is UTF-16"* for backwards compatibility.

Two lessons, both cautionary:

1. **Character-offset anchoring drags in an encoding negotiation problem.** A browser
   (JS strings = UTF-16), a Python/Rust backend (UTF-8), and an LLM (tokens) will disagree
   about offsets over emoji, CJK, and accented text. If you use offsets at all, **specify the
   encoding explicitly** and normalise at one boundary.
2. LSP gets away with positional coordinates only because it pairs them with a **versioned
   document identifier** and a stream of `TextDocumentContentChangeEvent`s keeping both sides
   in lockstep. cloudreview has no such live sync — the human's browser and the agent's context
   drift independently. **So positions alone will not work; you need content-based anchors.**
</lsp-cautionary-tale>

</token-efficiency>

<anchor-stability>

The question: if the agent regenerates or edits the document, how do annotations survive?

<w3c-web-annotation>
**Source:** https://www.w3.org/TR/annotation-model/ (W3C Recommendation)

The standard defines complementary Selectors:

- **`TextQuoteSelector`** — `exact` text plus optional `prefix`/`suffix` context:
  ```json
  { "type": "TextQuoteSelector", "exact": "anotation",
    "prefix": "this is an ", "suffix": " that has some" }
  ```
- **`TextPositionSelector`** — `start` (inclusive) / `end` (exclusive) character positions:
  ```json
  { "type": "TextPositionSelector", "start": 412, "end": 795 }
  ```
- **`RangeSelector`** — a `startSelector` + `endSelector` pair for selections crossing
  structural boundaries.
- **`CssSelector`**, **`FragmentSelector`**, **`DataPositionSelector`** — DOM path,
  media-fragment, and byte-offset variants.

The key guidance is redundancy:

> Multiple Selectors can be given to describe the same Segment in different ways in order to
> maximize the chances that it will be discoverable later.

Consumers **MUST** pick one if they disagree.

**Verdict: adopt a subset.** Do not implement JSON-LD or the full data model — that is
overkill. Do adopt the **core idea**: every annotation carries *several* independent ways to
find its target, tried in order.
</w3c-web-annotation>

<hypothesis-fuzzy-anchoring>
**Source:** https://web.hypothes.is/blog/fuzzy-anchoring/

Hypothesis is the production-hardened implementation of the above. It stores **three** selectors
per annotation and tries **four** strategies in order:

1. XPath/range selector against the DOM.
2. If the structure changed: text-position selector (character offset) over normalised content.
3. Fuzzy match on the **quote plus ~32 characters of context on each side**.
4. Plain fuzzy match on the quote alone.

Matching is verbatim → deterministic normalisation (smart quotes, whitespace) → **Levenshtein
distance within ~5% tolerance**.

**Verdict: adopt this algorithm.** It is the best-tested answer to exactly cloudreview's problem
and the numbers (32 chars context, 5% edit distance) are sensible defaults to start from.
</hypothesis-fuzzy-anchoring>

<recommended-anchor-scheme>
cloudreview has an advantage Hypothesis does not: **the agent authors the document**, so it can
mint anchors up front instead of reverse-engineering them.

Layered scheme, cheapest and most stable first:

1. **Explicit agent-minted ids — primary.** The agent emits a stable id per addressable block
   when it writes the doc (`<!-- cr:q3 -->`, or a heading suffix `{#q3}`). Ids are *semantic and
   short* (`q3`, `auth-strategy`) — 1–3 tokens, self-describing even if the agent's context was
   compacted, and **stable across regeneration if the agent reuses them**. This is the same
   trick as EASE's "arrays → dictionaries with stable keys."
2. **Content hash — verification.** Store a hash of each block's normalised text. On return,
   a mismatch tells the agent *"you edited this section while the human was commenting on it"* —
   which is information the agent needs, not an error to hide.
3. **Quote + context — repair.** Store `exact` + 32 chars of `prefix`/`suffix`. When an id
   disappears (agent rewrote the doc structurally), re-anchor by fuzzy match per Hypothesis.
4. **Document version — the outer guard.** Every review is bound to a document version/content
   hash, exactly as GitHub binds a review to `commit_id`.

**Do not use line numbers or character offsets as the primary anchor.** They break on any
insertion above the target, and they inherit the LSP encoding problem.

**On CRDTs:** Peritext (Ink & Switch) and Loro solve anchoring properly — Peritext gives each
character two anchor positions (before/after) so formatting spans survive concurrent edits, and
Loro uses "style anchor" control characters. Even Peritext has known edge cases around
tombstones. **Verdict: ignore.** A CRDT is the right answer for *concurrent multi-writer*
editing. cloudreview is one agent writing then one human annotating — sequential, single-writer
at each phase. Adopting a CRDT here would be a large complexity import for a problem you do not
have. Revisit only if multiple humans must annotate the same doc simultaneously.
</recommended-anchor-scheme>

</anchor-stability>

<auth>

Requirement: a page on the public internet, one human, one review session, must be genuinely
safe, and must not make the human do busywork.

<magic-links-and-signed-urls>
**Sources:** consolidated from current practitioner guidance (MojoAuth, SuperTokens, Ping
Identity, and others). These are **secondary sources** — the numeric recommendations below are
industry convergence, not a normative standard.

Consensus properties of a safe magic link / signed URL:

- **≥128 bits of entropy** from a CSPRNG.
- **Single-use** — invalidated on first use regardless of expiry, to prevent replay.
- **Short TTL** — convergent guidance is 5–15 minutes for login links; up to 30 for recovery.
- **Bound to the requesting device/session** where possible.
- Signed or stored server-side, never a guessable identifier.

**The TTL guidance does not transfer directly.** Those numbers assume a link sitting in an inbox
that the user clicks within minutes. A cloudreview link is handed to a human who may legitimately
spend an hour annotating. So split the two concerns:

- **Entry token**: short-lived (~15 min), single-use, high entropy. It exists only to get the
  human from the terminal into the browser.
- **Session**: on redemption, set a normal `HttpOnly; Secure; SameSite=Lax` session cookie
  scoped to that one review, with a longer idle-based lifetime (hours). The long-lived thing is
  the cookie, not the URL.

This also satisfies the MCP spec's phishing mitigation, which requires the server to verify that
whoever opens the URL is the user the elicitation was minted for.
</magic-links-and-signed-urls>

<mcp-url-mode-security-requirements>
These are **normative MUSTs** from the 2026-07-28 elicitation spec and they are cheap to comply
with, so comply:

Servers:
1. **MUST NOT** put sensitive user information (credentials, PII) in the elicitation URL.
2. **MUST NOT** provide a URL that is **pre-authenticated** to a protected resource — "the URL
   could be used to impersonate the user by a malicious client."
3. **SHOULD** use HTTPS outside development.

Clients (what Claude Code will do to you): MUST NOT pre-fetch the URL, MUST NOT open it without
explicit consent, MUST show the full URL for examination, MUST open it so the client/LLM cannot
inspect content or input, SHOULD highlight the domain, SHOULD warn on Punycode.

**Requirement #2 is a genuine design constraint and easy to get wrong.** The instinct is to mint
`https://cloudreview.app/r/<secret-token>` and be done. The spec forbids a URL that *is* the
credential. The compliant pattern is the one the spec itself describes for OAuth: send the user
to a **connect route** that establishes identity (session cookie, or an interactive sign-in),
verifies it matches the principal the elicitation was minted for, and only then grants access to
the review. The spec's stated phishing scenario is exactly this: Alice triggers the elicitation,
tricks Bob into opening the URL, and Bob's actions get bound to Alice's identity.

Also relevant: `requestState` **MUST** be integrity-protected (HMAC/AEAD) and **SHOULD** embed
the authenticated principal, a TTL, and an originating-request identifier.
</mcp-url-mode-security-requirements>

<auth-recommendation>
**Lightest thing that is actually safe, in order of increasing effort:**

1. **Local-only, no auth (development / single-user, and genuinely defensible).** Bind the
   review server to `127.0.0.1` on an ephemeral port. Nothing is on the public internet, so
   there is nothing to authenticate. **If cloudreview's real use case is "one developer, one
   machine," this is the correct answer and everything below is over-engineering.** Start here.
2. **Hosted, single user: signed URL + immediate cookie exchange.** As in
   `<magic-links-and-signed-urls>`. Short single-use entry token → session cookie scoped to one
   review. Add a per-review revocation flag so Finish invalidates everything.

   **Steal Trigger.dev's `publicAccessToken` here.** When the review is created, mint two
   credentials, not one: the short-lived single-use *entry* token in the URL, and a **scoped,
   browser-safe token** that authorises exactly one operation — submitting annotations to *this*
   review — and nothing else. The page holds the second; it is CORS-safe to expose because it
   cannot do anything but complete the review it belongs to. This is what lets a published page
   call back without embedding a privileged key, and it is precisely how you satisfy MCP's
   *"MUST NOT provide a URL which is pre-authenticated to access a protected resource"* — the
   URL grants no ambient authority, only the ability to finish one specific review.
3. **Hosted, team: GitHub OAuth or Entra ID.** Only once more than one human can review, or
   an audit trail of *who said what* is required. Entra ID if the user's org is Microsoft-shop
   (this one is: `norsyn.dk`, and the surrounding tooling is .NET/AutoCAD/Revit). GitHub OAuth
   if the audience is open-source developers. Both are heavier than the problem at step 1–2.

**Verdict: build step 1 first and step 2 behind a flag.** Do not build step 3 until a second
human needs to review. Note that a public hosted deployment is what forces MCP's URL-mode
identity-binding requirements onto you; localhost sidesteps them entirely.
</auth-recommendation>

</auth>

<survey-hitl-platforms>

<humanlayer>
**What:** originally a hosted HITL API + Python/TS SDK — wrap a dangerous function in an approval
decorator, the call blocks, a human approves in Slack/email/SMS, the result returns to the LLM.

**Link:** https://github.com/humanlayer/humanlayer · https://humanlayer.com

**Status — important:** the HITL SDK is **effectively abandoned**. The repo README now reads
verbatim: *"public issues repo for humanlayer - the code here is pretty much all deprecated -
you can try the rebuild of humanlayer at https://humanlayer.com - thanks for all your support -
dex"*. PyPI `humanlayer` is frozen at **0.7.9 (2025-06-03)**. `docs.humanlayer.dev` API-reference
pages 404 and the certificate is expired. The company pivoted to a Claude Code orchestration
IDE (a Go daemon + Tauri/React app).

**Mechanism (v0.7.9, for reference):** `require_approval(contact_channel, reject_options)`
wraps a function and **synchronously polls every 3 seconds** while `call.status.approved is
None`. `human_as_tool()` returns a `Callable[[str], str]` — "ask a human" as an ordinary LLM
tool. Data model splits `spec` (intent) from `status` (outcome), with `run_id` + `call_id`
correlation and an opaque `state: dict` round-tripped through the human step.

**Return payload:** a boolean plus an optional free-text `comment`, or a single `response`
string. `reject_option_name` lets the human pick a pre-declared canned option so the model
receives a stable enum token rather than prose. Token-efficient, but it is a *point decision*.

**Verdict: ignore as a dependency, borrow four ideas.** (a) `run_id` + `call_id` correlation
pair; (b) the opaque `state` dict round-tripped through the human step so the agent side stays
stateless — this is the same idea as MCP's `requestState`, arrived at independently;
(c) `spec`/`status` separation; (d) `ResponseOption` with `prompt_fill` as pre-declared
structured choices. The 3-second polling loop assumes a human answers in minutes, which is the
wrong assumption for document review.
</humanlayer>

<langsmith-annotation-queues>
**What:** a reviewer workflow inside LangSmith for attaching human feedback to *traced runs* —
rubric scores, free-text notes, corrections. Single-run and pairwise (A/B) queues.

**Link:** https://docs.langchain.com/langsmith/annotation-queues

**Mechanism:** runs enter a queue manually, via automation rules (filter on errors/low scores),
from dataset experiments, or via SDK (`create_annotation_queue`, `add_runs_to_annotation_queue`,
`create_feedback`, `list_feedback`). UI is the LangSmith web app only.

**Can an agent pull results back mid-run?** **No.** There is no wait/subscribe primitive and no
webhook that resumes a paused run. The docs frame queues as evaluation and dataset-curation
tooling — offline, post-execution. You could poll `list_feedback` by `run_id`, but that is
inventing a protocol on top of an eval tool. *(Exact `list_feedback` parameter names
**unverified** — the rendered reference pages did not yield them.)*

**Verdict: ignore.** Wrong category — offline eval, not an agent-pause primitive, and locked to
a LangSmith account. The one borrowable idea is the **feedback config / rubric**: pre-declaring
feedback keys and allowed values so what comes back is uniform and machine-consumable.
</langsmith-annotation-queues>

<agent-inbox>
**What:** an open-source web "inbox" that surfaces LangGraph `interrupt()` calls as reviewable
items. Closest existing analogue to cloudreview: agent publishes Markdown → web page → structured
response.

**Link:** https://github.com/langchain-ai/agent-inbox (hosted at dev.agentinbox.ai)

**Schema** (verbatim from `src/components/agent-inbox/types.ts`; the same types ship in
`langgraph.prebuilt.interrupt`):

```typescript
export interface HumanInterruptConfig {
  allow_ignore: boolean;
  allow_respond: boolean;
  allow_edit: boolean;
  allow_accept: boolean;
}
export interface ActionRequest { action: string; args: Record<string, any>; }
export interface HumanInterrupt {
  action_request: ActionRequest;
  config: HumanInterruptConfig;
  description?: string;
}
export type HumanResponse = {
  type: "accept" | "ignore" | "response" | "edit";
  args: null | string | ActionRequest;
};
```

`description` is documented as *"Should be detailed, and may be markdown"* and is rendered as the
item body — so **long Markdown in, one coarse decision out.** Resume goes back via
`Command(resume=[...])`; per the README the inbox *"will always send back a list with a single
`HumanResponse` object in it"*, so the node reads `response[0]`.

**Verdict: borrow the schema ideas, ignore as a deployment target.** It requires a LangGraph
deployment URL, an assistant ID, and a LangSmith API key in the browser. More fundamentally, the
response granularity is **one verdict per interrupt** — there is no notion of N annotations
anchored to M locations. Encoding per-section comments inside the `response` string means
fighting the schema. Worth stealing: the four `allow_*` **capability flags**, so the payload
tells the page which controls to render instead of the UI hardcoding them. cloudreview should do
the same (`allow_comment`, `allow_image`, `allow_rewrite` per section).
</agent-inbox>

<ag-ui>
**What:** an event-stream protocol standardizing agent→UI communication, with TS/Python SDKs and
integrations across LangGraph, CopilotKit, Microsoft Agent Framework, assistant-ui.

**Link:** https://docs.ag-ui.com · https://github.com/ag-ui-protocol/ag-ui

**Is there a real INTERRUPT event?** **No.** There is no `INTERRUPT` event type in the event
list (lifecycle / text / tool / state / activity / reasoning / raw / custom). HITL is instead a
**DRAFT** interrupt-aware *outcome* on `RunFinished`:

```typescript
type Interrupt = {
  id: string; reason: string; message?: string; toolCallId?: string;
  responseSchema?: JsonSchema; expiresAt?: string; metadata?: Record<string, any>
}
type RunFinishedOutcome = { type: "success" } | { type: "interrupt"; interrupts: Interrupt[] }
type RunAgentInput = { /* ... */
  resume?: Array<{ interruptId: string; status: "resolved" | "cancelled"; payload?: any }> }
```

The agent **ends the run** carrying interrupts (after emitting `StateSnapshot` +
`MessagesSnapshot` so resume is checkpoint-agnostic); the client starts a **new run on the same
thread** with a `resume` array.

**Verification status:** the docs label these *"currently in draft status and may change before
finalization"*, and `outcome` is *"Omitted on legacy producers."* The JS SDK reference still
documents `RunFinishedEvent` without an `outcome` field, and discussion #158 (which asks exactly
this question) has **no official maintainer answer**; community consensus there is that HITL
today is done by emitting `TOOL_CALL_START` for a frontend tool and pausing until
`TOOL_CALL_RESULT`. **Whether `outcome` has landed in shipped SDK code is unverified** — the SDK
documentation says no.

**Verdict: borrow two ideas, ignore as a dependency.** (a) **`responseSchema`** — the agent
declares the JSON Schema of the answer it wants. This is the single most valuable idea in the
whole survey and the only mechanism found that lets an agent express "return me an array of
anchored annotations" without abusing the schema. (b) **End-the-run-and-resume-on-a-new-run** —
structurally identical to MCP's MRTR, and the right shape for genuinely async review because
nothing holds a connection or process open. Ignore the streaming event zoo; cloudreview
publishes one document once, not a token stream.
</ag-ui>

</survey-hitl-platforms>

<survey-framework-primitives>

<langgraph-interrupt>
**What:** `interrupt(value)` pauses a graph and surfaces a JSON-serializable value; resume with
`Command(resume=value)` on the **same `thread_id`**, and that value becomes `interrupt()`'s
return.

**Link:** https://docs.langchain.com/oss/python/langgraph/interrupts

**Mechanism:** surfaced as `result["__interrupt__"]` from `graph.invoke()`, or
`stream.interrupts` with `stream_events(version="v3")`. Requires a checkpointer —
*"use a durable checkpointer in production"*; `InMemorySaver` is explicitly *"not
restart-durable."* Waits **indefinitely**, no timeout. Parallel interrupts resume via a map
keyed by interrupt `id`.

**Sharp edge worth recording:** on resume the node **re-executes from its start** up to the
`interrupt()` call — *"any code that ran before the interrupt will execute again."* Any side
effect before the interrupt runs twice.

**Verdict: borrow the model, ignore the dependency.** Bidirectional JSON both ways, unbounded
wait, durable with a Postgres checkpointer — the best-articulated pause/resume semantics found.
Wrong ecosystem for a Claude Code tool. The re-execution gotcha generalises: **cloudreview's
"publish the document" step must be idempotent**, because MRTR retries the whole `tools/call`
too. Keyed on a document content hash, a retry must return the *existing* review, not publish a
second one.
</langgraph-interrupt>

<temporal>
**What:** a durable execution engine. Workflow state is reconstructed from a persisted event
history, so a workflow awaiting a signal consumes no process and survives any restart.

**Link:** https://docs.temporal.io/develop/python/message-passing

**Mechanism:**
```python
@workflow.signal
def approve(self, input: ApproveInput) -> None: self.approved_for_release = True

@workflow.update
def set_language(self, language: Language) -> Language:
    previous_language, self.language = self.language, language
    return previous_language

await workflow.wait_condition(lambda: self.approved_for_release)
```
**Signal = fire-and-forget, no return value. Update = request/response**, caller blocks and gets
a typed return, with a persisted `WorkflowExecutionUpdateAccepted` event.

**Durability — by a wide margin the strongest surveyed:** *"A Temporal Workflow Execution has no
time limit."* Durable timers span *"as short as one second or as long as several years."* The
real ceilings are history size (**51,200 events / 50 MB**, warning at 10,240 / 10 MB). Signals
reach only workflows that have not closed.

**Verdict: borrow the semantics, ignore the engine.** `@workflow.update` is the right shape for
cloudreview's reply leg — **typed payload in, typed ack out**, so the review page can render
"accepted, 40 annotations recorded, 2 failed to anchor" instead of firing blindly into the void.
Adopting Temporal (worker fleet, determinism constraints) is heavy for one doc-review pause, but
the pattern — a persisted run record plus a typed resume message keyed by run id, where the
agent's wait is *reconstructable* rather than a live coroutine — is the reference standard.
</temporal>

<crewai-human-input>
**What:** `human_input=True` on a `Task` pauses after the agent's answer and feeds human
feedback back as an LLM message.

**Link:** https://docs.crewai.com/en/learn/human-input-on-execution ·
https://docs.crewai.com/en/learn/human-in-the-loop (Enterprise)

**Correction to the common wisdom:** it is **no longer hardcoded stdin**. Current `main` defines
a `HumanInputProvider` protocol with both `handle_feedback()` and `handle_feedback_async()`,
plus ContextVar-scoped `set_provider()` / `get_provider()`. There is a real extension seam. But
in OSS it remains an in-process `await` — **not durable**; kill the process and the run is gone.

CrewAI **Enterprise** adds a genuinely async webhook path: kickoff registers a
`humanInputWebhook`, execution enters `Pending Human Input`, and resume is
`POST {BASE_URL}/resume` with `{execution_id, task_id, human_feedback, is_approve}`. Documented
gotcha: webhook URLs are **not** carried over from kickoff and must be resent on `/resume`.

**Verdict: ignore.** The feedback payload is a **plain string** fed to the LLM. No structured
annotation model, no anchors. The narrow two-method provider interface is a tidy shape, but you
would be inventing the entire schema yourself anyway.
</crewai-human-input>

<autogen-userproxyagent>
**What:** in v0.4+/AgentChat, `UserProxyAgent` is stripped to one job — call a user-supplied
input function and return the result as a chat message. The v0.2 `human_input_mode` enum
(`ALWAYS`/`TERMINATE`/`NEVER`) is **gone**.

**Link:** https://microsoft.github.io/autogen/stable/reference/python/autogen_agentchat.agents.html#autogen_agentchat.agents.UserProxyAgent

**Signature confirmed from source:**
```python
SyncInputFunc = Callable[[str], str]
AsyncInputFunc = Callable[[str, Optional[CancellationToken]], Awaitable[str]]
InputFuncType = Union[SyncInputFunc, AsyncInputFunc]

def __init__(self, name: str, *, description: str = "A human user",
             input_func: Optional[InputFuncType] = None) -> None:
```
There is also an `InputRequestContext` (ContextVar-backed) exposing `request_id()` so a web/WS
front end can correlate an outstanding request to a reply.

**Durability: no — and the source says so.** The docstring warns that `UserProxyAgent` *"puts a
running team in a temporary blocked state until the user responds."* Its own recommended
mitigation for slow humans is **not** durability but bailing out: `HandoffTermination`, persist
team state, restart later.

**Verdict: ignore.** A bare `Awaitable[str]` — one string in, one string out, no durability, and
the framework's own answer to "the human takes 3 hours" is "terminate the team." Explicitly the
wrong shape. Borrow only `request_id()` correlation, which HumanLayer's `call_id` already covers.
</autogen-userproxyagent>

<openai-agents-sdk>
**What:** tools marked `needs_approval` halt the run before executing; pending
`ToolApprovalItem`s surface on `RunResult.interruptions`; you approve/reject and resume from a
serializable `RunState`. Real and shipped in **both** Python and JS SDKs.

**Link:** https://openai.github.io/openai-agents-python/human_in_the_loop/

**Mechanism:**
```python
state = result.to_state()
state.approve(interruption, always_approve=False)   # or state.reject(interruption)
result = await Runner.run(agent, state)
```
Persistence via `state.to_string()` / `RunState.from_string(...)` and `to_json()` / `from_json()`.

**Durability: yes, at the application level — the interesting bit.** The JS docs state the flow
*"is designed to be interruptible for longer periods of time without keeping your server
running"* and that these APIs *"enable long-running approvals spanning hours or days across
different processes or server restarts."* No maximum pause duration is documented — the ceiling
is your storage. **Unverified:** how a multi-day-old `RunState` behaves against a rotated model
version, given frozen tool-call ids inside the blob.

**Verdict: borrow the `RunState` mechanic, not the vocabulary.** *Pause → serialize to an opaque
string → store → resume in a different process* is exactly the "3 hours to annotate" story
without a workflow engine — and it is the same idea as MCP's `requestState`. But the decision
surface is **binary approve/reject per tool call** plus a free-text rejection message. Smuggling
40 anchored annotations through `reject(message=...)` would be an abuse.
</openai-agents-sdk>

<triggerdev-wait-tokens>
**What:** an agent creates a waitpoint token, gets back an id + URL + browser-safe credential, a
human completes it from anywhere, and the run resumes with the submitted payload.
**This is the closest existing analogue to cloudreview's mechanics.**

**Link:** https://trigger.dev/docs/wait-for-token

**Mechanism:**
```ts
const token = await wait.createToken({ timeout: "10m" });
// => { id: "waitpoint_...", url, publicAccessToken, isCached }

const result = await wait.forToken<ApprovalToken>(token.id);
if (result.ok) { /* result.output is typed T */ }
```
Completion from a browser, over plain HTTP:
```
POST https://api.trigger.dev/api/v1/waitpoints/tokens/{tokenId}/complete
Authorization: Bearer {publicAccessToken}     // browser-safe, CORS-enabled
Body: { "data": { ...arbitrary structured output } }
```
Options: `timeout` (default `"10m"`), `idempotencyKey`, `idempotencyKeyTTL`, `tags`.
`listTokens()` filters by status (`WAITING`/`COMPLETED`/`TIMED_OUT`).

**Durability: yes, and mechanically interesting.** Runs are checkpointed with CRIU
(Checkpoint/Restore In Userspace) — the process is snapshotted, the machine torn down, and on
completion restored in a fresh environment. Paused time is not billed.

**Caveats:** the 10-minute default timeout is far too short for document review and must be set
explicitly; **no documented maximum was found — unverified**. A widely-blogged `useWaitToken`
React hook does **not** appear in current docs — **unverified**.

**Verdict: borrow the token model wholesale.** Two things to steal specifically:
1. **The completion payload is an arbitrary structured JSON object**, typed as `T` on resume —
   unlike OpenAI's approve/reject, an annotation array drops straight in.
2. **`publicAccessToken` — a scoped, browser-safe, CORS-enabled credential returned alongside
   the URL.** This solves the problem cloudreview hits immediately: how a published page calls
   back without embedding a privileged key. This directly informs `<auth>` — and it satisfies
   MCP's "MUST NOT provide a pre-authenticated URL" rule, because the credential is scoped to
   completing *one* waitpoint, not to the account.
</triggerdev-wait-tokens>

<inngest-agentkit>
**What:** `step.waitForEvent()` inside a tool handler — durable, replay-based, multi-hour waits.

**Link:** https://agentkit.inngest.com/advanced-patterns/human-in-the-loop

```ts
const developerResponse = await step.waitForEvent("developer.response", {
  event: "app/support.ticket.developer-response",
  timeout: "4h",
  match: "data.ticketId",
});
```

**Verdict: borrow (weaker sibling of Trigger.dev).** Structurally near-identical and genuinely
durable, and the `timeout: "4h"` in the official example confirms multi-hour waits are intended.
But correlation is by **event field matching** rather than an issued token, so you must generate
and thread your own id, and anyone who can emit the event can complete the wait. No built-in
scoped browser credential. Trigger.dev's model is the better one to copy.
</inngest-agentkit>

<other-products>
- **Braintrust human review** (https://www.braintrust.dev/docs/annotate/human-review) — hosted
  review queues over spans/logs; reviewers give categorical scores, sliders, and *"Free-form
  input: String values written to `metadata` or `expected` at a specified path."* **Verdict:
  ignore as a pause primitive** (nothing blocks on it, and no documented API to read review
  results back out was found). **Borrow one idea:** free-form input written *to a specified path
  in a structured object* is a clean way to model an annotation targeting a location.
- **Humanloop** — the team was acquired by Anthropic in 2025 and the standalone platform is
  being sunset with a published migration guide. **Ignore; do not build on it.**
</other-products>

</survey-framework-primitives>

<the-gap-nobody-fills>
Worth stating plainly, because it is the strongest justification for building cloudreview at all:

**Every system surveyed models a *point decision* attached to a *single item*** — approve/deny
(HumanLayer, OpenAI Agents SDK), accept/edit/respond/ignore (Agent Inbox), a rubric score
(LangSmith, Braintrust), or one free-text string (CrewAI, AutoGen). **Not one has a first-class
notion of many annotations anchored to positions within one long artifact.**

The two that could express it without schema abuse are AG-UI's `responseSchema` (**draft**, and
apparently not in shipped SDKs) and Trigger.dev's arbitrary `data` blob (**generic** — it gives
you a pipe, not a schema). The annotation model itself — anchors, re-anchoring, per-section
verdicts, images — is cloudreview's to design regardless of which transport it picks. That is
the product.

Note also a striking convergence, arrived at independently by four designs: MCP's
`requestState`, HumanLayer's opaque `state` dict, AG-UI's end-run-and-resume-on-a-new-run, and
OpenAI's serializable `RunState` are all **the same idea** — push the wait state through the
other party so the waiting side stays stateless. MCP's version is the one cloudreview gets for
free, and it is the best-specified of the four (integrity protection, TTL, principal binding all
mandated).
</the-gap-nobody-fills>

<recommendation>

<a-mcp-http-or-both>
**Both — but with a strict split, and the HTTP API is the product.**

- **HTTP API = the system.** Document publication, the review page, annotation storage,
  the submit/Finish transition, the annotation-fetch endpoint. All business logic. This must
  exist regardless, because URL-mode elicitation *requires* the data to travel out-of-band, and
  because the review page is a web app.
- **MCP server = a thin adapter (~200 lines) over that API.** It exposes one or two tools
  (`publish_review`, and possibly `fetch_annotations`), and drives the human hand-off with
  **URL-mode elicitation delivered via MRTR**.

The canonical flow:

1. Agent calls `cloudreview.publish_review(document, anchors)`.
2. Server stores the doc, mints a review id, returns an `InputRequiredResult` containing a
   URL-mode `elicitation/create` (the review page URL) and an integrity-protected
   `requestState`.
3. Claude Code shows the URL, gets consent, opens the browser, retries the call with
   `action: "accept"` and the echoed `requestState`.
4. Server checks whether the human clicked Finish. Not yet → another `InputRequiredResult`
   (no `inputRequests`, which the client MAY retry immediately) → a bounded poll loop.
   Done → return the annotations as the tool result.
5. Auto-backgrounding means the agent keeps working while the human annotates.

Three details that come straight from the survey and are cheap to get right up front:

- **`publish_review` must be idempotent, keyed on a document content hash.** MRTR retries the
  *entire* `tools/call`, and LangGraph's identical gotcha (*"any code that ran before the
  interrupt will execute again"*) shows what happens if you skip this: you publish the same
  document two, three, five times. A retry must return the *existing* review.
- **Return a typed acknowledgement, not a void.** Borrow Temporal's `@workflow.update`
  semantics: when the human clicks Finish, the page should get back "accepted, 40 annotations
  recorded, 2 could not be anchored" — not a fire-and-forget 204. The human is the one person
  who can fix an annotation that failed to anchor, and only while they are still looking at it.
- **Let the payload drive the UI.** Borrow Agent Inbox's `allow_*` capability flags: the agent
  declares per section what the human may do (`allow_comment`, `allow_rewrite`, `allow_image`),
  and the page renders accordingly instead of hardcoding one interaction model. And borrow
  AG-UI's `responseSchema` idea — have the agent declare the shape of the answer it wants, since
  that is the one thing no shipped system provides and the whole reason a generic
  approve/reject tool cannot do this job.

Why not MCP-only: MCP is young and just shipped a breaking change; a local HTTP core with an MCP
adapter survives that. Why not HTTP-only: you would give up native Claude Code discovery,
consent-gated URL opening, OAuth, and auto-backgrounding, and you would have to invent a CLI.

**Ship order:** HTTP API + review page first, prove the round trip with `curl`, then add the MCP
adapter. And **verify elicitation support empirically** against the installed Claude Code
version before betting the architecture on it — with a plain `fetch_annotations` tool as the
fallback path if URL-mode elicitation turns out not to work (the agent just calls it after the
human says "done").
</a-mcp-http-or-both>

<b-most-token-efficient-return-format>
**A compact, line-oriented, anchor-referential format — with a JSON escape hatch for the few
annotations that need structure.** Roughly:

```
review: auth-spec@a3f19c2   verdict: revise   40 notes

q3 > too vague. split into "who authenticates" and "how tokens rotate"
q7 > cut
q9 > ok
q12 > use Entra ID, not GitHub OAuth — we're a Microsoft shop
q14 ~ {"img":"cr://img/7a2f","note":"see the whiteboard photo"}
q18 > [anchor moved: matched by quote at 94%] this assumes Postgres; we're on SQL Server
```

Why this shape:

1. **It never re-sends the document.** This is ~90% of the total saving (~9,000 → ~1,000 tokens
   on a 6,000-word doc). Everything else is a refinement.
2. **Anchors are short, semantic, agent-minted ids** (`q3`, `auth-strategy`) — 1–3 tokens,
   self-describing enough to survive context compaction, stable across regeneration, and free of
   the index-arithmetic failure mode the JSON Whisperer paper documents.
3. **~3–4 tokens of format overhead per annotation**, versus ~10–12 for a JSON array — ~25–30%
   cheaper on a 40-annotation payload.
4. **Silence is free.** Only annotated anchors appear. 60 sections, 40 annotations → 40 lines.
5. **Verdict shorthand for the common case.** `ok` / `cut` / `expand` cost ~3–4 tokens against
   ~27 for prose. On a questionnaire where most sections are accepted, this is the largest
   remaining win.
6. **A document version on the header line** (`auth-spec@a3f19c2`), copying GitHub's `commit_id`,
   so the agent can detect that it edited the doc mid-review.
7. **Re-anchoring is reported, not hidden.** When fuzzy matching moved an annotation, say so
   inline — the agent needs to know its confidence is degraded.
8. **`~` + JSON only where structure is genuinely needed** (images, multi-line code,
   ranges). Pay the JSON tax on the 5% of annotations that need it, not the 95% that don't.

Budget: ~1,000–1,500 tokens for a 40-annotation review — comfortably under Claude Code's
10,000-token warning threshold and far under the 25,000 hard cap. **Paginate above ~150
annotations** rather than getting truncated. Document this format in the `cloudreview` skill so
the agent knows how to read it without being taught in-context every time.
</b-most-token-efficient-return-format>

<c-single-biggest-trap>
**Returning the document instead of referring to it.**

It is the natural implementation — the human annotated a document, so send back the annotated
document — and it is wrong in three compounding ways:

- **It wastes ~8–9× the tokens**, re-transmitting text the agent authored minutes ago and still
  has in context.
- **It hits Claude Code's ceilings.** 25,000-token hard cap on MCP tool output, warning at
  10,000. A long document plus annotations will be silently truncated, and truncation of a
  *review* is a correctness bug: the agent acts on a partial set of the human's instructions
  while believing it has them all.
- **It destroys the signal.** Forty comments buried in 6,000 words of unchanged prose is a
  needle-in-haystack problem for the model. The comments *are* the payload; the document is
  noise the agent already knows.

The discipline: **the return trip carries only what the agent does not already know** — which
anchors were touched, what was said, and whether the document version still matches.

Two runners-up worth guarding against:

- **Anchoring to line numbers or character offsets.** Breaks on the first edit above the target,
  and imports LSP's UTF-16-vs-UTF-8 encoding mess. Use agent-minted semantic ids with
  quote-based fuzzy repair.
- **The idle timeout.** A remote HTTP MCP server that goes quiet for 5 minutes while the human
  reads gets its call killed. Either emit progress notifications, or — better — use MRTR's
  repeat-`InputRequiredResult` loop so no single call ever waits long.
</c-single-biggest-trap>

</recommendation>

<open-questions>
1. **Does the installed Claude Code build actually honour URL-mode elicitation, and over MRTR or
   the legacy server-initiated pattern?** Unverified. Test with
   `github.com/mcp-use/mcp-elicitation-demo` before committing.
2. **Does Claude Code support MCP Apps (`ui://`) at all?** Absent from its MCP docs; assumed no.
   Unverified.
3. **Exact `AskUserQuestion` tool schema** — only secondary sources found. Unverified.
4. **Whether auto-backgrounded MCP calls survive a `/compact`.** Docs say background tasks do
   not survive *exiting the session*; compaction behaviour is unstated. Unverified, and it
   matters for long reviews.
</open-questions>

<sources>
Primary:
- MCP elicitation (2026-07-28): https://modelcontextprotocol.io/specification/2026-07-28/client/elicitation
- MCP elicitation (2025-06-18, superseded): https://modelcontextprotocol.io/specification/2025-06-18/client/elicitation
- MCP Multi Round-Trip Requests: https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/mrtr
- MCP client concepts (elicitation/roots/sampling, deprecations): https://modelcontextprotocol.io/docs/learn/client-concepts
- MCP Apps SEP-1865: https://modelcontextprotocol.io/seps/1865-mcp-apps-interactive-user-interfaces-for-mcp
- Claude Code MCP: https://code.claude.com/docs/en/mcp
- Claude Code hooks: https://code.claude.com/docs/en/hooks
- Claude Code skills/slash commands: https://code.claude.com/docs/en/slash-commands
- GitHub PR reviews API: https://docs.github.com/en/rest/pulls/reviews
- W3C Web Annotation Data Model: https://www.w3.org/TR/annotation-model/
- LSP 3.17 specification: https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/
- LangGraph interrupts: https://docs.langchain.com/oss/python/langgraph/interrupts
- Trigger.dev wait tokens: https://trigger.dev/docs/wait-for-token
- Temporal message passing (signals/updates): https://docs.temporal.io/develop/python/message-passing
- Temporal workflow limits: https://docs.temporal.io/workflow-execution/limits
- OpenAI Agents SDK HITL (Python): https://openai.github.io/openai-agents-python/human_in_the_loop/
- OpenAI Agents SDK HITL (JS): https://openai.github.io/openai-agents-js/guides/human-in-the-loop/
- Agent Inbox (types.ts schema): https://github.com/langchain-ai/agent-inbox
- AG-UI events / interrupts (draft): https://docs.ag-ui.com/concepts/events · https://docs.ag-ui.com/concepts/interrupts
- AG-UI discussion #158 (HITL question, unanswered by maintainers): https://github.com/ag-ui-protocol/ag-ui/discussions/158
- AutoGen UserProxyAgent: https://microsoft.github.io/autogen/stable/reference/python/autogen_agentchat.agents.html#autogen_agentchat.agents.UserProxyAgent
- CrewAI human input: https://docs.crewai.com/en/learn/human-input-on-execution · https://docs.crewai.com/en/learn/human-in-the-loop
- Inngest AgentKit HITL: https://agentkit.inngest.com/advanced-patterns/human-in-the-loop
- LangSmith annotation queues: https://docs.langchain.com/langsmith/annotation-queues
- Braintrust human review: https://www.braintrust.dev/docs/annotate/human-review
- HumanLayer (deprecated): https://github.com/humanlayer/humanlayer · https://pypi.org/project/humanlayer/

Secondary / supporting:
- Hypothesis fuzzy anchoring: https://web.hypothes.is/blog/fuzzy-anchoring/
- JSON Whisperer (EMNLP 2025): https://arxiv.org/html/2510.04717v1
- URL-mode elicitation explainer (WorkOS): https://workos.com/blog/mcp-url-mode-elicitation
- MCP client capability matrix (stale re: Claude Code): https://github.com/apify/mcp-client-capabilities
- Elicitation conformance test server: https://github.com/mcp-use/mcp-elicitation-demo
- Anthropic on MCP 2026-07-28: https://claude.com/blog/bringing-mcp-2026-07-28-to-claude
- claude-code issues #2799, #7108 (2025 elicitation requests, now stale)
</sources>
