---
name: skyrimnet-save-explorer
description: Build a self-contained, offline HTML "save explorer" web page from a Skyrim SkyrimNet/IntelEngine AI-mod playthrough — an interactive illuminated-journal visualization of one character's run (event timeline, a memory "constellation" from embedding projections, kill ledger, diary, OmniSight field notes, faction relations, provocation chronicle, off-screen gossip, and any war). Use this whenever the user mentions SkyrimNet, IntelEngine, a Skyrim "save explorer"/visualizer, their `SkyrimNet-*.db` or `IntelEngine-*.db` databases, memory embeddings, OmniSight screenshots/captures, faction relations, or wants to turn their Skyrim AI-mod save data or logs into a web page or field journal — even if they don't explicitly say "save explorer."
---

# SkyrimNet Save Explorer

Build the user a **SkyrimNet Save Explorer**: one self-contained, offline HTML file that turns their
Skyrim AI-mod playthrough (the **SkyrimNet** and **IntelEngine** mods) into an elegant, interactive
"save explorer" — an illuminated field journal for a single character's run.

The finished page is an **app shell**: a sidebar on the left, one view at a time in the content column,
addressed by hash routes. It lands on an **Overview** — the masthead, the headline stats, and panels
summarising each section — and the sidebar groups the rest:

- **SkyrimNet** — the character's lived experience: an event timeline, an interactive "constellation" of
  memories, the memories themselves, a kill ledger, a diary, and OmniSight field notes (a distilled
  "portrait of the land" + matched capture images), plus any other notable story beats found in the logs.
- **IntelEngine** — the political layer: faction relationships, a chronicle of provocations, off-screen
  gossip, and — **only if the save actually has one** — an ongoing war and its battles.

Each view owns its own search, filters and paging; a view with no data is left out of the sidebar
entirely. Everything — name, in-game day count, every count and label — is derived from the user's
uploaded data.

> **Not a long scroll.** A real save runs to hundreds of memories and captures. Do not render every table
> onto one page, and do not copy the shell of an older example: the two-tab bar, the floating
> roman-numeral TOC, and the collapsible sections are gone. `references/page-build.md` explains what
> replaced them and why.

## Non-negotiable principles

These hold across every phase; internalize them before starting.

1. **Collect files first — do not build anything until the user has uploaded their data.** The whole
   page is derived from real files; there is nothing to build from until they arrive. Follow the Phase 1
   collection protocol one file at a time.
2. **Derive everything from the user's data; hardcode nothing from any example.** The character's name,
   day count, every stat, label, and count must come from the uploaded files — never from a sample in
   these instructions. If a value can't be derived, say so rather than inventing it.
3. **One self-contained file that opens offline by double-click.** No build step, no frameworks, no
   external JS/CSS libraries. The only allowed external references are a Google Fonts `<link>` and
   fonts/images embedded as base64 data URIs.
4. **Never modify the user's databases — read-only SELECTs only.** Inspect the SQLite files with Python's
   `sqlite3` module. If a read-only URI handle fails to open, copy the DB to a writable temp location and
   open the copy, still issuing SELECTs only.

## Key moments where you must ask the user

The workflow is collaborative and several decisions are the user's to make. These are easy to skip —
don't:

