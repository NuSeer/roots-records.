# Roots & Records — Multiple Trees + liberu-Inspired Features

**Date:** 2026-06-09
**Target file:** `C:\Users\jfwat\Downloads\roots-and-records-v4.html` (pulled from the VPS — the **source of truth**; deployed at VPS `/var/www/html/roots/`, served at `/roots/`)
**Canonical GitHub repo (push target):** `github.com/NuSeer/roots-records.` (the **dotted** repo). The non-dotted `roots-records` and the VPS-vs-GitHub staleness are reconciled by building on the live VPS v4 and pushing the result up to the dotted repo. Spec is also committed there.
**Status:** Design — APPROVED 2026-06-09; proceeding to implementation plan

---

## 1. Goal

Stop Roots & Records (R&R) from being hardwired to one family (the four hardcoded grandparent lines). Make it a **multi-tree** app with a switcher, so the user can keep their own family tree, build a separate tree for their children, and eventually run paying genealogy clients (the Sankofa direction). Each tree is its own isolated workspace that **seeds fresh as data is added** — no hardcoded people or lines in new trees.

Secondary: selectively incorporate portable, high-value features from
`liberu-genealogy/genealogy-laravel`, adapted to R&R's single-file client-side architecture.

**Non-negotiable:** the user's existing real family data (already in their browser localStorage / synced to PocketBase) must migrate intact. Nothing gets wiped.

---

## 2. Current Architecture (as-is, from code recon)

- Single ~700 KB HTML file, all logic inline.
- Entire tree stored as one JSON blob: `localStorage['rr_data_v1']` (constant `SAVE_KEY`).
- Synced to PocketBase via `syncToPocketBase()` / `loadFromPocketBase()`.
- Four family lines hardcoded:
  `const lines = ['Unknown','Foreman / Drewitt (Lawson)','Smith (Elsie)','Reid (Charlie)','Smith (Erma)','Composite']`
  plus a key→label map, AI-interview prompts naming the lines, and CSS color vars `--foreman / --smith1 / --reid / --smith2`.
- ~40 views via `nav('<id>')`: home, glance, family members, records, timeline, notes, hypotheses, pedigree, photos, reports, progress, upload, importexport, dna, dna-analysis, leeds, matchtimeline, composite, network, census, soundex, migration, newspaper, archives, ai-research, ai-brick, ai-dna, ai-story, ai-interview, quotes, book, reunion, legacy, recorder, scratch, calendar, focus, settings, alerts.
- Other keys: `rr_auth`, `rr_auth_time`, `rr_default_line`, `rr_dna_summary`, `rr_dna_pin`, `rr_ensembl_cache`, `rr_groq_key`, `rr_marker_overrides`, `rr_music_pos`, `rr_playlists`, `rr_user_name`.

---

## 3. Multi-Tree Storage Model

**Approach:** namespaced localStorage blobs (minimal, low-risk change to a working app — not an IndexedDB rewrite).

