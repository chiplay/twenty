# Sightmap evaluation against Twenty CRM

Skeptical third-party evaluation. Nothing was taken on faith; every claim below was
exercised against a running instance of Twenty (this repo) unless marked untestable.

**Evaluated:**
- sightmap.org and docs.sightmap.org (33 pages per llms.txt), read 2026-07-31
- Spec repo github.com/sightmap/sightmap @ `95b2040c512eb9b52714d16637b32f713d702a0d` ("Version Packages (#108)", 2026-07-31, tag `go/v0.17.0`)
- CLI `@sightmap/sightmap` **0.17.0** via `npm install -g`
- Test bed: Twenty CRM monorepo, dev seed workspace (599 companies, 1000+ people), frontend at localhost:3001, NestJS GraphQL at localhost:3000
- Environment: Linux container, root user, headless, Node 22/24

**Verdict in one paragraph.** The spec is real, coherent, and unusually well written; the
CLI's authoring loop (snapshot/coverage/sel-probe/validate) genuinely works and took a
2,525-interactive-node data grid from 0% to 100% named coverage in about 12 minutes.
But the consumption story — the actual value proposition — did not survive contact:
in six controlled fresh-context trials, agents given the corpus were consistently
**~2x slower, ~30% more expensive in tokens, and made more wrong turns** than a
baseline agent with a bare accessibility tree, with identical 6/6 answer accuracy.
Two implementation defects (annotated-tree elision on large tables, and
`snapshot --url` re-navigation races) erased the corpus's advantages, and the app's
search box beat semantic navigation every time. Memory worked exactly as designed
once — and steered the agent onto a slower path. The reference implementation also
violates its own spec's versioning MUST in a silent way. Details and reproduction
below.

---

## Phase 1 — Claims table

