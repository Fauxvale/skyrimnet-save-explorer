Build me a **SkyrimNet Save Explorer**: one self-contained, offline HTML file that turns my Skyrim
AI-mod playthrough (the **SkyrimNet** and **IntelEngine** mods) into an elegant, interactive "save
explorer" — an illuminated field journal for a single character's run.

The page is an **app shell**: a sidebar on the left, one view at a time in the content column, addressed
by hash routes. It lands on an **Overview** — the title card, the headline stats, and panels summarising
each section — and the sidebar groups the rest:

- **SkyrimNet** — the character's lived experience: event timeline, an interactive "constellation" of
  memories, the memories themselves, a kill ledger, a diary, and OmniSight field notes (a distilled
  "portrait of the land" + matched capture images), plus any other notable story beats found in the logs.
- **IntelEngine** — the political layer: faction relationships, a chronicle of provocations, off-screen
  gossip, and — **only if my save actually has one** — an ongoing war and its battles.

Each view owns its own search, filters and paging; a view with no data is left out of the sidebar
entirely. Everything — name, in-game day count, every count and label — must be **derived from my uploaded
data**. Nothing is hardcoded from any example.

**It must not be one long scroll.** My save runs to hundreds of memories and captures. Don't render every
table onto one page, and don't copy the shell of an older example — the two-tab bar, the floating
roman-numeral TOC and the collapsible sections are gone. Phase 3 explains what replaced them and why.

**Do not build anything yet.** First collect my files, one at a time, per the protocol below.

---

## Phase 1 — Collect the files (one at a time)

Ask for **one file at a time**, in order. After each upload, **inspect it and report what you found**
(row counts, character name, date range, anything missing) before asking for the next. Wait for each
upload. While inspecting, also **flag anything notable the standard layout doesn't cover** — a major
world event, quest arc, milestone, unusual populated table — and ask whether I'd like a dedicated
section for it (see "Beyond the standard layout").

The databases are **SQLite**. Inspect with Python's `sqlite3` module (the CLI may be absent). **Read-only
SELECTs only — never modify my files.** If a read-only URI handle (`file:...?mode=ro`) fails to open
(e.g. read-only mount), copy the DB to a writable temp location and open the copy, still SELECT-only.

### Step 1 — SkyrimNet database

Ask for my **SkyrimNet database** (`SkyrimNet-<numbers>.db`). Read its schema; confirm these tables exist
and their row counts. This DB feeds most of the **SkyrimNet tab**:

- **`memories`** — first-person memories. Columns: `content`, `actor_uuid`, `location`, `game_time`
  (float clock), `importance_score` (~0.5–0.85), `emotion`, `memory_type` (`EXPERIENCE`, `RELATIONSHIP`,
  `KNOWLEDGE`, `TRAUMA`, `SKILL`), `tags` (JSON array), `related_event_ids` (JSON array).
- **`memory_embeddings`** — one row per memory: `memory_id`, `embedding` (raw vector bytes), `dimension`
  (e.g. 384). Powers the Constellation. **Decoding gotcha:** the value is served as TEXT not BLOB, so set
  `con.text_factory = bytes` before reading (otherwise sqlite3 tries to UTF-8-decode and throws or
  reports a wrong `LENGTH()`). Even then `LENGTH()` may be mangled — `np.frombuffer(blob,
  dtype=np.float32)` is authoritative, yielding exactly `dimension` little-endian float32s
  (`dimension × 4` bytes; 384 → 1536), no header/compression. See the projection recipe in Phase 2.
- **`events`** — full event log (often 1000+ rows): `event_type`, `event_data` (JSON), `location`,
  `game_time`, and **`game_time_str`** (a human date like `"7:45 AM, Middas, 20th of Last Seed, 4E 201"`).
  Drives the day timeline and (via `death` events) the kill ledger. Also read `originating_actor_UUID` /
  `target_actor_UUID` and `playtime` where present.
- **`uuid_mappings`** — maps numeric `uuid` values to `actor_name`. Use to resolve every actor UUID.
  **The player's name comes from here** (the actor flagged as player, e.g. bio template `player_special`).
- **`omnisight_screenshots`** — location/scene notes: `subject_name`, `description` (prose),
  `location_name` / `location_cell`, `capture_time_game` (date). Screenshot pixels are **not** in the DB —
  use the text + metadata, plus matching files in `omnisight-images/` for the card images.
- **`diary_entries`** — long-form first-person `content` with `emotion` and `entry_date`.

Tell me the character's name and in-game date span, then proceed.

### Step 2 — IntelEngine database

Ask for my **IntelEngine database** (`IntelEngine-<numbers>.db`), source for the **IntelEngine tab**.
Confirm:

- **`faction_relations`** — `faction_a`, `faction_b`, `relation_score` (−100…+100), `war_active`,
  `trade_active`.
- **`faction_wars`** — `faction_a`, `faction_b`, `battles_fought`, `faction_a_morale`, `faction_b_morale`,
  `faction_a_strength`, `faction_b_strength`, `victor`, `start_time`. **May be empty** — many saves never
  trigger a war.
