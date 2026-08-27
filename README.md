## Claude + Google Workspace: AI-driven document generation with grounding and cost controls

A pattern reference from building a production Google Sheets → Google Slides deck generation pipeline for OpnRoad M&A, a Canadian M&A advisory firm. The pipeline generates confidential deal documents (Confidential Information Memorandums, Blind Profiles, Business Plans) with AI-written narrative sections grounded to real deal data.

This repo contains no client data, no deal information, no real API keys, no proprietary prompts, and no source code. It's a writeup of the engineering patterns used to make LLM-driven document generation reliable, cheap, and safe enough to trust with confidential M&A work.

---

### The core problem

Any team generating client-facing documents at scale (M&A memos, legal briefs, sales collateral, reports) hits the same three problems the moment they add an LLM to the pipeline:

1. **Hallucination risk on client-facing output.** A fabricated financial metric or an invented customer name in an M&A CIM is not a small error. It's a lawsuit.
2. **Runaway cost on multi-slide / multi-section generation.** Four to six Claude calls per document, hundreds of documents per month, adds up fast.
3. **State management under real-world latency.** A 30-60 second generation can't block the user's chat window. Bot UIs need to acknowledge fast and continue in the background.

Below are the patterns I use to solve each of those cleanly.

---

### Pattern 1: Grounding contract with "empty is safer than wrong"

Every field the LLM fills is bound to a specific data source in the prompt: a Cognito form field, a financial metric from a source spreadsheet, or a fact from a fetched company website. The system prompt explicitly forbids:

- Inventing metrics, dates, certifications, customer names, or superlatives
- Filling from general knowledge of the industry
- Using vague marketing language when the grounded data doesn't support it

If a field can't be grounded, the model returns empty string. The token stays visible in the output document (`{{ai:field_name}}`) so the reviewer can see exactly what needs manual completion.

Empty is safer than fabricated. This is not the default LLM behavior. You have to engineer against it.

---

### Pattern 2: Prompt caching for cost reduction

The shared company context (form data, website content, financial summaries) is identical across every slide's generation call for a given deal. Anthropic's `cache_control: "ephemeral"` breakpoint gets a huge discount here:

- First call pays 1.25x on cache-write tokens
- Every subsequent call within 5 minutes pays 0.10x on cache-read tokens

Result: a 4-slide document generation dropped from roughly $0.15 per deck to $0.037 per deck. Same output, 75% cost reduction. At production volume this matters.

Key: the cached prefix must be *stable*. Any per-call variation (deal name, slide-specific instruction) goes AFTER the cache breakpoint. Anything shared (company facts, financial figures, source website text) goes BEFORE.

---

### Pattern 3: Model tiering per output type

Not every slide needs the same model. The pipeline picks per output:

- **High-stakes fact-heavy narrative (executive summary, business analysis):** Claude Opus. Best reasoning, lowest hallucination risk. Worth the cost on customer-facing sections.
- **Structured extraction (financial tables, metadata):** Claude Sonnet. Good enough, faster, cheaper.
- **Speculative ideation (growth opportunities, discussion prompts):** Sonnet. Speculation is expected; Opus's precision isn't the right tradeoff here.

Model tiering can cut monthly LLM spend 30-40% on a mixed workload without sacrificing quality on the parts that matter.

---

### Pattern 4: Preflight before you spend tokens

Every AI generation call is preceded by a permission preflight against the target Google Sheets (READ) and Google Slides (WRITE via a sentinel `replaceAllText` call).

- Sheet not shared with the service account? Return `sheet_read_denied` with the service account email. Zero Claude cost.
- Deck not shared? Return `deck_write_denied`. Zero Claude cost.
- Transient 5xx/429? Return `_transient`. Bot can retry cleanly.

Users get an actionable error in 2 seconds instead of a generation failure after 30 seconds of billed Claude tokens.

---

### Pattern 5: Google Slides chart re-linking (an API workaround)

The Google Slides API has no "change source spreadsheet" call for existing linked charts. Copying a master deck to a per-deal deck leaves the charts still pointing at the master spreadsheet.