| # | Claim | Where stated | Testable | Result |
|---|-------|--------------|----------|--------|
| C1 | `npm install -g @sightmap/sightmap` gives a working CLI | landing, start/install | yes | **PASS** — 2.7 s, `sightmap version 0.17.0` |
| C2 | `sightmap skills install` writes both skills to `~/.agents/skills` | start/quickstart | yes | **PASS** |
| C3 | `browser start` launches Chrome + corpus server; YAML hot-reloads | quickstart, authoring | yes | **PASS on the happy path; FAIL on Linux after `browser install`** (see D1) and fails opaquely as root (D2) |
| C4 | `snapshot --coverage` reports T1/T2/T3 and suggests selectors for unlabeled clusters | quickstart, cli/coverage | yes | **PASS** — suggestions correct and actionable; per-row UUID testids produce noisy suggestions |
| C5 | `sel-probe` prints live + offline counts, warns on divergence | authoring skill | yes | **PASS** — code re-counts true live total; but the `matches:` line prints the display-capped count (default `--max 10`) with no "of N" (D8) |
| C6 | `validate` exits nonzero on errors; warnings pass | quickstart | yes | **PASS for field/ref/conflict checks; FAIL on version enforcement** (D3) |
| C7 | Every file must begin `version: 1`; schema has `const: 1`, `required` | spec/v1/schema.md | yes | **FAIL in reference impl** — `version: 2` and missing `version` both validate silently (`go/sightmap/loader.go:43` parses, nothing checks) |
| C8 | Most-specific-wins route matching with per-segment scores | spec | yes | **PASS** — `/objects/tasks` (literal) beat `/objects/*` live; `discover` resolves precedence correctly |
| C9 | Child selectors scoped to parent subtree | spec | yes | **PASS** — observed throughout authoring |
| C10 | Memory surfaces in a `[Guide]`/header section when parent is active | spec, what-is-sightmap | yes | **PASS for views/components** in snapshot header; **FAIL for requests** (D6) |
| C11 | Component queries resolve on extracted properties; 0 matches errors; N matches errors with candidates | browser skill | yes | **PASS** — disambiguation output is excellent (`3 components match: Google, GlobalLogic, Genesys`) |
| C12 | `click` refuses covered/off-screen targets loudly | browser skill | yes | **PASS** — "covered by another element at its center" |
| C13 | `fill` detects not-stuck values, suggests `--clear` | browser skill | not fully | Untested against a failing input; fill on the search box worked |
| C14 | `wait-for --view/--component/--selector/--url` | browser skill | yes | **PASS mostly** — but `wait-for --component` timed out twice in trials while the target was arguably present; timeout is loud |
| C15 | Console/network collector, ~1000-entry buffers | browser skill | yes | **PASS**, but on a Vite dev server the buffer floods instantly (29k+ dropped), evicting app API calls (D7) |
| C16 | Union-aware coverage across a view's capture set | authoring skill | partially | Set machinery works (novelty gate, presence lines); not deeply exercised with many captures |
| C17 | Novelty gate skips captures that add nothing | authoring skill | partially | First captures always wrote; gate not triggered in this eval |
| C18 | 0-interactive-node page → `∅` + non-zero exit | authoring skill | yes | **PASS** — fired repeatedly during SPA hydration; **but a partially rendered page (23 nodes of 2,525) passes silently** (D9) |
| C19 | Conflicts: first-match-wins + loud `[Conflicts]`; equal-specificity views flagged | skill, validate | yes | **PASS** — per-node conflict lines at runtime, clear winner named offline |
| C20 | Corpus lets agents act by semantic identity instead of guessed selectors | what-is-sightmap | yes | **PASS mechanically** (component-query click worked one-shot), **FAIL as a net win** (Phase 4: slower than baseline) |
| C21 | Memory persists quirks so the next agent doesn't rediscover them | landing, spec/memory | yes | **PASS mechanically, mixed value** — B3 followed the memory path correctly; the path was slower than search, and one memory entry caused a wrong turn |
| C22 | Plain YAML, versionable, no hosted service | what-is-sightmap | yes | **PASS** |
| C23 | `validate`/`lint` for CI | landing | yes | **PARTIAL** — validate exits 1 correctly; `lint` exits 0 even when the corpus is unparseable (D4) |
| C24 | `browser install` fetches Chrome for Testing | cli/browser | yes | **PASS**, but `--help` on it executes the 184 MB install (D5), and on Linux `start` can't find what `install` installed (D1) |
| C25 | `:not/:is/:where/:has` supported; sibling combinators not | authoring skill | partially | `:has()` accepted by validate; not exercised live |
| C26 | `:param` normalized to `*` in request routes | spec | yes | PASS at validate level; request runtime consumption is absent anyway (D6) |
| C27 | Properties extracted live; offline omits values | spec | yes | **PASS** — offline snap greps show structure without values |
| C28 | Tag resolution = union across applicable definitions | spec | no (not exercised) | Untested |
| C29 | Integer spec stream, pre-1.0 tightening | VERSIONING.md | policy | n/a |
| C30 | Conformance fixtures | spec/conformance | yes | 23 fixtures present (numbering gaps: 009, 018+, 107) |
| C31 | "Agents add memory notes" | landing | yes | PASS — I added 10 memory entries during authoring; they round-trip |
| C32 | Skills teach the loop | quickstart | yes | PASS — skills are detailed and accurate about their own gotchas |
| C33 | `discover` classifies routes mapped/surveyed/unseen | cli/corpus | yes | **PASS** — correctly labeled 12 routes, spotted `/page/*` unseen |
| C34 | `search` skips view/request fields | cli/corpus | yes | PASS (documented, still surprising: view names return "no matches", exit 0) |
| C35 | Originated in Subtext/FullStory | what-is-sightmap | no | provenance |

Unfalsifiable marketing language is rare; the docs are unusually concrete. The one
aggregate claim that fails empirically is the implied one: that a corpus makes agents
better at using the app today (see Phase 4).

---

## Phase 2 — Install and first run

Published instructions only:

```
$ npm install -g @sightmap/sightmap     # 2.7 s real
$ sightmap version                      # sightmap version 0.17.0
$ sightmap skills install               # installed 2 skills → /root/.agents/skills
$ sightmap browser install              # 184 MB Chrome for Testing 151.0.7922.71, worked through proxy
```