- **`war_battles`** — `location_name`, `attacker`, `defender`, `result`, `attacker_losses`,
  `defender_losses`, `narrative`. Also empty when no war.
- **`faction_events`** — provocations: `faction_a`, `faction_b`, `event_type` (`war_declaration`,
  `border_skirmish`, `espionage`, `assassination_attempt`, `battle_result`, `tariff`, `trade_dispute`,
  `insult`), `description`, `relation_delta` (signed int), `game_time`.

If I also have a small **`political_state.json`**, use it for **faction identity** — display names,
leaders, holds — for readable labels. It mirrors current war/relations state too, but the IntelEngine DB
is authoritative for the numbers.

Then **tell me whether this save has a war** — any rows in `faction_wars` / `war_battles`, or a
`war_declaration` in `faction_events`. If none, the war/battle sections are simply omitted; the rest of
the tab still builds.

### Step 3 — SkyrimNet logs

Ask for my **SkyrimNet logs**. Explain what each offers and take only what you need:

- **`openrouter_output.log`** (~1 MB — *recommended*) — the model's structured outputs. **Only** source
  for the gossip section; good for diary/OmniSight text. Keep OmniSight images from local files.
- **`conversation_log.log`** (~78 KB) — raw dialogue transcript (corroboration only).
- **`openrouter_input.log`** and dated siblings (~4–13 MB) — mostly *prompts*; large, redundant with the
  DBs. Usually **skip**; may be too big to upload.

These logs are **not** JSON-lines: a header line followed by a body. Split on the header pattern:

```
^\[.*\] \[.*\] (Generate request|Generate response|GenerateStreaming complete response) \[req_.*\]:
```

