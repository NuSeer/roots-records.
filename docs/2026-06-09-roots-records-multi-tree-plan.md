# Roots & Records — Multi-Tree Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Convert Roots & Records from a single hardcoded-family app into a multi-tree app with a switcher, migrating the user's existing data safely into a "My Family" tree and letting new trees (e.g. "My Children") start empty.

**Architecture:** Namespaced localStorage blobs — a `rr_trees` registry + `rr_active_tree` pointer; each tree's data under `rr_data_v1::<treeId>` (same shape as today's single `rr_data_v1`). A `treeKey()` helper routes all tree-scoped storage access through the active tree. One-time migration moves legacy keys into a `my-family` tree and leaves the originals as backup. The four hardcoded lines become per-tree data living only inside the migrated tree.

**Tech Stack:** Vanilla JS in one HTML file (`roots-and-records-v4.html`); localStorage; PocketBase sync; git (push to `github.com/NuSeer/roots-records.`).

**Spec:** `C:\Users\jfwat\Downloads\roots-and-records-docs\2026-06-09-roots-records-multi-tree-design.md`

**Source of truth:** `C:\Users\jfwat\Downloads\roots-and-records-v4.html` (pulled live from VPS).

**Verification convention:** Open the file via `file:///C:/Users/jfwat/roots-records/roots-and-records-v4.html` in a browser, open DevTools Console, and run the self-check snippet given in each task. "FAIL/PASS" refer to console output. Manual UI checks are spelled out explicitly.

---

### Task 0: Repo + working copy setup (reconcile VPS ↔ GitHub)

**Files:**
- Create (working dir): `C:\Users\jfwat\roots-records\` (fresh clone of the dotted repo)
- Copy in: live `roots-and-records-v4.html`

- [ ] **Step 1: Clone the canonical (dotted) repo**

```bash
cd /c/Users/jfwat
git clone "https://github.com/NuSeer/roots-records..git" roots-records
cd roots-records
git checkout -b multi-tree
```
Expected: clone succeeds; on new branch `multi-tree`. (If the `.git` suffix on a dotted repo name fails, use the URL `https://github.com/NuSeer/roots-records.` without `.git`.)

- [ ] **Step 2: Baseline-sync the repo to the live VPS version**

Copy the live file over whatever stale copy the repo holds, standardizing the filename:
```bash
cp "/c/Users/jfwat/Downloads/roots-and-records-v4.html" "/c/Users/jfwat/roots-records/roots-and-records-v4.html"
git add roots-and-records-v4.html
git status
```
Expected: `roots-and-records-v4.html` staged (modified or new).

- [ ] **Step 3: Commit the baseline**

```bash
git commit -m "chore: sync repo to live VPS v4 (baseline before multi-tree)"
```
Expected: commit created. This makes the diff for all later work meaningful.

- [ ] **Step 4: Copy the spec into the repo**

```bash
mkdir -p docs
cp "/c/Users/jfwat/Downloads/roots-and-records-docs/2026-06-09-roots-records-multi-tree-design.md" docs/
cp "/c/Users/jfwat/Downloads/roots-and-records-docs/2026-06-09-roots-records-multi-tree-plan.md" docs/
git add docs && git commit -m "docs: add multi-tree design spec and plan"
```
Expected: docs committed.

---

### Task 1: Tree registry + namespacing helpers

**Files:**
- Modify: `roots-and-records-v4.html` — add a new `<script>` helper block immediately AFTER the line defining `const SAVE_KEY = 'rr_data_v1'`.

- [ ] **Step 1: Write the helper block**

Insert directly after `const SAVE_KEY = 'rr_data_v1';`:
```javascript
/* ---- Multi-tree registry & namespacing ---- */
const TREES_KEY = 'rr_trees';
const ACTIVE_TREE_KEY = 'rr_active_tree';
// Keys that are scoped per-tree (everything else stays global/account-level):
const TREE_SCOPED_BASES = ['rr_data_v1','rr_default_line','rr_dna_summary','rr_marker_overrides','rr_dna_pin'];

function getTrees(){ try { return JSON.parse(localStorage.getItem(TREES_KEY)) || []; } catch(e){ return []; } }
function setTrees(list){ localStorage.setItem(TREES_KEY, JSON.stringify(list)); }
function getActiveTreeId(){ return localStorage.getItem(ACTIVE_TREE_KEY) || 'my-family'; }
function setActiveTreeId(id){ localStorage.setItem(ACTIVE_TREE_KEY, id); }
function activeTree(){ return getTrees().find(t => t.id === getActiveTreeId()) || null; }
// Namespace a base key to the active tree, e.g. treeKey('rr_data_v1') -> 'rr_data_v1::my-family'
function treeKey(base){ return base + '::' + getActiveTreeId(); }
function slugifyTreeName(name){
  return (name||'').toLowerCase().trim().replace(/[^a-z0-9]+/g,'-').replace(/^-+|-+$/g,'').slice(0,40) || ('tree-' + getTrees().length);
}
```

