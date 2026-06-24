# AI Tree Lab — Design Spec

**Date:** 2026-06-21
**App:** Roots & Records (`roots-and-records-v4.html`)
**Author:** J.L. Foreman + Claude

## Goal

Let the AI research assistant review the **active tree** and generate one or more
**alternate trees** the user can preview and save. Headline mode: split a tree into
separate per-line trees, each with guidance on *who to start with* and *what to research*.

## User decisions (locked)

- **Primary mode:** Split into separate line-trees ("AI proposes, you pick").
- **All four modes built now** (shared pipeline; mode = different prompt + output shape).
- **Research plan shows as Banner + Notes** in each saved tree.
- **Review-before-save:** AI proposes; user checks which candidates to save. Nothing auto-created.

## Modes

| Mode | Adds speculative people? | Output |
|------|--------------------------|--------|
| 🌳 Split into line-trees *(primary)* | No (reorganize only) | One candidate per family line found |
| 🔮 Hypothesis / what-if | **Yes** (flagged) | 1+ candidates with proposed ancestors to break a brick wall |
| 🧹 Clean-up copy | No (correct only) | One corrected candidate; flags dupes/date conflicts/missing links |
| ⚖️ Competing theories | **Yes** (flagged) | 2–3 candidates, each a different lineage theory |

## Where it lives