Time to first useful output (first accessibility-tree snapshot of the running app):
**148 s from first `browser start` attempt**, nearly all spent on two environment
failures the docs don't cover:

1. **D1 — `browser install` → `browser start` is broken end-to-end on Linux.**
   `install` puts Chrome in `~/.sightmap/browsers/`; `start` on Linux only does
   `exec.LookPath` over `google-chrome|google-chrome-stable|chromium|chromium-browser`
   (`go/browser/launcher.go:151-158`). The macOS branch checks the sightmap-managed
   install first (`launcher.go:121`); Linux never does. Error: `chrome: no Chrome
   binary found in PATH on Linux`. Workaround: symlink/wrapper in PATH.
2. **D2 — root/container users cannot start Chrome, and the real error is swallowed.**
   Chrome refuses `--no-sandbox`-less launch as root; sightmap passes no such flag,
   offers no `--chrome-arg` passthrough, and hides Chrome's stderr. All you get is
   `chrome did not become ready: timed out waiting for CDP at 127.0.0.1:7892`. The
   same opaque timeout also appears when a stale `SingletonLock` survives a hard
   container stop. Coding agents overwhelmingly run in exactly this environment; this
   is the first wall every one of them will hit.

After those two workarounds, everything in the quickstart worked as documented.

---

## Phase 3 — Generate

Authoring was done by following `sightmap-authoring/SKILL.md` literally (probe before
YAML, snapshot → coverage → iterate, `data-component` source hook as last resort).

Key raw outputs (abridged; full logs in the session, snapshots not committed per brief):

```
$ sightmap snapshot --coverage        # before any YAML
2525 interactive · 0 direct T1 (0%) · 0 scoped T2 (0%) · 2525 orphaned T3 ✗

$ sightmap snapshot --coverage        # after ~12 min of iteration
2525 interactive · 752 direct T1 (30%) · 1773 scoped T2 (70%) · 0 orphaned T3 ✓
```

The final corpus (committed under `.sightmap/`): 6 views (SignInUp, RecordIndexPage,
TasksPage, RecordShowPage, SettingsPage-stub), 4 global components, ~15 components
total, 3 request entries, 10 memory notes. `report` output at the end:

```
 View             Route          Snaps  T1    T2   T3
 TasksPage        /objects/*         1  32%   68%   0 ✓      <- route column is wrong, see D10
 RecordIndexPage  /objects/*         1  32%   68%   0 ✓
 RecordShowPage   /object/*/*        1   7%   92%   4 ✗
```

**What the tooling got right.** The coverage loop is genuinely effective: the
unlabeled-cluster hints were correct on every iteration; `sel-probe`'s offline/live
cross-check caught nothing false; validate/lint feedback was immediate; the corpus
hot-reloaded on every edit; `discover` correctly mapped 12 route patterns and flagged
`/page/*` as unseen. Most-specific-wins routing behaved exactly per spec.

**What the app did to the format.** Twenty is a hostile (i.e. realistic) target:
- No semantic landmarks at all — zero `nav`/`main`/`table` elements, ~100 `data-testid`s
  in a huge app, everything else Linaria-hashed classes.
- The Filter/Sort/Options buttons are literally `<button class="s1h5yce0">Filter</button>`.
  **No stable CSS selector can exist for them.** Components match by CSS only — there is
  no text= or role= escape hatch — so reaching 0 orphans required editing app source
  (`data-component="TopBar"`, exactly the fix the quickstart prescribes). That is a
  format-level limit worth being honest about in the docs: on hashed-class apps you
  cannot finish a corpus without patching the app.
- Table row testids are `row-id-<record-uuid>` — the suggester emits 20+ one-per-record
  selectors as noise; the useful form (`[data-testid^="row-id-"]`) I had to write myself.
- Record-page widget containers use raw page-layout UUIDs as testids, which are
  **per-workspace database rows**: a corpus authored on one workspace has dead widget
  selectors on every other deployment. Marked `stability: unstable` in the corpus.