- `rr_trees` → `[{ id, name, createdAt }]` — the registry.
- `rr_active_tree` → id of the currently open tree.
- Per-tree data blob: `rr_data_v1::<treeId>` (identical shape to today's `rr_data_v1`).
- Tree-scoped settings namespaced per tree: `rr_default_line::<id>`, `rr_dna_summary::<id>`, `rr_marker_overrides::<id>`, `rr_dna_pin::<id>`.
- **Account/global keys stay global** (shared across trees): `rr_auth`, `rr_auth_time`, `rr_groq_key`, `rr_user_name`, `rr_music_pos`, `rr_playlists`, `rr_ensembl_cache`.

### 3.1 Migration (data safety — critical)

On first load of the new version:
1. If `rr_trees` is absent **and** legacy `rr_data_v1` exists:
   - Create tree `{ id:'my-family', name:'My Family', createdAt: now }`.
   - Copy `rr_data_v1` → `rr_data_v1::my-family`. Copy legacy tree-scoped settings → `::my-family` namespaced equivalents.
   - Set `rr_active_tree = 'my-family'`.
   - **Leave the legacy keys in place as an untouched backup** (do not delete).
2. If `rr_trees` is absent and no legacy data: create an empty default tree `{ id:'my-family', name:'My Family' }` with no lines/people.

Result: existing family data appears as the "My Family" tree, unchanged; old keys remain as a recovery backup.

---

## 4. Switcher UI

- Dropdown in the sidebar header (next to "🌿 Roots & Records"): shows active tree name; lists all trees; actions **New Tree**, **Rename**, **Delete** (delete asks for confirmation and warns about data loss; offers GEDCOM export first — see §5.1).
- Selecting a tree sets `rr_active_tree` and re-renders all views from that tree's blob.
- **New tree starts empty:** no lines, no people. Lines and people are created on add ("seed on add fresh").
- The four original lines exist **only inside the migrated "My Family" tree**, where the user can edit/clear them. They are no longer hardcoded anywhere in the app.

---

## 5. liberu-Inspired Features (incorporated, adapted to client-side)

### 5.1 GEDCOM import / export (HIGH VALUE)
- Client-side GEDCOM 5.5.1 parser → maps individuals, families (FAM), relationships (HUSB/WIFE/CHIL), events (BIRT/DEAT/MARR), sources into the R&R data model and the active tree.
- Export: generate a `.ged` file from the active tree.
- Surfaced in the existing **Import/Export** view. Enables importing an existing tree instead of retyping, and exporting before destructive actions (tree delete).

### 5.2 Relationship / family data model
- Ensure people support structured links: parents, spouse(s), children; plus family + event entities. This is the backbone for actually completing a tree and for GEDCOM round-tripping. Audit current model first; extend where flat.

### 5.3 Privacy / living-person redaction (IMPORTANT — `/roots/` is public)
- Per-person `living` / `private` flag (default: living = no death date and birth within ~100 yrs).
- A **Public View** toggle that redacts living persons (name → "Living", hide details) in views and in any shared/exported public output. Especially relevant for the children's tree (living people on a public URL).

### 5.4 Charts
- Add **descendant** and **fan** charts (R&R already has pedigree + network + composite).

### 5.5 Per-person research checklist
- A checklist of standard research tasks per person/line, tracked alongside existing notes/progress/hypotheses.

### 5.6 Media attachments on events
- Extend existing photos to allow attaching media (image/doc/audio) to events, not just people.

---

## 6. Re-scoped / Deferred (do NOT port literally)

- **Facial-recognition photo tagging** — optional stretch via a CDN face library; flagged, NOT in core scope.
- **Live external-service API integrations** (MyHeritage / Ancestry / FamilySearch / FindMyPast) and **ML match-scoring background job queue** — require a server + paid API keys. Re-scoped as: **deep-link search launchers** (same pattern R&R already uses for newspapers/archives) + **on-demand AI match scoring via the existing Groq assistants**. No server, same user benefit.

---

## 7. Untouched (user's unique features — read/write active tree only)

DNA, DNA-analysis, Leeds, match-timeline, composite, network, census, soundex, migration, newspaper, archives, AI research/brick/dna/story/interview, quotes, book, reunion, legacy, recorder, scratch, calendar, focus, music, alerts. Logic unchanged; they simply read/write the **active tree's** blob instead of the single global blob.

---

## 8. PocketBase Sync

- Per-tree sync: each tree syncs to its own server record keyed by tree id, so trees never overwrite each other. Exact PB collection/field mapping to be confirmed when wiring `syncToPocketBase()` / `loadFromPocketBase()` to accept a tree id.

---

## 9. Out of Scope for This Pass (separate follow-on work)

- **Sankofa feature comparison** — produce a gap list (R&R vs Sankofa Next.js app); ASK before incorporating anything; unique features untouched.
- **Debug pass** — find/fix bugs; ASK before changing any unique feature.

---

## 10. Verification

- Migration test: load with a pre-existing `rr_data_v1` → confirm "My Family" tree has all prior people/records and legacy keys remain.
- New-tree test: create "My Children" → confirm empty, independent; adding a person does not appear in "My Family".
- GEDCOM round-trip: export a tree, re-import into a new tree, compare individuals/relationships.
- Privacy: toggle Public View → living persons redacted everywhere.
- Regression: spot-check each untouched view renders from the active tree.

---

## 11. Plan 2 — Feature Roadmap (additive, non-destructive; built AFTER multi-tree)

All Plan 2 work is per-tree and MUST honor the same guardrails: do NOT touch the DNA/SNP engine or the African American genomic references (~102 rsIDs), and preserve every existing archival source. Backend = the user's VPS (`187.124.146.184`, PocketBase + node) and/or Vercel. **Sankofa is NOT the user's app and is out of scope.**

**P2.1 — Verified African American research sources (wire-in)**
Add as pre-filled deep-link launchers in the Archives/Newspaper view (verified live 2026-06-09):
- Last Seen / Information Wanted — `informationwanted.org/items/browse?search=TERM`
- Freedom on the Move — `database.freedomonthemove.org`
- Enslaved.org — `enslaved.org/search/people?q=TERM`
- DocSouth (North American Slave Narratives, UNC) — `docsouth.unc.edu`
- Lowcountry Africana — `lowcountryafricana.com`
- NPS Civil War Soldiers & Sailors (USCT) — `nps.gov/civilwar/search-soldiers.htm`
- Mapping the Freedmen's Bureau — `mappingthefreedmensbureau.com`
- (AfriGeneas — re-verify before adding; connection refused on first check.)

**P2.2 — African American Research Agent (FLAGSHIP) — BOTH tiers, Gemini-powered**
- **Model:** Gemini (default `gemini-2.5-flash`; `gemini-2.5-pro` for deep runs) — matches the H.E.L.P. Center AI stack. API key in the VPS `.env`, never in client JS. (Groq/Claude available as fallbacks if desired.)
- **UI:** new agent panel (or upgrade `ai-research` view). Pick a target ancestor from the active tree.
- **Flow:** client gathers active-tree data + target ancestor → POST to a new VPS endpoint (e.g. `/api/aa-agent`) on the node backend → server calls Gemini.

  **Tier 1 — Guided Strategist:** multi-specialist synthesis (Records · Slave-era & Freedmen's · DNA/clusters · Historical context · Brick-wall strategy) → one methodology-aware plan (1870 brick wall, slave schedules, Freedmen's Bureau/Bank, cohabitation records, last-enslaver identification, DNA+paper-trail). Output = prioritized steps, each with a one-click **pre-filled deep-link** into the P2.1 verified sources (seeded with ancestor name/place).

  **Tier 2 — Autonomous Digger:** uses **Gemini + Google Search grounding** to actually search the open-access sources and return **cited candidate records/leads**. Every lead is *proposed* for user confirmation — never written to the tree automatically. Open-access sources only; honor each site's ToS/rate limits.
- **Findings** (confirmed) log to the active tree's research notes/checklist. **Privacy:** living-person redaction respected in any shared output.

**P2.3 — GEDCOM 5.5.1 + GedcomX import/export** (native JS; GedcomX = FamilySearch interop). Mirror tag coverage of `liberu/laravel-gedcom`. Export offered before destructive tree actions.

**P2.4 — Interactive family tree** via the MIT `family-chart` library (donatso/family-chart): pan/zoom, editable, bound to the active tree. (Not magicsunday's lib — GPL + build-coupled.)

**P2.5 — Tree validation / quality checks** — flag missing parents, impossible/contradictory dates, orphaned people, likely duplicates (idea from obsidian-canvas-roots).

**P2.6 — Fan + descendant charts** (R&R already has pedigree/network/composite).

**P2.7 — CSV import** (bulk entry) and **P2.8 — kinship/relationship calculator**.

**P2.9 — Family Book** (per-tree): compile a tree's people + photos + stories/quotes + timeline + sources into a printable/exportable book (HTML→PDF), with living-person privacy applied to shared/public versions. (Check/upgrade the existing `book` view first.)

**Out of scope (server/future):** real multi-user/collaborative trees (would be built on VPS+PocketBase or Vercel); facial-recognition photo tagging; live external-API integrations (MyHeritage/Ancestry/FindMyPast → deep-link launchers instead).