A new dedicated **🧪 AI Tree Lab** nav section (its own page, listed under
`🔮 AI Research Assistant`). No new backend. Uses the existing `callAI(messages, opts)`
chain (Gemini → Groq, user's BYO keys from Settings). If no key is configured, reuse the
app's existing "add a key in Settings" prompt.

The **engine upgrades** (persona, Research Memory, Gemini grounding, multi-step loop) live
inside `callAI` / shared helpers, so they power BOTH the Tree Lab page and the existing
AI Research Assistant chat — built once, not duplicated.

## Data model (existing, reused)

- Trees registered in `localStorage['rr_trees']` = `[{id, name, createdAt}]`.
- Each tree's data blob at `localStorage['rr_data_v1::<treeId>']` = shape of `getAllData()`:
  `{people, notes, hypos, legacyNotes, records, timelineEvents, ...}`.
- **People are referenced by NAME** (`p.father`, `p.mother`, `p.spouse` are name strings),
  not by id. → Splitting is a simple subset copy; cross-line parents remain as unlinked
  name strings (acceptable, shown without a linked card).
- Person shape: `{name, given?, surname?, sex?, birth, death, place, father, mother,
  spouse, marriage?, occ, notes, line, source?, _pbId?}`.

## Pipeline

1. **Build snapshot** of the *active* tree from the live `people` array: for each person
   send `{name, sex, birth, death, place, father, mother, spouse, line, occ,
   hasRecords:bool, noteCount:int}`. Cap at ~200 people; if larger, send a per-line
   summary instead and note the truncation in the UI. Also inject relevant **Research
   Memory** (see "AI engine upgrades" §4) so the AI knows what's already been tried.
2. **Multi-step reasoning loop** (see "AI engine upgrades" §2) instead of one Q→A:
   plan → propose → reflect → synthesize. Every `callAI` turn runs under the
   **AA-genealogy specialist persona** (§3) and, on the Gemini leg, with **Google-Search
   grounding** enabled (§1) so suggested records carry real citations.
3. **Prompt** the final turn with the snapshot + a mode-specific instruction that
   **requires strict JSON** matching the schema below. Hard rules embedded in the prompt:
   - Split / Clean-up: *"Do NOT invent any person not present in the input."*
   - Hypothesis / Competing: speculative people MUST set `speculative: true` and a
     `reason`.
4. **Parse** the reply robustly: strip ```json fences, extract the first balanced
   `{...}`, `JSON.parse`. On parse failure, retry once with a "return JSON only" nudge;
   if it still fails, show the raw text and an error toast (no tree created).
5. **Render candidates** as review cards (see UI). Nothing saved yet.
6. **Save selected** → for each checked candidate, create a new tree.
7. **Record to memory:** append a short episodic summary of this analysis to Research
   Memory (§4) so the next run builds on it.

## AI output schema

```json
{
  "mode": "split | hypothesis | cleanup | competing",
  "summary": "one-paragraph overview of what the AI found",
  "candidates": [
    {
      "name": "Foreman Line",
      "line": "Foreman / Drewitt (Lawson)",
      "people": [
        { "name": "...", "birth": "", "death": "", "place": "",
          "father": "", "mother": "", "spouse": "", "line": "",
          "occ": "", "notes": "",
          "speculative": false, "reason": "" }
      ],
      "startWith": { "person": "Name", "why": "..." },
      "research": [
        { "task": "Find a death record for X", "repository": "e.g. FamilySearch / state archive",
          "why": "...", "sources": [ { "title": "", "url": "" } ] }
      ],
      "summary": "what this candidate represents"
    }
  ]
}
```

`research[].sources` is populated only on the **grounded** (Gemini) path; on the Groq
fallback it is empty and the UI marks the result "ungrounded — verify independently."
`research[]` may also be a plain string for backward simplicity; the renderer handles both.

## Save behavior

For each selected candidate:

1. `id = slugifyTreeName(candidate.name)`; de-dupe against existing tree ids.
2. Push `{id, name, createdAt, aiGenerated:true, mode, sourceTree:<activeId>}` to `rr_trees`.
3. Build the blob: clone `candidate.people` into `people`. Each person carries `line`.
   Speculative people get `source:'AI hypothesis — unverified'` and `speculative:true`.
4. **Research plan → Notes:** push each `candidate.research[]` item as a note
   `{text, line:candidate.line, date, todo:true}` into the blob's `notes` (when the item is
   an object, format `task — repository (why)` and append any `sources` links).
5. **Research plan → Banner:** store `data.aiPlan = {startWith, research, mode,
   sourceTree, createdAt}` in the blob; render a dismissible banner at the top of the
   tree when that tree is active.
6. Write `localStorage['rr_data_v1::<id>'] = JSON.stringify(blob)`. Do **not** auto-switch;
   show a toast "Saved N alternate tree(s) — switch from the tree menu to open."

## AI engine upgrades (adapted patterns)

Four patterns adapted from reviewed open-source deep-research projects
(ii-researcher, open-deep-research, Aetherius, aifa). **Patterns only — no code copied;**
licenses noted: ii-researcher & open-deep-research are Apache-2.0; Aetherius is
CC BY-NC-SA 4.0 (non-commercial, do not copy code); aifa is AGPL-3.0 (copyleft, do not
copy code). All implemented in plain JS through the existing `callAI(messages, opts)` —
no backend, no new user keys, no vector DB.

These upgrades are also wired into the existing **AI Research Assistant chat**, not only
the AI Tree Lab — the persona, memory, grounding, and multi-step loop apply wherever the
assistant answers research questions.

### 1. Gemini Google-Search grounding (from the deep-research repos)
- Add an `opts.grounded` flag to `callAI`. When set **and** the active provider is Gemini,
  include the Google Search grounding tool in the request body
  (`tools: [{ google_search: {} }]` per the Gemini 2.5 API).
- Browsers cannot do general web search/scrape themselves (CORS); Gemini grounding is the
  only no-server way to get real, current sources. **Groq has no equivalent** → grounding
  silently disabled on the fallback leg, and results are flagged "ungrounded."
- Parse `groundingMetadata` from the Gemini response into `sources[]` (title + url) and
  surface them on each research item. Never present an ungrounded suggestion as a verified
  citation (ties to the no-made-up-data rule).

### 2. Multi-step research loop (from ii-researcher reflection + open-deep-research plan→iterate)
- A small JS controller `runResearchLoop(question|treeSnapshot, mode)` that makes 2–4
  sequential `callAI` calls instead of one:
  1. **Plan** — decompose the brick-wall question / split goal into specific sub-questions.
  2. **Propose** — for each, name concrete record sets, repositories, jurisdictions, eras.
  3. **Reflect** — compare against what the tree + Research Memory already contain; drop
     redundant suggestions, flag gaps.
  4. **Synthesize** — emit the final strict-JSON artifact (candidates / research plan).
- Bounded (hard cap of 4 turns) to control token cost; each step's output feeds the next.
- A streamed status line ("Planning… Proposing records… Reflecting…") gives visible
  progress (adapted from open-deep-research's activity log).

### 3. AA-genealogy specialist persona (from Aetherius persona design)
- A stable system prompt framing the assistant as an **African American genealogy
  specialist**: aware of the pre-1870 "brick wall," Freedmen's Bureau & Freedman's Bank
  records, slave schedules, post-Emancipation surname adoption, Great Migration patterns,
  and the FAN/cluster (reasonably-exhaustive-search) method.
- Stored once as a constant, prepended to every research call (chat + Tree Lab).

### 4. Research Memory in localStorage (from Aetherius tiered memory, no vector DB)
- Two new global (NOT per-tree) localStorage keys:
  - `rr_research_facts` — durable, deduped findings:
    `[{ id, text, person, surname, place, recordType, sources?, addedAt }]`.
  - `rr_research_log` — timestamped episodic summaries:
    `[{ id, when, summary, treeId, mode }]`.
- **Retrieval without embeddings:** tag each entry (person/surname/place/recordType) and
  keyword-match against the current question/tree to inject only the relevant subset into
  `messages`. Cap injected memory (e.g. top 15 facts) to protect the token budget; when the
  store grows large, run a one-shot `callAI` summarization pass to condense it.
- Surfaced in a small "Research Memory" panel in the AI Research Assistant (view / edit /
  delete entries) so the user controls what the AI remembers.

## Guardrails

- **No made-up data leaks into real trees.** Split & Clean-up are reorganize/correct only
  (prompt-enforced + a post-parse check that drops any person whose name is not in the
  source tree for those two modes).
- **Speculative people are quarantined.** `speculative:true` people render with a badge
  and are **excluded from `syncToPocketBase()`** (filter them out before `pbSave`).
- **Provenance.** Every generated tree carries `aiGenerated/mode/sourceTree/createdAt`;
  shown in the tree switcher (small "AI" tag) and the banner.
- **Review-before-save.** No candidate is persisted until the user checks it and clicks Save.
- **Grounded vs. ungrounded labeling.** Results from the Gemini grounded path show their
  real `sources`; Groq-fallback results are badged "ungrounded — verify independently."
  Suggested record citations are never presented as confirmed facts.

## UI

`AI Tree Lab` card:
- Mode `<select>` (4 options) + short helper text per mode.
- `Analyze my tree` button → spinner → results area.
- Results: `summary` paragraph, then one card per candidate:
  - Header: candidate name + line + people count + (mode badge).
  - "Start with **Name** — why".
  - Research checklist (bulleted; each item may show repository + source links, or an
    "ungrounded" badge when sources are absent).
  - Speculative additions listed with a ⚠ badge (hypothesis/competing only).
  - Checkbox "Save this as a new tree".
  - **`📄 Generate Report` button** (works on the candidate, before or after saving).
- Footer: `Save selected (N)` button.
- Live status line during analysis (Plan→Propose→Reflect→Synthesize).

`AI Research Assistant` (existing chat) also gains a small **Research Memory** panel
(view / edit / delete facts) and a grounded/ungrounded indicator on answers.

## Reports (build both)

Reuse the existing report infrastructure: the `Reports` section, `generateReport(type)`
rendering target, `aiGenerateReport()` AI writer, and `window.print()` / markdown export.
No new export plumbing.

**A. Per-candidate report (AI Tree Lab).** Each candidate card has `📄 Generate Report`.
It compiles a **Research Report** for that candidate and renders it in the existing report
preview/print view:
- Title + provenance line (mode, source tree, date).
- Overview (`candidate.summary` / top-level `summary`).
- People list (name, dates, place, parents, line) — speculative people clearly marked ⚠.
- **Who to start with** (`startWith.person` + `why`).
- **Research plan** (the `research[]` checklist).
- Hypothesis/Competing modes: the theory and its reasoning.
Printable/exportable via the existing `window.print()` and markdown export buttons.

**B. Reports-section type.** Add an **"Alternate Tree Report"** entry to the existing
Reports section that runs on whichever tree is active — including any *saved* alternate
tree. For an AI-generated tree it reads the stored `data.aiPlan` (startWith + research +
mode + sourceTree) plus the tree's people to produce the same Research Report layout. For
a non-AI tree it degrades gracefully to a plain people/overview report.

Both paths render through the **same report builder** (one function, e.g.
`buildResearchReport(source)`) so the per-candidate and saved-tree reports are identical
in layout — only the data source differs (candidate object vs. saved tree blob).

## Out of scope (v1)

- Per-tree PocketBase sync for alternate trees (still gated to My Family; alternate trees
  are local until the server adds a `treeId` field — tracked separately).
- Editing a candidate inline before saving (save first, edit in the normal tree UI).
- GEDCOM export of a candidate before saving (export after saving via existing export).

## Testing

- Split a tree with 3+ lines → one candidate per line; each person lands in exactly one
  candidate; cross-line parent names preserved as strings.
- Clean-up on a tree with a known duplicate / conflicting date → flagged, no invented people.
- Hypothesis mode → at least one `speculative:true` person; verify it is excluded from
  `syncToPocketBase()` and badged in the UI.
- Save 2 of 3 candidates → exactly 2 new `rr_trees` entries + 2 blobs; research items
  appear as notes and the banner renders on switch.
- Malformed AI JSON → one retry, then graceful error, no tree created.
- `📄 Generate Report` on a candidate → research report renders with overview, people,
  start-with, and research plan; prints/exports via existing buttons.
- "Alternate Tree Report" on a saved AI tree → same layout, sourced from `data.aiPlan`;
  on a non-AI tree it degrades to a plain people/overview report without errors.
- **Grounding:** with a Gemini key, a research question returns `sources[]` (real URLs)
  and renders them; force the Groq fallback → result badged "ungrounded," no fake sources.
- **Multi-step loop:** status line advances through Plan→Propose→Reflect→Synthesize; loop
  never exceeds 4 turns.
- **Research Memory:** a found fact is written to `rr_research_facts`; a later related
  question injects it into the prompt; memory panel lists/edits/deletes it; keys survive
  reload and are global (not wiped by tree switch).
- **Persona:** research answers reflect AA-genealogy specifics (e.g. suggests Freedmen's
  Bureau / 1870 brick-wall strategy) rather than generic advice.
- SNP/genomic reference DB (106 rsIDs) untouched — verify count unchanged after build.

---

## §4 Implementation — Research Memory (locked 2026-06-24)

Implements design §4. Fact capture is **on-demand + manual** (no silent
auto-extraction): a "Remember this" button runs one extraction call, and the panel
has a manual add form. The episodic log is written automatically (no extra AI call).

### Storage (global keys — NOT per-tree, survive tree switches, never in a tree blob)
- `rr_research_facts` = `[{id, text, person, surname, place, recordType, sources?, addedAt}]`
- `rr_research_log`   = `[{id, when, summary, treeId, mode}]`
- Helpers: `getResearchFacts()/setResearchFacts(a)`, `addResearchFact(f)` (dedupe by
  normalized lowercase `text`), `updateResearchFact(id,patch)`, `deleteResearchFact(id)`,
  `getResearchLog()/addResearchLog(entry)/clearResearchLog()`. IDs via existing id scheme
  (no `Date.now`/`Math.random` reliance for dedupe key — use normalized text).

### Capture
- **`🧠 Remember this`** button rendered under each AI Research Assistant answer →
  one `callAI` extraction → expects JSON array `[{text,person,surname,place,recordType}]`
  (reuse `extractJSON`, tolerate a bare array) → `addResearchFact` each (deduped) →
  toast + `renderResearchMemory()`.
- **`➕ Add fact`** manual form in the panel (text required; person/surname/place/
  recordType optional).
- `treeLabAnalyze` appends `{when, summary:result.summary, treeId:activeTree().id, mode}`
  to `rr_research_log` after a successful analysis.

### Retrieval / injection (keyword match, no embeddings, cap 15)
- `relevantFacts(queryText, cap=15)` — tokenize query, score each fact by overlap of its
  tags (person/surname/place/recordType) + `text` words with the query; return top `cap`.
- **AI Research chat:** in `sendAI('ai-research', …)` prepend a system note
  `"Known facts from the researcher's prior work (treat as notes, not new sources):\n- …"`.
- **Tree Lab:** `runResearchLoop` injects facts relevant to the snapshot's surnames/places.
- **Condense:** `condenseMemory()` — one-shot `callAI` summarization merging facts into a
  smaller deduped set when the store is large (manual button; user confirms before replace).
- Injected facts are always labeled prior-research notes — never presented as confirmed
  citations (no-made-up-data rule).

### Panel UI (in the AI Research Assistant page)
- Collapsible **`🧠 Research Memory`** section:
  - Fact list: each row shows `text` + tag chips, with ✏️ edit (inline) and 🗑️ delete.
  - `➕ Add fact` form.
  - `🧹 Condense memory` button (shown when facts exceed a threshold, ~40).
  - Recent **episodic log** (read-only, last ~10) with a `Clear log` button.
- `renderResearchMemory()` repaints the panel; called on capture/add/edit/delete/condense
  and when navigating to the AI Research page.

### Guardrails
- Facts written ONLY on explicit user action (button or manual add) — never silently.
- Keys are global; excluded from `getAllData()`/tree blobs and from `TRACKED_KEYS`-style
  per-tree sync. Not synced to PocketBase.
- SNP/genomic reference DB (106 rsIDs) untouched — verify count unchanged after build.