- The settings content pane has no hooks at all; shipped as a `stability: stub` view.
- The record chip renders avatar-letter + name ("GGoogle"), defeating `text` and
  `text_only` extraction (avatar is a text span, not an img); needed a structural
  `div:nth-of-type(2)` sub-selector extract.

**What a human must hand-edit after the loop:** prefix-selector generalizations,
property extractions for every chip-like control, the `source:` links, all memory
notes, stability markers, and any component whose only identity is its text. In this
app that was roughly half the total authoring effort; none of it is automatable with
the current suggester.

**Spec-level insight from authoring:** navigation-quirk knowledge must be attached to
the *component you act on* (component memory surfaces wherever it matches), not the
destination view — view memory only appears once you are already there, which is too
late. The docs never say this; they should.

---

## Phase 4 — Consume (the value-proposition test)

Design: two read-only tasks, three conditions, fresh agent context per run, identical
model, same CLI harness in all conditions so the *only* variable is the corpus:

- **empty** — `.sightmap` with no YAML: plain accessibility tree + numeric-id clicks
  (stand-in for "agent with generic browser tooling"; the Playwright MCP baseline died
  with a container restart, and holding the harness constant is the cleaner control anyway)
- **nomem** — full corpus with every `memory:` block stripped
- **full** — the corpus as committed, memory included

Task A (navigational): find company "Cisco", report Domain Name + Account Owner.
Task B (semantic/state): open the FULL record page (not the preview panel) for
"Microsoft", report its URL + Address field. Task B is aimed straight at the app's
side-panel quirk, which the full corpus documents in `RecordNameChip` memory.

| Trial | Condition | Tool calls | Tokens | Wall clock | Wrong turns | Correct |
|-------|-----------|-----------|--------|-----------|-------------|---------|
| A1 | empty | **11** | **36.8k** | **100 s** | **1** | yes |
| A2 | nomem | 26 | 46.6k | 242 s | 7 | yes |
| A3b | full | 24 | 49.5k | 275 s | 5 | yes |
| B1 | empty | **7** | **36.4k** | **108 s** | **2** | yes |
| B2 | nomem | 22 | 45.7k | 196 s | 3 | yes |
| B3 | full | 29 | 49.7k | 321 s | 9 | yes |

(A discarded first run of A/full took 51 calls/643 s after the shared viewport drifted
to 400 px mid-experiment — excluded as contaminated, but note the tool never surfaced
"you are in a mobile layout" either.)

**The delta is negative.** On this app, corpus-equipped agents did the same tasks about
half as fast, at ~30% more tokens, with more wrong turns, and no accuracy gain (6/6
everywhere). That is the headline, and it is not noise — the failure modes were
identical and mechanical across all four corpus runs:

1. **Tree elision (D11), the dominant tax.** `snapshot` renders `[RecordTable]` as a
   leaf: the Guide header counts 60 matched `RecordTableRow`s, but the printed tree
   contains none of them, and no row text is greppable. Every corpus agent burned
   4-7 calls discovering that the advertised artifact — the annotated tree — omits
   the exact content it annotated, then fell back to `browser eval` with raw CSS
   selectors, i.e. the thing the corpus exists to prevent.
2. **`snapshot --url` races (D12).** The skill teaches "ALWAYS navigate before
   snapping" with `--url`; but `--url` *re-navigates* and snapshots the loading page
   (0 interactive nodes, hard error). Both full-corpus agents lost 3+ calls to this
   before reading `--help` and dropping the flag. The skill's own advice induces the
   failure.
3. **Search beats semantics.** The winning route in 5 of 6 runs was the app's own
   search box — which the corpus doesn't describe at all (my authoring miss, but also
   evidence that a good in-app search flattens the navigation value of a map).
4. **Memory: worked once, net negative.** B3 is the only run that used memory, and it
   did precisely what the note said (chip → side panel → Open button) — the mechanism
   works, provenance to the note is explicit in the agent's log. But that documented
   path was slower than B1/B2's search route, and my second memory note ("chips render
   twice; click occurrence #1") encoded hover-state trivia that produced an
   `index out of range` wrong turn in B3. Memory transfers the author's model,
   including the author's mistakes, with the authority of documentation.