then JSON-parse the block that follows (guard for plain-text and ```json-fenced bodies; a body may be one
object or an array). **Discard** the startup health-check pings reading `"Hello! Yes, I'm working"`.
Expect many blocks to be un-parseable echoes/fragments — fine; keep what parses.

Two object families matter:

1. **Gossip** — objects with `type:"npc_gossip"`:

```json
{ "type": "npc_gossip", "npc": "Arivanya", "npc2": "Ulundil",
  "narration": "leaned close and murmured about the Stormcloak uproar...",
  "gossip": "Ulfric called Thalmor justiciars parasites on Nord soil — loudly, in the square" }
```

   `npc`=speaker, `npc2`=listener, `narration`=scene, `gossip`=rumor. (Some are `type:"npc_interaction"`
   with `fact1`/`fact2` instead — you may include those as memories exchanged.)

**If I don't provide logs, don't invent gossip** — hide or annotate that section and tell me it was
skipped for lack of a log.

### Step 4 — external assets (optional)

I may hand you **external assets** — fonts (`.ttf`/`.otf`/`.woff2`), images/textures (borders, panels,
icons, backgrounds), or OmniSight capture images. **When I supply an asset, use it directly** rather than
recreating it: inspect it and report what you found (dimensions, format, likely use). Two ways to wire it
in:

- **Embed (default, single self-contained file):** inline as a **base64 data URI** — fonts via
  `@font-face`, images via `url(data:...)` / `border-image` / `<img src="data:...">`.
- **Reference directly:** keep as a real file, point the page at it by **relative path**, and deliver
  those files **alongside** the `.html` in an `assets/` folder so it still opens offline (just not as a
  lone file).

Keep a CSS/SVG fallback for anything I *don't* supply, and tell me plainly which assets you used (and
how) and which you fell back on. Use **only** assets I provide through chat — never fetch, download, or
invent. For **my own theme**, assets are optional: ask only when they fit what I described. For OmniSight
image embeds, ask first whether I want to upload the screenshots (as **one archive**, not individual
images); if yes, place them in `omnisight-images/` and wire them into the cards.

---

## Phase 2 — Transform the data

Build the JSON payloads to embed. Derive everything from my files.

### SkyrimNet payload (`sndata`)

- **`player`** — the player's name (from `uuid_mappings`).
- **`stats`** — header numbers: total events, total memories, distinct minds (actors with memories),
  OmniSight captures, in-game day count, and a one-line subtitle (name + date span, plus playtime hours
  if derivable from `playtime`).
  **Total Spend is opt-in and user-supplied:** the header can include a "Total Spend" stat — the
  real-world money I spent running this playthrough (API/LLM calls), **not** in-game gold. It can't be
  derived. Before finalizing the header, **ask whether I want a Total Spend stat**. If yes, **ask how much
  (in real dollars) I've spent** and use my figure **verbatim** (formatted as I type it). If no, skip it.
- **`events_by_day`** — a map keyed `"<in-game day label>|<event_type>" -> count`. Parse the day/date from
  `events.game_time_str` (e.g. `"20th of Last Seed"`), count events per `(day, event_type)`, and emit an
  ordered list of day labels. Bucket `event_type`s into five categories:
  - **Dialogue** (gold): `dialogue`, `dialogue_background`, `dialogue_npc`, `dialogue_player`,
    `dialogue_player_stt`, `dialogue_player_text`, `gamemaster_dialogue`
  - **Death & combat** (blood red): `death`
  - **Thoughts & narration** (steel blue): `player_thoughts`, `direct_narration`, `diary_entry_created`
  - **Trade, rest, world** (moss green): `trade_start`, `trade_complete`, `sleep_start`, `sleep_stop`,
    `book_read`, `furniture_used`, `item_given`
  - **Followers & tasks** (grey): anything else
- **`memories`** — array; per memory: `content`, `actor` (resolved via `uuid_mappings`), `emotion`,
  `location`, `imp` (= `importance_score`), `type` (= `memory_type`), `day` (in-game day label from
  `game_time`/related event, **or `null`** — see "Timestamp hygiene"), `gt` (raw `game_time`, for
  sorting), and 2-D coords `x, y` (PCA) and `ux, uy` (UMAP) — see the projection recipe.
- **`edges`** — array of `[i, j, similarity]`: for each memory, its **2 nearest neighbors** by cosine
  similarity in the full embedding space, with the similarity.
- **`leaderboard`** — kills per killer, ranked by total. Each:
  `{ killer, total, cats: { <category>: count }, victims: [[victim_name, count], …] }`. Parse `death`
  events (killer/victim from `event_data`, falling back to `originating_actor_UUID` /
  `target_actor_UUID`), resolve names, and bucket each victim into **People, Undead, Wildlife, Monsters,
  Dragon** for the stacked bar.
- **`diary`** — `{ actor, loc, content, emotion }` (array if several). Keep paragraph breaks and
  `*emphasis*` markers intact.
- **`screenshots`** — OmniSight captures as `{ name, loc, ts, desc }` from `omnisight_screenshots`
  (`ts`=capture date/time; `loc`=readable location/cell). Match each to its local image in
  `omnisight-images/` and embed it in the card.
- **`omni_summary`** — **a synthesized "portrait of the land."** Read **all** OmniSight descriptions and
  distil them into one cohesive prose description of the whole landscape: `{ title, paragraphs:[…2-3…],
  foot }`. Ground it strictly in what the captures describe — real named places, terrain, structures,
  wildlife, weather, mood (do a quick frequency pass for recurring places/atmosphere) — and **never
  invent** locations or events. `title` ≈ `"A Portrait of the Land"`; `foot` a derived line like
  `"Distilled from all <N> OmniSight captures — <first day> to <last day>, 4E 201."`

**Actor colours:** rank actors by memory count; the **top ~5 get distinct colours**, everyone else grey.
Reuse this mapping everywhere an actor is coloured (constellation dots, memory rows, leaderboard names).

### Timestamp hygiene — check this, don't assume it

`memories.game_time` is supposed to be the in-game clock, and `gt` exists so the page can sort
chronologically. **In real saves the column mixes units**, and passing it through unexamined makes the
page sort wrongly and silently.

Observed in a real 255-memory save: 249 rows carried an in-game clock in the range `0`–`1,281,339`, while
**6 rows carried a Unix epoch** — `1784014205`, which decodes to a real-world date (the day the page was
built). Wall-clock had leaked into a game-time field. Those 6 rows plus 1 row with `game_time == 0` were
**exactly** the rows for which no day label could be resolved: `day is None` ⟺ `gt` unusable.

The important part: **`gt` isn't broken, it's contaminated.** Sorting the 249 well-formed rows by `gt`
reproduced the day order with zero inversions. Quarantine the outliers; don't abandon `gt` sorting or
invent a different key.

So, when building `memories`:

1. **Classify each `game_time`** against the clock the rest of the save uses. Compute the distribution
   first — don't hardcode a threshold from this document. A value that decodes as a plausible recent
   real-world Unix timestamp (roughly `> 1e9`, i.e. year 2001+) is wall-clock, not game time; `0`, `NULL`
   and non-finite values are missing; the rest is the in-game clock. The clusters sit orders of magnitude
   apart, so the split is unambiguous — but **verify it in my data** and tell me what you found.
2. **Emit `day: null`** for any memory whose timestamp is unusable. Don't guess a day, don't fall back to
   the first or last day, and don't drop the memory — its `content` is still real.
3. **Still emit `gt`** for those rows so nothing is lost, and let the page quarantine them when sorting.
   Sorting must never place a contaminated row among dated ones.
4. **Report it to me** in Phase 4: how many memories are undated, and why.

Apply the same scepticism to any other time field. If a join depends on two time values agreeing, **check
that they do** on my actual data before building on it, and tell me when they don't.

**Computing `x,y` / `ux,uy` and edges** — do it all offline in Python; the browser only ever receives
coordinates, edge indices, and similarity weights, never raw vectors:

1. **Decode.** Open with `con.text_factory = bytes`. Join `memory_embeddings` to `memories` on
   `memory_id`; decode each with `np.frombuffer(blob, dtype=np.float32)`. Expect `len(vec) == dimension`
   (384). `dimension × 2` bytes → float16; smaller/irregular → compressed store (but standard DBs are
   plain float32). Stack into an `N × dimension` matrix **in memories order**, so row *i* is memory *i*
   everywhere downstream (tooltips, filters, edges all index this shared array).
2. **PCA → `x,y`.** Mean-center; take the top-2 principal components via SVD (top two singular vectors
   scaled by their singular values). Report the fraction of variance captured (sum of top-2 squared
   singular values / total) in the caption — usually ~20%, an honest "blurry linear shadow."
3. **UMAP → `ux,uy`.** Run UMAP on the same matrix with **`metric='cosine'`**, `n_neighbors=10`,
   `min_dist=0.15`, fixed `random_state` (reproducible). Cosine is deliberate — it's the metric the mod's
   recall uses. If `umap-learn` isn't installed, try to `pip install` it; else fall back to cosine t-SNE
   (sklearn); else reuse the PCA coords. **Tell me which you used.** Normalise both projections into a
   comparable 0–1 range.
4. **Edges (in the full space, not the projection).** L2-normalize every row, compute the full cosine
   matrix (`Xn @ Xn.T`), and per memory keep its top-2 neighbors above a similarity floor (~0.75). Dedupe
   symmetric pairs into an undirected list; ship each as `[index_a, index_b, similarity]`, similarity
   driving line opacity. Because edges are truth from the 384-d space while positions are projected, long
   edges in PCA are exactly the neighbor pairs the linear projection tore apart — a built-in diagnostic.
5. **Serialize.** Round coords to ~4 decimals, embed as JSON. In the
   `<script type="application/json">` tag, escape `</` as `<\/` so no text can close the script block.

The `text_factory = bytes` trick applies to any SkyrimNet DB you inspect with Python.

### IntelEngine payload (`inteldata`)

- **`relations`** — from `faction_relations`: each pair with signed `relation_score`, whether at war, and
  readable display names (via `political_state.json` if present, else de-camelCased).
- **`factions`** — from `political_state.json` if present: `{ name, leaders, hold }` per faction, for
  labels, hover detail, faction colours, whisper classification.
- **`war`** — from `faction_wars`, **only if a war exists**: belligerents, `battles_fought`, start,
  both sides' morale & strength, victor status, each side's leaders, and who declared it (from the
  `war_declaration` `faction_event`). Omit this key entirely when there's no war.
- **`battles`** — from `war_battles`: location, attacker/defender, result, losses each side, narrative.
  Empty/omitted when no war.
- **`events`** — from `faction_events`: the provocation chronicle — faction pair, `event_type`,
  `description`, signed `relation_delta`, ordered by `game_time`. You may carry the raw `game_time`
  through as a `day`, but note: **IntelEngine's clock is not SkyrimNet's clock.** `faction_events.game_time`
  is a small numeric day counter (e.g. `3.3` … `15.4`), on a completely different scale from the in-game
  calendar labels (`"20th of Last Seed"`) that `events_by_day` and `memories[].day` use, with **no reliable
  mapping between them**. Don't convert one to the other, don't join IntelEngine events onto the SkyrimNet
  timeline, and don't present a numeric day as a calendar date.

### Gossip payload (`gossipdata`)

Array of `{ speaker, listener, rumor, scene }` from successful `npc_gossip` records in the output log
(`speaker`=`npc`, `listener`=`npc2`, `scene`=`narration`, `rumor`=`gossip`). Keep only **distinct**
exchanges (dedupe). Empty if no log. Displayed gossip must come **only** from these saved outputs — the
dialogue/input logs are corroborating provenance, not a source of new rumors.

---

## Phase 3 — Build the page

Produce **one self-contained `.html` file**. No build step, no frameworks, no external JS/CSS libraries.
The only external references allowed are a Google Fonts `<link>` and fonts/images embedded as base64 data
URIs. Embed payloads as `<script type="application/json">` blocks; render with **vanilla JS**. It must
open by double-clicking offline. Honor `prefers-reduced-motion`, keep visible keyboard focus, be
responsive to mobile.

### Structure (every theme)

**The page is an app shell: a sidebar and one view at a time.** This matters. My save holds hundreds of
records — a real run is ~255 memories, ~163 captures, ~54 gossip records. Printing every table onto one
scroll, expanded, means I scroll past hundreds of rows to reach anything, and the page ends up organised
**by database table rather than by anything I care about**. Don't do that. Give each section its own
screen and let me choose one.

- **Sidebar (desktop, ≳900px)** — pinned full height on the left, with its own scroll.
  - **Brand block** at the top: a small italic serif kicker **"The Adventures of"**, my character's name
    in the uppercase display face beneath it, then a mono line with the in-game date span, closed by a
    thin rule.
  - **Nav entries** in groups: an ungrouped **Overview** first, then **SkyrimNet**, then **IntelEngine**
    (each group labelled with a small mono all-caps header). Every entry shows its **record count** on the
    right — the counts are my map of where the substance is.
  - **Active entry** takes an accent left rule, an accent label, and an accent count.
  - A small footer names the source database files.
- **Views and routing** — each section is a view on a **hash route** (`#/memories`, `#/captures`). Only
  the active view is in the DOM; the **back button must work**; no hash lands on Overview.
- **View header** — sticky atop the content column: a small accent kicker naming the half of the journal
  ("A SkyrimNet Field Journal" / "An IntelEngine Dispatch"), the view title, an italic caption. The bar
  spans the full width beside the sidebar — its border and blur are meant to reach the edge — but its text
  aligns to the content column, not to the bar. Overview is the exception — it carries its own masthead.
- **Content column** — cap the column (~`1180px` reads well) and **centre it** in the space beside the
  sidebar. A `max-width` alone is not enough: without `margin-inline:auto` the column stays pinned left and
  the surplus piles up as dead space on the right, which is glaring on my wide monitor and easy to miss if
  you only check at laptop width. The view header must resolve to the **same** cap and inset, or its title
  sits left of the content beneath it on every view. Put the cap on a wrapper *inside* the header, not on
  the header itself, so the bar stays full-bleed while its text lines up; with `box-sizing:border-box` that
  wrapper has to repeat the column's horizontal padding for the two left edges to meet. Give the
  narrow-screen rules the same treatment — moving the header's padding onto the wrapper means the mobile
  override moves with it.
- **Overview is the landing view.** It answers "what happened in this run?" without scrolling far:
  - a **masthead** — the monumental title card: italic "The Adventures of", my name set **large**
    (uppercase display face), the italic subtitle. Only here; on every view it would cost a screenful each
    time;
  - a **stat row** from `stats` (including Total Spend if I opted in);
  - any **data notices** (see "Be honest about gaps");
  - a grid of **panels** summarising each section and linking into it — activity-by-day bars, busiest
    minds, heaviest memories, a few captures, the war if there is one.
- **Conditional views** — a view with no data is **left out of the nav entirely**. No empty panels, no
  "no war found" placeholders. Nothing renumbers, because nothing is numbered.
- **Narrow screens (≲900px)** — the sidebar becomes an **off-canvas drawer**: a sticky top bar with the
  current view name and a hamburger; tapping slides the sidebar in over a scrim. Scrim, entry, or Escape
  closes it.
- Confirm **no horizontal overflow** at any width (let long captions wrap), and check the page at a
  **wide** viewport (≳1500px) as well as at laptop width — that is where an uncentred column gives itself
  away.

#### What this replaces, and why

If you've seen an older version of this page — a two-tab bar, a floating roman-numeral TOC that reflowed
into a mobile bar + overlay, collapsible sections on one long scroll — **that's all gone. Don't copy it.**

- **Tabs → sidebar groups.** Two tabs hid half the journal behind a click and still left each tab a long
  scroll. Now every section is one click from everywhere.
- **Floating TOC → the sidebar.** The TOC navigated a scroll that no longer exists. A real column can't
  overlap the content or need re-centering, which the old approach had to work around.
- **Collapsible sections → routing.** Collapsing was a workaround for too much on one page. With one view
  at a time there's nothing to collapse — drop the chevrons and `aria-expanded` toggles.
- **Roman numerals → record counts.** Numerals implied a reading order a routed shell doesn't have. A
  count ("255", "163") tells me something they never did.
- **The "keep-visible" chip row → per-view filters.** That hack existed only because the constellation's
  chips also drove the memory list further down the same scroll. Separate views now: the constellation
  owns its filter, Memories owns its own. Drop the exception and the "the filter above also filters this
  list" note.

### Look & feel — ask me which theme

**Before settling on any colors, fonts, or styling, ask me to pick one of two:**

1. **Preset — Illuminated Manuscript** (below). If picked, apply exactly; ask nothing further about design.
2. **My own theme** — I describe a color scheme / design I'd prefer. If picked, **invoke your design
   skill** (if available) and design to my spec — palette, typography, styling — keeping the same
   structure, sections, and interactions. Show me the resulting palette/type choices before building the
   full page. If no design skill, apply solid design fundamentals. **If I supply my own fonts/images/
   assets** (Phase 1, Step 4), embed and use them directly as base64 data URIs rather than approximating.

#### Theme 1 — Illuminated Manuscript (preset)

- **Fonts:** `Oswald` (uppercase, letter-spaced headings/labels), `Crimson Pro` (serif body),
  `JetBrains Mono` (numbers/metadata), via Google Fonts `<link>`.
- **Palette (`:root`):** `--bg:#14161a; --panel:#1a1d23; --panel2:#20242b; --text:#e8e4da; --dim:#98938a;
  --gold:#c9a45c; --steel:#7da3b8; --blood:#b0584e; --moss:#8ba06b; --violet:#a98cc0; --amber:#cf9a54;`
  plus hairlines like `rgba(232,228,218,.16)`.
- **Category colours:** Dialogue=gold, Death=blood, Thoughts=steel, Trade/world=moss, Followers=dim/grey.
  Reserve **amber** for data-gap notices, so a gap never reads as content.
- **Motifs:** the monumental Overview masthead, "double rule" hairlines, uppercase letter-spaced view
  titles with a trailing hairline, the accent-marked active sidebar entry.

*(Everything below is the structural baseline for whichever theme I pick.)*

### Rules every view obeys

These are the difference between a page I can use and a data dump with a nice font.

- **Any list longer than ~25 records is paged or virtualized.** Never print 255 rows at once. Show a live
  "N of M shown" count.
- **Any collection of images is lazy-loaded** (`loading="lazy"`) and rendered on demand — don't build 163
  `<img>` tags to show 12.
- **Every image opens in a lightbox.** An image I can't enlarge is a thumbnail of nothing.
- **Any filter over a long list is searchable**, not a fixed chip row. Chips suit a handful of categories;
  they can't address 80 actors.
- **Escape every piece of model text before it reaches the DOM** — memory content, rumors and
  descriptions may contain `<`, `&`, or markup. When highlighting a search hit, escape **first**, then
  wrap the match, or a crafted query becomes live markup.
- **Never silently drop a record.**
- **One file means one CSS namespace — name every class for what it is.** This page puts a full-height
  sidebar and a war panel of two belligerents in one stylesheet, and generic names collide. Calling both
  `.side` is a real bug that has shipped here: the belligerent columns inherited the sidebar's
  `height:100vh; display:flex; border-right`, blowing the war panel to ~3.5× its proper height, stranding
  the "Versus" mark in 800px of dead space, and painting the sidebar's border down the middle of the panel
  — with no console error and no horizontal overflow to give it away. Prefer `.sidebar` and `.belligerent`
  over `.side`. The same trap waits in `.stat`/`.page`/`.view`/`.card`/`.row`.
- **Use `overflow-x: clip`, never `overflow-x: hidden`, on `html`/`body`.** These two rules fight, and
  silently: "no horizontal overflow" pushes you toward `overflow-x:hidden`, and `hidden` **breaks every
  `position:sticky` element on the page** — sidebar and view header included. The mechanism: when one
  overflow axis is `hidden`, the other computes from `visible` to **`auto`**, so `<html>`/`<body>` quietly
  become scroll containers. A sticky element sticks to its *nearest scrolling ancestor*, which is then
  `<body>` — and `<body>` never scrolls itself, the document does. Sticky resolves to doing nothing. `clip`
  clips identically but creates no scroll container, so `overflow-y` stays `visible` and sticky works
  against the viewport. This shipped here: the sidebar scrolled off the top, cut in half, while every other
  check passed.

#### Be honest about gaps

Some records can't be placed — a memory whose timestamp is unusable has no day (see "Timestamp hygiene").
**Don't drop them and don't guess.** Give them a visible home (an "Undated" group at the end of the list,
an explicit bucket for unplaceable captures). Where it's worth explaining, render a short **notice**
(amber, distinct from content) on Overview or at the head of the affected list, saying how many records
are affected and why — **with numbers you actually computed**. Tell me the same in chat at handover.

### SkyrimNet views

1. **Overview** — see Structure above.
2. **The Timeline** *(title it from my data, e.g. "Fourteen Days")* — one vertical stacked bar per in-game
   day, each segment a category colour, bar height ∝ that day's event count, with a day label + total and
   a colour legend. From `events_by_day`.
3. **Constellation of Memory** — an SVG scatter of every memory. Position from the projection; **dot size
   ∝ importance**; **colour by actor** (top ~5 distinct, rest grey). Draw `edges` as faint lines (opacity
   ∝ similarity). Interactions: a **UMAP/PCA toggle** (updates positions + caption; report PCA variance
   fraction), **actor filter chips** (All / top actors / Everyone else) that dim non-matching stars,
   **hover tooltips** (memory text + meta), and **draggable stars** with a light spring sim (home-spring +
   edge-spring + damping) so neighbors tug along. If reduced motion, place stars statically. Add a caption
   pointing narrow-screen readers to **Memories**, where the same records read as a list.
4. **The Memories We Made** — memories as a list. The densest view in the journal; it needs real tools:
   - **Full-text search** over content (match actor and location too), **hits highlighted**, with a live
     "N of M memories shown" count.
   - **Type filters** — chips per `memory_type` with counts, plus "All types".
   - **A mind filter reaching every actor** — a searchable picker over the full actor list with per-actor
     counts, not a chip row of the top 5. A run has ~80 minds; five chips address 6% of them. Build the
     filter input **once** and re-render only the list beneath it as I type: rebuilding the input each
     keystroke re-creates the element with the caret at index 0, so typing "nazeem" yields "meezan" and
     matches nothing.
   - **A sort toggle** — chronological or by importance.
   - **Paging** (~25 rows), paging over **rows, not day groups**, or a 50-memory day blows past the page.
   - **Grouping** — sorted chronologically, group by in-game day with a heading and count, in day-list
     order, sorted within a day by `gt`. Undated memories form their own group **at the end**, never mixed
     into a real day.
   - Each row: actor + `type · emotion` left, the memory text with location beneath in the middle, an
     **importance bar** (labelled `imp <score>`) right.
5. **Blood Leaderboard** — `leaderboard` as a ranked killer table: rank number (`01`, `02`, …), killer
   name (coloured by actor), **total kills**, a horizontal **stacked bar segmented by victim category**
   (People / Undead / Wildlife / Monsters / Dragon, each its colour) with a matching legend above, and a
   row of **victim chips** (`Bandit ×3`, `Draugr ×2`, …).
6. **The Diary** — each entry as a "journal page": a bordered/inset panel, header with author and place, a
   **drop-cap** first letter, paragraphs split on blank lines, `*starred*` phrases as italic emphasis.
   **Render them as a simple stack of pages** — no page-flip widget with prev/next/indicator chrome. Saves
   hold a handful of entries; flip controls cost more than they give and hide entries behind clicks. If no
   diary entry, omit the view.
7. **OmniSight Field Notes** — open with a framed **"A Portrait of the Land"** summary panel: the
   `omni_summary` prose (drop-capped first paragraph, a mono footer naming capture count + date span),
   distilled from **all** captures and grounded only in them. Below, a grid of cards from `screenshots`:
   each with its matched `omnisight-images/` image, subject name, location/metadata line, and description.
   Then:
   - **Lazy-load every image**; render an initial batch (~24) with a "show all N captures" control.
   - **A lightbox.** Clicking or keyboard-activating a card opens the full image with its name, metadata
     and description; **Left/Right** move through the set, **Escape** closes, focus returns to the card.
     Cards are real controls (`role="button"`, `tabindex="0"`, Enter/Space).
   - Captures that couldn't be placed on a day get their own labelled group rather than being dropped.

### IntelEngine views

1. **The Great Powers** — `relations` as **bars sliding along a track**:
   - Sort rows by score **ascending** (most hostile / the war at top).
   - Each row is a grid: **faction A** (right-aligned, coloured, leaders + seat on hover) · a **track**
     (thin full-width rule with a centre tick at 0) · **faction B** (left-aligned, coloured, with an
     inline **"⚔ At War"** tag when at war) · the **score** far right (coloured by sign: red neg / green
     pos / dim zero).
   - On the track, a **coloured bar slides from centre to the score**: negative extends **left in
     blood-red** (hostility); positive extends **right, tinted by faction A's colour** (alliance). Width
     ∝ |score|/100 of each half.
   - Flag any at-war pair (e.g. subtle red glow on the track).
   - Caption in the spirit of: *relations from −100 (open war) to +100 (alliance); each pair drifts as the
     world generates off-screen politics; hover a name for its leaders and seat.*

> **The War and Battles views are conditional.** If no war (Step 2), **omit both entirely** — no empty
> panel, no placeholder — and leave them out of the sidebar. The rest of the IntelEngine group is
> unaffected.

2. **The War** *(only if a war exists)* — one panel per entry in `war`: an intro note (who declared it and
   when, battles fought, victor status), then the two belligerents either side of a `Versus`, each with
   **morale** and **strength** gauges (0–100 bars) and their leaders.
   - **`war` is a list, and my save really may hold several.** Don't assume one. With more than one,
     **title the view "The Wars"**, sort any **ongoing war first** (it's the live situation), and give each
     panel a header naming its belligerents plus an ongoing/victor flag — otherwise several wars read as
     one undifferentiated blur.
   - **The declaration is model prose and usually has no terminal punctuation.** Appending the outcome to
     it produces run-ons like *"…drive the 'Elven puppeteers' from Skyrim Concluded after 3 battles"*.
     Strip trailing punctuation and add your own separator. Same anywhere you concatenate model text with
     your own sentence.
   - Phrase the outcome from the actual numbers: a war with no battles yet is *"No battles fought yet"*,
     not *"the war rages on — 0 battles fought"*.
   - On a narrow screen the two belligerents **stack**; side by side leaves ~150px each, which wraps a
     faction name onto two lines and squashes the gauges.
3. **Battles** *(only if a war exists)* — `battles` rows: location, attacker/defender, result, each
   side's losses, narrative.
4. **Chronicle of Provocations** — `events` rows in time order: event-type tag, description, faction pair,
   signed `relation_delta` (red negative, green positive).
5. **Whispers Beyond the Road** — `gossipdata` as a **responsive card grid**. Above it: a row of **summary
   stat blocks** (total whispers, distinct voices, political count, player-rumor count) and a row of
   **filter chips**. Each **card**:
   - one or more **colour-coded category tags** (below), a top border tinted to its category;
   - a **speaker → listener** header (uppercase, listener in accent);
   - the **scene** as **capitalized italic context ending in a colon** (take `narration`, capitalize first
     letter, strip trailing punctuation, append `:`);
   - the **rumor as a pull-quote** at the card foot (larger serif, left accent rule, decorative quote mark).

   **Classify each whisper** (at render time, scanning the rumor) into:
   - **About the player** *(gold)* — references the player (name, alias/epithet, or a closely associated NPC).
   - **Political** *(steel)* — factions, leaders, war, tariffs, named political locations, related terms.
   - **Personal** *(green)* — everything else.

   Chips are **All** + those three, each with its count. Derive matching terms from the data (player name
   from `uuid_mappings`; faction names/leaders/holds from IntelEngine + `political_state.json`) rather than
   hardcoding. If no log was uploaded, omit this view.

> **IntelEngine keeps its own clock.** `faction_events` carry IntelEngine's own numeric day (e.g. `3.3` …
> `15.4`), which is **not** the in-game calendar (`"20th of Last Seed"`) the timeline and memories use.
> There's no reliable mapping between the two in the data. Never join them, never put IntelEngine events
> on the SkyrimNet timeline, and never render a numeric day as a calendar date. If you surface the number,
> label it as IntelEngine's own.

### Beyond the standard layout

The above is the standard build, but my save is my own. If while reading my data you find something
**genuinely notable that none of those views capture** — a significant world event, completed quest arc,
dragon-soul or bounty milestone, distinctive populated table — **don't silently drop it or cram it into
an ill-fitting view.** Tell me what you found and **ask whether I'd like a view for it.** If yes, design
it in the same aesthetic (shared palette/fonts, its own JSON payload, its own sidebar entry in the right
group). Only add views I've approved.

### Footer

A short italic footer in the sidebar naming the source database files (actual uploaded filenames) and
noting that off-screen gossip is reconstructed from the saved model outputs.

---

## Phase 4 — Verify before handing over

- Confirm it opens offline; **render it** (headless browser ideal) and **visit every route** — each view
  populates, with no empty panels, `undefined`, `NaN`, or console errors (a blocked Google Fonts request
  offline is fine). Visiting one view proves nothing about the others; if you script the sweep, confirm
  the tool is really changing route — a mangled hash, or a navigation to the URL you're already on, will
  silently re-render the current view and pass every check while testing nothing.
- **Then look at every view — and look at it scrolled.** Screenshot each route and actually read the
  picture. Every other check in this list is negative — absence of errors, absence of overflow — and
  **layout breakage is none of those things**, so it sails straight through. A war panel once shipped whose
  belligerent columns collided with the sidebar's CSS class, inherited `height:100vh`, and rendered ~3.5×
  too tall, the "Versus" mark stranded in 800px of dead space and the sidebar's border drawn down the middle
  of the panel — no console error, no `undefined`, no horizontal overflow, every interaction working. Only
  looking finds that.
- **Scroll before you believe it.** A screenshot at the top of the page is not evidence the page works: at
  `scrollY = 0` a broken `position:sticky` element is pixel-identical to a working one. This exact hole let
  a broken sidebar ship *after* the rule above was already being followed — the page was screenshotted, it
  looked perfect, and the sidebar scrolled away and got cut in half the moment a reader moved. So: on a
  long view, **scroll down and confirm the sidebar and the view header are still pinned at `top: 0` and the
  sidebar's footer is still on screen**; on narrow screens, the same for the sticky top bar. Assert it
  (`getBoundingClientRect().top === 0` while `scrollY > 0`), don't eyeball it.
- Two smells to measure as well as eyeball: **a container far taller than the content it holds** (something
  is inheriting a height it shouldn't), and **a view whose text length is implausibly short** for the
  records it should show.
- Confirm every interaction: sidebar routing and the **back button**; UMAP/PCA toggle; the constellation's
  actor chips dimming stars; **memory search** (hits highlighted, live count right, and markup in the
  query rendered as text rather than injected); type/mind/sort filters; **paging**; the OmniSight
  show-all; the **capture lightbox** (opens, Left/Right move, Escape closes, focus returns); whisper
  filters.
- Confirm every capture image resolves — count the files in `omnisight-images/` against the payload, and
  check the lightbox shows a decoded image, not a broken reference.
- Confirm layout: **no horizontal overflow** at desktop and mobile; on narrow screens the drawer opens and
  the scrim/Escape close it.
- Confirm the name, day count, "The Adventures of …" title, and every stat came from **my** data — not
  any example — and the OmniSight summary is grounded in my actual captures.
- Confirm **nothing was silently dropped**: undated memories appear in their own group, unplaceable
  captures have a home, and any notice states counts you actually computed.
- Tell me plainly what you fell back on and what my data couldn't support — e.g. "UMAP not installed, used
  t-SNE"; "no log provided, gossip omitted"; "no OmniSight images, text-only capture cards"; "6 memories
  carry a real-world timestamp instead of the in-game clock and 1 has none, so 7 are grouped as Undated".
  Report the real numbers from my save, not these examples. Papering over a gap is worse than the gap.
- Then give me the finished `.html` file.