- [ ] **Step 2: Self-check (FAIL expected before reload)**

In console on the OLD file: `typeof treeKey` → `"undefined"`. Expected: FAIL (not yet present).

- [ ] **Step 3: Reload and verify helpers exist**

Run in console after reload:
```javascript
console.assert(treeKey('rr_data_v1') === 'rr_data_v1::my-family', 'treeKey default');
console.log('OK', getTrees(), getActiveTreeId());
```
Expected: PASS (no assertion error), logs `OK [] my-family`.

- [ ] **Step 4: Commit**

```bash
git add roots-and-records-v4.html && git commit -m "feat: add multi-tree registry and namespacing helpers"
```

---

### Task 2: One-time migration of legacy data into "My Family"

**Files:**
- Modify: `roots-and-records-v4.html` — add `migrateToMultiTree()` in the helper block (after Task 1 code) and call it ONCE at the very top of the existing startup/init path (before the first `loadSavedData()` / data read).

- [ ] **Step 1: Write the migration function**

Append to the helper block:
```javascript
function migrateToMultiTree(){
  if (localStorage.getItem(TREES_KEY)) return; // already migrated
  const legacy = localStorage.getItem('rr_data_v1');
  const id = 'my-family';
  setTrees([{ id, name: 'My Family', createdAt: new Date().toISOString() }]);
  setActiveTreeId(id);
  // Copy legacy tree-scoped keys into the namespaced slots; LEAVE originals as backup.
  TREE_SCOPED_BASES.forEach(base => {
    const v = localStorage.getItem(base);
    if (v !== null && localStorage.getItem(base + '::' + id) === null) {
      localStorage.setItem(base + '::' + id, v);
    }
  });
  console.log('[migrate] multi-tree initialized; legacy keys preserved as backup', { hadLegacyData: legacy !== null });
}
```

- [ ] **Step 2: Call it at startup**

Find the existing initialization (the first place that calls `loadSavedData()` on load — search for `loadSavedData(`). Insert `migrateToMultiTree();` on the line immediately BEFORE that first call.

- [ ] **Step 3: Self-check with simulated legacy data (FAIL→PASS)**

In console:
```javascript
localStorage.removeItem('rr_trees'); localStorage.removeItem('rr_active_tree');
localStorage.setItem('rr_data_v1', JSON.stringify({people:[{name:'TEST ANCESTOR'}]}));
migrateToMultiTree();
const moved = JSON.parse(localStorage.getItem('rr_data_v1::my-family'));
console.assert(moved && moved.people[0].name==='TEST ANCESTOR', 'data migrated');
console.assert(localStorage.getItem('rr_data_v1')!==null, 'legacy backup preserved');
console.assert(getActiveTreeId()==='my-family', 'active set');
console.log('MIGRATION OK');
```
Expected: `MIGRATION OK`, no assertion failures.

- [ ] **Step 4: Commit**

```bash
git add roots-and-records-v4.html && git commit -m "feat: migrate legacy single-tree data into My Family tree (non-destructive)"
```

---

### Task 3: Route tree-scoped storage through the active tree

**Files:**
- Modify: `roots-and-records-v4.html` — every read/write of the tree-scoped bases must go through `treeKey()`.

- [ ] **Step 1: Replace SAVE_KEY storage access**

Search for every occurrence of `localStorage.getItem(SAVE_KEY)`, `localStorage.setItem(SAVE_KEY,`, and `localStorage.removeItem(SAVE_KEY)`. Replace `SAVE_KEY` with `treeKey(SAVE_KEY)` in those storage calls ONLY (leave the `const SAVE_KEY` definition intact). Confirmed call sites from recon: the save path (`...setItem(SAVE_KEY, JSON.stringify(data))` and `...setItem(SAVE_KEY, str)`) and the load path (`...getItem(SAVE_KEY)`).

- [ ] **Step 2: Replace other tree-scoped key access**