Workaround: in a single atomic `batchUpdate`, delete each linked chart from the new deck and recreate it linked to the deal spreadsheet, preserving `chartId`, position, size, and group membership. Works because every deal spreadsheet is a copy of the same template, so chart IDs line up cleanly.

This is one of those patterns that isn't documented anywhere sensible. Google issue tracker mentions it as a known limitation. Non-obvious workaround, ships reliably in production.

---

### Pattern 6: Template markers stripped on copy

The master template deck ships with visual authoring markers (colored circles / squares) that human authors use to indicate which slides are AI-driven, which are analyst-owned, and which are optional. When the pipeline copies the master to a per-deal deck, it strips all markers so the generated deck is clean for the buyer.

Master stays annotated for the next deal. Generated decks stay professional. Same template, different rendering per audience.

---

### Pattern 7: Stateful bot with background execution

User-facing flow lives in a chat bot (Zoho Cliq). Generation takes 30-60 seconds. Blocking the bot's response loop is not an option.

Pattern:
- Session state (report type chosen, sheet URL, AI-decision made) lives in a `bot_sessions` table with 60-minute TTL, keyed by user + chat ID.
- Bot acknowledges the user's action immediately: "Starting generation for [deal name]. I'll message you when it's ready."
- Actual generation runs as a background task via `EdgeRuntime.waitUntil()` in the Supabase edge function.
- On completion, the bot sends a follow-up message with the deck link and a summary of what filled vs what needs manual review (`stillMissing[]` array).

The user gets a responsive UI. The generation runs to completion regardless of bot session state. No blocked requests, no silent failures.

---

### Pattern 8: Character-limit hard caps with sentence-boundary trimming

LLMs don't respect character limits reliably. If your slide title fits 25 characters and the body fits 200, you either enforce it in code or you get visual overflow.

After generation, every field with a hard character cap is trimmed at the last sentence boundary before the limit. Trims cleanly ("...The primary revenue driver."), never mid-word ("...The primary reven").

Enforced after `replaceAllText`, before styling restoration. Runs in linear time. No LLM re-call needed.

---

### Pattern 9: Brace-matching JSON extraction

Claude sometimes prefaces its JSON output with prose ("Here's the JSON response:") or wraps it in markdown code fences. Naive `JSON.parse()` fails on both.

Instead of greedy regex (which breaks on `{` inside string literals), use a brace-matching parser that walks balanced braces + escaped characters to extract the first valid JSON object from a mixed response. Handles all the real-world edge cases the model throws at you.

---

### Pattern 10: Font and style restoration post-replacement

Google Slides' `replaceAllText` inherits the first character's style from the token being replaced. A token that starts with a bold character means the entire replaced text renders bold, even if you only wanted the title bold and the body plain.

After replacement, the pipeline applies per-card `updateTextStyle` operations to split title (bold, larger font) from body (regular, smaller font), matching the reference deck's design system. Small detail. Big UX difference between "looks templated" and "looks designed."

---

### Stack this all runs on

- **Backend:** Supabase Edge Functions (Deno + TypeScript), gated with `x-test-secret` header (timing-safe comparison)
- **APIs:** Google Sheets v4, Google Slides v1, Google Drive v3 (service account JWT auth)
- **LLM:** Anthropic Claude API (Opus for high-stakes, Sonnet for speculative and utility)
- **State:** Postgres (Supabase) `bot_sessions` table with 60-min TTL
- **UI:** Zoho Cliq bot (incoming webhooks + message handlers)
- **Secrets:** Supabase function secrets (never in code, never in workflow JSON)

---

### About this writeup

I'm Hassan Nawaz, a senior AI-native full-stack engineer. I built this pipeline for OpnRoad M&A as part of an ongoing engagement. Roughly 30-60 seconds per generated deck, 4 parallel LLM calls per deck, ~$0.037 per deck after prompt caching, running reliably in production.

Contact: [stellaflo.com](https://stellaflo.com) · [linkedin.com/in/ehassannawaz](https://www.linkedin.com/in/ehassannawaz)