- **Collect one file at a time.** After each upload, inspect it and report what you found (row counts,
  the character's name, the date range, anything missing) *before* asking for the next file.
- **Offer a new view when something notable turns up.** If a file reveals a major world event, quest
  arc, milestone, or unusual populated table the standard layout doesn't cover, tell the user and ask
  whether they'd like a dedicated view for it. Only add views they approve.
- **Total Spend is opt-in and user-supplied.** Before finalizing the header, ask whether they want a
  "Total Spend" stat (real-world money spent running the playthrough — API/LLM calls, not in-game gold).
  If yes, ask how much in real dollars and use their figure verbatim. If no, skip it entirely.
- **Ask which theme before styling anything** — the Illuminated Manuscript preset, or the user's own
  described theme (invoke a design skill if one is available).
- **Detect and announce whether the save has a war.** After the IntelEngine DB arrives, tell the user
  plainly whether there's a war. If not, the war and battle views are omitted entirely — no empty panels,
  and no sidebar entries for them.
- **Ask before embedding OmniSight screenshots.** If the user wants image embeds, ask them to upload the
  screenshots as **one archive** (not individual images), then place them in `omnisight-images/`.

## The four phases

Work through these in order. Each of the first three has a detailed spec in `references/`; **read that
file when you reach the phase** rather than all at once.

### Phase 1 — Collect the files → `references/data-sources.md`

Ask for files one at a time, in this order, inspecting and reporting after each:

1. **SkyrimNet database** (`SkyrimNet-<numbers>.db`) — source for most of the SkyrimNet tab.
2. **IntelEngine database** (`IntelEngine-<numbers>.db`) — source for the IntelEngine tab; detect the war.
3. **SkyrimNet logs** — `openrouter_output.log` is recommended and is the only source for gossip.
4. **External assets (optional)** — fonts, images, OmniSight captures the user supplies.

`references/data-sources.md` has the full table schemas, the embedding-decoding gotcha, the log-parsing
protocol, and the asset-wiring guidance.

### Phase 2 — Transform the data → `references/payloads.md`

Build the JSON payloads to embed in the page: `sndata` (SkyrimNet), `inteldata` (IntelEngine), and
`gossipdata`. This is where the memory-embedding projection lives — the PCA/UMAP layout and the
nearest-neighbor edges are computed **offline in Python**; the browser only ever receives coordinates,
edge indices, and similarity weights, never raw vectors. `references/payloads.md` has every payload
field and the full 5-step projection recipe.

### Phase 3 — Build the page → `references/page-build.md`

Produce the single self-contained `.html`. `references/page-build.md` covers the app-shell structure
(sidebar, hash-routed views, the Overview landing view, the mobile drawer), the rules every view obeys
(paging, lazy images, lightbox, searchable filters, escaping model text, showing data gaps), the theme
fork and the Illuminated Manuscript preset, and the exact spec for every view.

### Phase 4 — Verify before handing over

Do these checks yourself before delivering:

- Confirm it opens offline; **render it** (a headless browser is ideal) and **visit every route** —
  check each view populates, with no empty panels, no `undefined`, no `NaN`, and no console errors (a
  blocked Google Fonts request offline is expected and fine). Visiting one view proves nothing about the
  others; if you script the sweep, confirm the tool is really changing route — a mangled hash, or a
  navigation to the URL you are already on, will silently re-render the current view and pass every check
  while testing nothing.
- **Then look at every view — and look at it scrolled.** Screenshot each route and actually read the
  picture. Every other check in this list is negative — absence of errors, absence of overflow — and
  **layout breakage is none of those things**, so it sails straight through them. A war panel once shipped
  whose belligerent columns collided with the sidebar's CSS class, inherited `height:100vh`, and rendered
  ~3.5× too tall, the "Versus" mark stranded in 800px of dead space and the sidebar's border drawn down the
  middle of the panel — with no console error, no `undefined`, no horizontal overflow, every interaction
  working. Only looking finds that.
- **Scroll before you believe it.** A screenshot at the top of the page is not evidence the page works: at
  `scrollY = 0` a broken `position:sticky` element is pixel-identical to a working one. This exact hole let
  a broken sidebar ship *after* the rule above was already being followed — the page was screenshotted, it
  looked perfect, and the sidebar scrolled away and got cut in half the moment a reader moved. So: on a
  long view, **scroll down and confirm the sidebar and the view header are still pinned at `top: 0` and the
  sidebar's footer is still on screen**; on narrow screens confirm the same for the sticky top bar. Assert
  it (`getBoundingClientRect().top === 0` while `scrollY > 0`), don't eyeball it.
- Two smells worth measuring as well as eyeballing: **a container far taller than the content it holds**
  (something is inheriting a height it shouldn't), and **a view whose text length is implausibly short**
  for the records it should be showing.
- Confirm every interaction works: sidebar routing and the browser **back button**; the UMAP/PCA toggle;
  the constellation's actor chips dimming stars; **memory search** (hits highlighted, live count correct,
  and markup in the query rendered as text rather than injected); type/mind/sort filters; **paging**;
  the OmniSight show-all; the **capture lightbox** (opens, Left/Right move, Escape closes, focus returns);
  and the whisper filters.
- Confirm every capture image resolves — count the files in `omnisight-images/` against the payload and
  check the lightbox shows a decoded image, not a broken reference.
- Confirm the layout is clean: **no horizontal overflow** at desktop and mobile widths; on narrow screens
  the drawer opens, the scrim and Escape close it, and the sidebar never traps the reader.
- Confirm the character name, day count, "The Adventures of …" title, and every stat came from the
  user's data — not any example — and the OmniSight summary is grounded in their actual captures.
- Confirm **no record was silently dropped**: undated memories appear in their own group, unplaceable
  captures have a home, and any notice states counts you actually computed.
- **Tell the user plainly what you had to fall back on and what the data couldn't support** — e.g. "UMAP
  not installed, used t-SNE"; "no log provided, gossip omitted"; "no OmniSight images provided, text-only
  capture cards"; "6 memories carry a real-world timestamp instead of the in-game clock and 1 has none,
  so 7 are grouped as Undated". Report the real numbers from their save, not these examples. Papering over
  a gap is worse than the gap.
- Then deliver the finished `.html` file.