For each of `rr_default_line`, `rr_dna_summary`, `rr_marker_overrides`, `rr_dna_pin`: find their `localStorage.getItem/setItem` calls and wrap the literal in `treeKey(...)`, e.g. `localStorage.getItem('rr_default_line')` → `localStorage.getItem(treeKey('rr_default_line'))`. (These are referenced via the constants `DNA_PIN_KEY` and string literals — wrap each at its storage call site.)

- [ ] **Step 3: Self-check round-trip per tree**

In console:
```javascript
setActiveTreeId('my-family');
localStorage.setItem(treeKey(SAVE_KEY), JSON.stringify({people:[{name:'FAM'}]}));
setActiveTreeId('kids');
localStorage.setItem(treeKey(SAVE_KEY), JSON.stringify({people:[{name:'KID'}]}));
setActiveTreeId('my-family');
console.assert(JSON.parse(localStorage.getItem(treeKey(SAVE_KEY))).people[0].name==='FAM','fam isolated');
setActiveTreeId('kids');
console.assert(JSON.parse(localStorage.getItem(treeKey(SAVE_KEY))).people[0].name==='KID','kid isolated');
console.log('ISOLATION OK'); setActiveTreeId('my-family');
```
Expected: `ISOLATION OK`.

- [ ] **Step 4: Manual regression**

Reload the file. Confirm your existing "My Family" people/records/timeline still render (they now load from `rr_data_v1::my-family`).

- [ ] **Step 5: Commit**

```bash
git add roots-and-records-v4.html && git commit -m "feat: route tree-scoped storage through active tree"
```

---

### Task 4: De-hardcode the four family lines

**Files:**
- Modify: `roots-and-records-v4.html` — line list, key→label map, color vars, AI-interview prompts, chromosome-painting markup.

> **PRESERVATION GUARDRAIL (African American genomic references):** This task touches ONLY family-line *names, keys, and colors*. The SNP→risk reference database (~115 rsIDs: APOL1, G6PD, TCF7L2, SMAD7, HBB/sickle, 8q24, HLA-DRB1, rs2814778 Duffy, etc.), the African-ancestry SNP sets (West/East/Central African; Yoruba/Ewe/Mandinka/Bantu), 1000 Genomes superpopulation logic, and maternal/paternal haplogroup markers MUST remain byte-identical. Do not edit any `rs#######`, gene name, interpretation string, `AFRICAN_SNPS.*`, `MATERNAL_MARKERS`, or `PATERNAL_MARKERS`. Verify with the Task 7 reference-count check.

- [ ] **Step 1: Make the lines list per-tree**

Find `const lines = ['Unknown','Foreman / Drewitt (Lawson)','Smith (Elsie)','Reid (Charlie)','Smith (Erma)','Composite'];`. Replace the hardcoded named lines with a per-tree lookup that falls back to structural-only:
```javascript
function getLines(){
  let data = {};
  try { data = JSON.parse(localStorage.getItem(treeKey(SAVE_KEY))) || {}; } catch(e){}
  const custom = Array.isArray(data.lines) ? data.lines : [];
  return ['Unknown', ...custom, 'Composite'];
}
```
Replace usages of the old `lines` array with `getLines()`. New/empty trees therefore show only `['Unknown','Composite']`.

- [ ] **Step 2: Seed the four lines into the migrated My Family tree only**

In `migrateToMultiTree()` (Task 2), after copying legacy data, if the migrated blob has no `lines`, seed the originals so the user's existing color-coding is preserved:
```javascript
const k = 'rr_data_v1::' + id;
let d = {}; try { d = JSON.parse(localStorage.getItem(k)) || {}; } catch(e){}
if (!Array.isArray(d.lines)) {
  d.lines = ['Foreman / Drewitt (Lawson)','Smith (Elsie)','Reid (Charlie)','Smith (Erma)'];
  localStorage.setItem(k, JSON.stringify(d));
}
```

- [ ] **Step 3: Replace hardcoded color vars with a generated palette**

Find CSS `--foreman / --smith1 / --reid / --smith2` and the key→label map (`'foreman':'Foreman / Drewitt (Lawson)'`, etc.) and the `foreman:'var(--foreman)'` color map. Replace with a function that assigns a color by line index from a fixed palette:
```javascript
const LINE_PALETTE = ['#c47a2a','#2d9e6a','#5a7adb','#8b5bbf','#d4607a','#3a9458','#c49030','#9e2850'];
function lineColor(lineName){
  const idx = getLines().indexOf(lineName);
  if (lineName==='Unknown') return 'var(--text3)';
  if (lineName==='Composite') return 'var(--gold)';
  return LINE_PALETTE[(idx<0?0:idx) % LINE_PALETTE.length];
}
```
Replace references to the old per-surname color vars with `lineColor(<lineName>)`.