5. Where the corpus *did* shine: `click 'RecordNameChip[label="Cisco"]'` resolved
   one-shot with zero searching (A2 step 14), and the ambiguity errors listing labeled
   candidates are the best interaction UX in the toolchain. The wins exist; they are
   currently drowned by items 1-2.

Caveats: n=1 per cell, one model, one app, my corpus (first-time author, ~40 min).
Fixing D11+D12 alone could plausibly flip Task A's sign; nothing in this data says the
concept is dead. What the data does say: **on this app today, the artifact does not pay
for itself at consumption time, and the burden of proof is on the tool.**

---

## Phase 5 — Break it

| Case | What I did | Result | Loud/Silent |
|------|-----------|--------|-------------|
| Malformed YAML (offline) | garbage indentation + unclosed bracket | `validate` exit 1, parse error with line number | **Loud** |
| Malformed YAML (live session) | same, while server running | every command errors; `serve-sightmap` logs compile error, keeps serving last good corpus | **Loud** |
| `lint` on unparseable corpus | same file | prints load error but **exits 0** | **Silent-ish (D4)** — CI gating on lint alone passes a broken corpus |
| Missing required fields | component w/o name/selector, view w/o name/route | 5 errors, exit 1, precise messages | **Loud** |
| `version: 2` / missing `version` | spec says MUST reject | **accepted, no error, no warning** | **Silent (D3)** — violates the spec's own conformance section |
| Conflicting entries | two views same route; dup global component | warnings name the winner ("only the first (PeopleA) applies"); runtime `[Conflicts]` lists every contested node | **Loud** |
| Unknown/typo'd fields (`memroy:`) | typo'd memory | warning with file+line, exit 0 | Loud warning — but note a typo'd `memory:` silently deactivates the note at runtime; warning-not-error is a judgment call |
| Circular / unresolved `$ref` | self-ref + missing name | both errors, exit 1, per spec | **Loud** |
| Two sightmaps in scope | parent/.sightmap + child/.sightmap; run from various cwds | strictly cwd-relative; no walk-up; from an unrelated subdir: warning + exit 0 + **empty corpus** | **Weak** — agents cd around monorepos; the corpus silently vanishes |
| Stale sightmap after UI change | renamed `data-component="TopBar"` in source | `ViewBar — 0 matches` warning, orphans reappear, cluster hints suggest the **new** selector | **Loud, actively helpful** — best failure mode in the tool. Caveat: identical warning fires for still-loading pages, so "dead" vs "not rendered yet" is indistinguishable |
| SPA routes absent at load | snapshot during hydration | `∅` + non-zero exit at 0 nodes | **Loud at exactly 0; silent at 23 of 2,525 nodes (D9)** — partial hydration passes as a valid page |
| Auth-gated pages | `clear-storage`, navigate to protected URL | SPA bounced to /welcome; snapshot matched SignInUp view and surfaced the credentials memory; "[Snapshot check] view root components all unmatched" fired | **Loud** — but `navigate` did NOT print the documented `(redirected to …)` for this async SPA bounce (D13) |
| Buffer overflow | Vite dev server | 29k network entries dropped, GraphQL calls evicted; overflow is at least announced | Announced, but the collector is unusable on module-serving dev servers (D7) |
| `--help` side effects | `sightmap browser install --help` | **executed the 184 MB install** | **(D5)** |
| Container restart | kill everything mid-session | stale `.session` cleaned by `status` probe; stale Chrome `SingletonLock` reproduces the opaque D2 timeout | Mixed |

---

## Ranked defect list

1. **D11 — Annotated tree elides matched subtrees on large tables.** Guide says 60
   rows matched; tree renders `[RecordTable]` as a leaf; no row is greppable. Breaks
   the product's core artifact at consumption time; single biggest cause of the
   negative Phase 4 delta. (Repro: companies index, 60 rows, `sightmap snapshot`.)
2. **D2 — Chrome-as-root failure is opaque and unfixable via CLI.** No `--no-sandbox`
   path, no arg passthrough, Chrome stderr swallowed → generic CDP timeout. Blocks the
   primary audience (containerized coding agents). Same opaque timeout for stale
   profile locks.