- [ ] **Step 4: Genericize the AI-interview prompts**

Find the four `onclick="askAI('ai-interview', ...)"` prompts that name "Foreman/Drewitt line" and "Reid and Smith (Erma) line". Replace the hardcoded-surname prompts with line-aware text driven by `getLines()` (e.g., "I want to talk about memories from my " + lineName + " line"), or generic "father's side"/"mother's side" wording. No surnames hardcoded.

- [ ] **Step 5: Self-check**

```javascript
setActiveTreeId('kids'); localStorage.setItem(treeKey(SAVE_KEY), JSON.stringify({}));
console.assert(JSON.stringify(getLines())===JSON.stringify(['Unknown','Composite']),'new tree empty lines');
setActiveTreeId('my-family');
console.assert(getLines().includes('Reid (Charlie)'),'my-family seeded lines');
console.log('LINES OK');
```
Expected: `LINES OK`.

- [ ] **Step 6: Commit**

```bash
git add roots-and-records-v4.html && git commit -m "feat: de-hardcode family lines; per-tree lines + generated palette"
```

---

### Task 5: Tree switcher UI

**Files:**
- Modify: `roots-and-records-v4.html` — sidebar header markup + new functions.

- [ ] **Step 1: Add the switcher markup**

In the sidebar header next to the "🌿 Roots &amp; Records" title, insert:
```html
<div id="tree-switcher" style="margin:8px 0;">
  <select id="tree-select" onchange="switchTree(this.value)"
    style="width:100%;background:var(--sidebar2);color:var(--text);border:1px solid var(--div);border-radius:6px;padding:6px;font-family:inherit;"></select>
  <div style="display:flex;gap:4px;margin-top:4px;">
    <button onclick="newTree()" title="New tree" style="flex:1;">＋</button>
    <button onclick="renameTree()" title="Rename" style="flex:1;">✎</button>
    <button onclick="deleteTree()" title="Delete" style="flex:1;">🗑</button>
  </div>
</div>
```

- [ ] **Step 2: Add switcher functions**

In the helper block:
```javascript
function renderTreeSwitcher(){
  const sel = document.getElementById('tree-select'); if(!sel) return;
  const trees = getTrees(), active = getActiveTreeId();
  sel.innerHTML = trees.map(t => `<option value="${t.id}" ${t.id===active?'selected':''}>${t.name}</option>`).join('');
}
function switchTree(id){ setActiveTreeId(id); location.reload(); }
function newTree(){
  const name = prompt('Name this tree (e.g. "My Children", "Smith Clients"):'); if(!name) return;
  let id = slugifyTreeName(name); const trees = getTrees();
  while (trees.some(t=>t.id===id)) id += '-' + (trees.length+1);
  trees.push({ id, name, createdAt: new Date().toISOString() }); setTrees(trees);
  setActiveTreeId(id);
  localStorage.setItem(treeKey(SAVE_KEY), JSON.stringify({})); // empty tree
  location.reload();
}
function renameTree(){
  const t = activeTree(); if(!t) return;
  const name = prompt('Rename tree:', t.name); if(!name) return;
  const trees = getTrees().map(x => x.id===t.id ? {...x, name} : x); setTrees(trees); renderTreeSwitcher();
}
function deleteTree(){
  const trees = getTrees(); if(trees.length<=1){ alert('Cannot delete your only tree.'); return; }
  const t = activeTree(); if(!t) return;
  if(!confirm(`Delete tree "${t.name}"? This removes its people, records, and DNA. Export to GEDCOM first if you want a backup.`)) return;
  TREE_SCOPED_BASES.forEach(base => localStorage.removeItem(base + '::' + t.id));
  const remaining = trees.filter(x=>x.id!==t.id); setTrees(remaining);
  setActiveTreeId(remaining[0].id); location.reload();
}
```
Call `renderTreeSwitcher();` at the end of the startup/init path (after `migrateToMultiTree()` and initial render).

- [ ] **Step 3: Manual UI verification**

Reload. Expected: dropdown shows "My Family". Click ＋, name it "My Children" → page reloads, dropdown shows "My Children", and all views are empty (no people, lines = Unknown/Composite only). Switch back to "My Family" → your real data is intact.

- [ ] **Step 4: Self-check isolation after add**

In "My Children", add one person via the normal UI, then in console:
```javascript
setActiveTreeId('my-family');
const fam = JSON.parse(localStorage.getItem(treeKey(SAVE_KEY))||'{}');
console.assert(!(fam.people||[]).some(p=>/new person/i.test(p.name||'')),'kid not leaking into family');
console.log('SWITCHER OK'); location.reload();
```
Expected: `SWITCHER OK`.

- [ ] **Step 5: Commit**

```bash
git add roots-and-records-v4.html && git commit -m "feat: tree switcher UI (new/rename/delete + switch)"
```

---

### Task 6: Per-tree PocketBase sync

**Files:**
- Modify: `roots-and-records-v4.html` — `syncToPocketBase()` and `loadFromPocketBase()`.

- [ ] **Step 1: Read the current sync functions**

Locate `function syncToPocketBase(){...}` and `function loadFromPocketBase(){...}`. Identify the record identifier they use (the fixed key/record id for the single tree).

- [ ] **Step 2: Key the server record by tree id**

Modify both functions so the PocketBase record id / filter incorporates `getActiveTreeId()` (e.g., a record per `roots_tree_<treeId>` or a `treeId` field in the filter), and so they read/write `localStorage.getItem(treeKey(SAVE_KEY))` rather than the global `SAVE_KEY`. Preserve the existing auth/endpoint logic; change only the record selector and the local key.

- [ ] **Step 3: Manual verification**

In "My Family", click "☁️ Sync to Server" → success message. Switch to "My Children", click Sync → success. Reload, click "📥 Load from Server" in each tree → each loads only its own data. Expected: no cross-contamination; My Family ≠ My Children on the server.

- [ ] **Step 4: Commit**

```bash
git add roots-and-records-v4.html && git commit -m "feat: per-tree PocketBase sync keyed by tree id"
```

---

### Task 7: Full verification pass, deploy, and push

**Files:** none new.

- [ ] **Step 1: Regression sweep**

Reload "My Family". Visit each major view (members, records, timeline, pedigree, photos, dna, leeds, census, newspaper, archives, ai-interview, reports). Confirm each renders without console errors and reflects the active tree.

- [ ] **Step 2: Migration safety re-check**

In console: `console.log(localStorage.getItem('rr_data_v1'))` → still returns your original blob (backup preserved). Expected: non-null.

- [ ] **Step 3: Push the branch to the dotted repo**

```bash
cd /c/Users/jfwat/roots-records
git push -u origin multi-tree
```
Expected: branch pushed to `github.com/NuSeer/roots-records.`.

- [ ] **Step 4: Deploy to VPS (ASK before running)**

Confirm with the user first (this overwrites the live `/roots/` page). On approval:
```bash
scp -i ~/.ssh/vps_key "/c/Users/jfwat/roots-records/roots-and-records-v4.html" root@187.124.146.184:/var/www/html/roots/roots-and-records-v4.html
```
Expected: live page now multi-tree. Hard-refresh `/roots/` and confirm the switcher appears and existing data is intact.

- [ ] **Step 5: Open the dotted-repo PR / merge**

Create a PR from `multi-tree` to the default branch (or merge directly per user preference).

---

## Self-Review

**Spec coverage:**
- §3 storage model → Tasks 1, 3 ✓
- §3.1 migration (non-destructive, leave backup) → Task 2 ✓
- §4 switcher (new/rename/delete, empty new trees, lines only in My Family) → Tasks 4, 5 ✓
- §7 untouched features read active tree → Task 3 (storage routing) + Task 7 regression ✓
- §8 per-tree PocketBase sync → Task 6 ✓
- §10 verification (migration, new-tree, isolation, regression) → Tasks 2,3,5,7 ✓
- liberu features (§5) → intentionally **out of scope for this plan**; Plan 2. ✓
- GEDCOM export referenced in delete-warning (Task 5) is a *message only* here; actual GEDCOM lands in Plan 2 — no code dependency. ✓

**Placeholder scan:** No TBD/TODO. Existing-code touch points (Tasks 3, 6) instruct locating confirmed call sites and give the exact transformation; helper/migration/switcher code is complete.

**Type consistency:** `treeKey`, `getTrees`, `setTrees`, `getActiveTreeId`, `setActiveTreeId`, `activeTree`, `getLines`, `lineColor`, `slugifyTreeName`, `renderTreeSwitcher`, `switchTree`, `newTree`, `renameTree`, `deleteTree`, `migrateToMultiTree`, `TREES_KEY`, `ACTIVE_TREE_KEY`, `TREE_SCOPED_BASES`, `LINE_PALETTE` — names consistent across all tasks.