3. **D1 — `browser install` → `browser start` broken on Linux.** `FindChrome` Linux
   branch never checks `~/.sightmap/browsers/`. One-line-ish fix at
   `go/browser/launcher.go:151`.
4. **D3 — Version field unenforced, silently.** `version: 2` / absent version validate
   clean; spec says MUST reject; schema says `const: 1`. `loader.go:43` parses it,
   nobody checks it. Kills the spec's forward-compat story.
5. **D12 — `snapshot --url` re-navigates and races; the skill recommends the exact
   pattern that fails.** Either make `--url` assert-current-URL (navigate only if
   different, then wait for render), or fix the skill text.
6. **D6 — `requests:` are write-only.** Names, payload docs, headers, memory never
   surface in `network list`/`get`. A third of the spec is dead weight in the only
   implementation.
7. **D4 — `lint` exits 0 on unparseable corpus.** CI trap.
8. **D9 — Blank-page guard only fires at exactly 0 interactive nodes.** A 23-node
   partial hydration of a 2,525-node page passes silently; my trial-reset probes hit
   this repeatedly, and so will every agent on a slow SPA. Consider warning when
   coverage drops >X% versus the view's capture set.
9. **D7 — Collector buffers useless on dev servers.** 1000 entries evicted instantly
   by Vite module traffic; needs type/host filtering at capture time, not query time.
10. **D5 — `browser install --help` executes the install.** Also `sel-probe --all`
    means "probe every view URL", while `suggest`'s `--exclude-known`/`gap`'s
    `--visible` are documented as "always on" flags — flag conventions drift
    command-to-command.
11. **D10 — `report` misattributes.** TasksPage row shows another view's route and an
    inherited capture set; uncaptured views print their URL in the Route column.
12. **D13 — `navigate` misses async SPA redirects** despite docs/skill promising
    `(redirected to FINAL)` for exactly this case (auth guard bounce).
13. **D8 — `sel-probe` `matches:` prints the display cap** (10) next to an offline
    count of 780 with no "showing 10 of N".
14. Cosmetic: `search` exits 0/"no matches" for view names (documented but surprising);
    `wait-for` flag is `--timeout-ms` where agents guess `--timeout`/`--wait` (both
    trial agents lost a call to this).

## What I'd tell the builder to do next, in order

1. Fix D11 and D12, then rerun the Phase 4 protocol — those two account for most of
   the negative delta, and the experiment is cheap to repeat (prompts are in this
   repo's history).
2. Make `browser start` survive containers: bundle-aware `FindChrome` on Linux,
   `--chrome-arg` passthrough or auto `--no-sandbox`-as-root, and always surface
   Chrome's stderr on launch failure.
3. Enforce `version` (it is one `if` statement away from spec conformance) and make
   `lint` fail on load errors.
4. Either implement request-entry surfacing in `network get` or cut `requests:` from
   the pitch until it does something.
5. Add a text/role fallback story for components — hashed-class apps (Linaria,
   CSS-modules, Tailwind-with-hashing) are the norm, and today the honest answer to
   "map this button" is "patch the app source", which the quickstart admits only
   obliquely.
6. Document the memory-placement rule this eval derived the hard way: quirks about
   *reaching* a view belong on the component you click, not on the destination view.
7. Teach the authoring skill that in-app search is a first-class navigation surface —
   map it, because agents will find it anyway and it will beat your row selectors.

## Reproduction notes

- Corpus: `.sightmap/` in this branch (snapshots and `.session` git-ignored per brief;
  no raw snapshot JSON committed).
- App-source change required to reach 0 orphans: one attribute in
  `packages/twenty-front/src/modules/ui/layout/top-bar/components/TopBar.tsx`.
- Chrome workaround used throughout: `/usr/local/bin/chromium` wrapper exec'ing the
  sightmap-installed Chrome for Testing with `--no-sandbox`.
- Phase 4 prompts and per-trial action logs are preserved in the session transcript;
  trial metrics above are from the harness's own usage accounting.
