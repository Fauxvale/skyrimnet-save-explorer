Build me a **SkyrimNet Save Explorer**: one self-contained, offline HTML file that turns my Skyrim
AI-mod playthrough (the **SkyrimNet** and **IntelEngine** mods) into an elegant, interactive "save
explorer" — an illuminated field journal for a single character's run.

The page has **two tabs**:

- **SkyrimNet** — the character's lived experience: event timeline, an interactive "constellation" of
  memories, the memories themselves, a kill ledger, a diary page, and OmniSight field notes (a distilled
  "portrait of the land" + matched capture images), plus any other notable story beats found in the logs.
- **IntelEngine** — the political layer: faction relationships, a chronicle of provocations, off-screen
  gossip, and — **only if my save actually has one** — an ongoing war and its battles.

Every section is **collapsible**, there is a **floating table of contents** that reflows into a mobile
bar + overlay on narrow screens, and everything — name, in-game day count, every count and label — must
be **derived from my uploaded data**. Nothing is hardcoded from any example.

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
  `game_time`/related event), `gt` (raw `game_time`, for sorting), and 2-D coords `x, y` (PCA) and
  `ux, uy` (UMAP) — see the projection recipe.
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
  `description`, signed `relation_delta`, ordered by `game_time`.

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

- **Header** — centered, reading **"The Adventures of &lt;name&gt;"**: a small gold eyebrow label
  (switches per tab, e.g. "A SkyrimNet Field Journal" / "An IntelEngine Dispatch"), a small italic serif
  kicker **"The Adventures of"**, then the name **large and monumental** (uppercase display face). Below:
  the italic subtitle and a row of stat blocks divided by thin rules (including Total Spend if opted in).
- **Two tabs** (`SkyrimNet` / `IntelEngine`) swapping the visible content and the TOC.
- **No divider ornaments between sections** — section headers alone do the work: roman numeral +
  uppercase title + trailing hairline. (No decorative knot/rule strips — they waste vertical space.)
- **Every section collapsible.** The header is a real toggle (`role="button"`, `tabindex="0"`,
  `aria-expanded`, Enter/Space) with a chevron that rotates when collapsed; collapsing hides the body.
  **Exception:** in the Constellation, the actor-filter chip row stays **outside** the collapsible body
  ("keep-visible") so it keeps filtering the memory list (Section III) even when the constellation is
  collapsed.
- **Floating TOC (desktop, ≳1120px):** pinned left, **vertically centered**, with a **roman-numeral index
  badge** per entry and a **clean active highlight** (active badge fills with accent, its label takes the
  accent, a thin accent rule marks it); scroll-spy keeps it in sync. Main content must **shift clear of
  the sidebar** (never overlap) and **re-center on wide screens** — e.g. give each centered column
  `margin-left: max(<sidebar-clearance>, calc((100vw − <content-max>)/2)); margin-right:auto;`.
- **TOC (narrow, ≲1120px):** replace the sidebar with a **sticky gradient top bar** (below the tab bar)
  showing current tab + roman badge + section name, with a **hamburger** opening a **full-screen overlay
  list** of all sections (roman badges, large tap targets); tapping scrolls to it and closes the overlay.
  Scroll-spy updates the bar's label.
- Confirm **no horizontal overflow** at any width (let section-header captions wrap).

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
- **Palette (`:root`):** `--bg:#14161a; --panel:#1a1d23; --text:#e8e4da; --dim:#98938a; --gold:#c9a45c;
  --steel:#7da3b8; --blood:#b0584e; --moss:#8ba06b;` plus hairlines like `rgba(232,228,218,.16)`.
- **Category colours:** Dialogue=gold, Death=blood, Thoughts=steel, Trade/world=moss, Followers=dim/grey.
- **Motifs:** centered header, "double rule" hairlines, roman-numeral section headers with trailing
  hairlines, the collapsible chevron, the floating TOC / mobile bar above.

*(Everything below is the structural baseline for whichever theme I pick.)*

### SkyrimNet tab — sections

1. **The Timeline** *(e.g. "Seven Days in Whiterun Hold" — derive the hold if one dominates)* — one
   vertical stacked bar per in-game day, each segment a category colour, bar height ∝ that day's event
   count, with a day label + total and a colour legend. From `events_by_day`.
2. **Constellation of Memory** — an SVG scatter of every memory. Position from the projection; **dot size
   ∝ importance**; **colour by actor** (top ~5 distinct, rest grey). Draw `edges` as faint lines (opacity
   ∝ similarity). Interactions: a **UMAP/PCA toggle** (updates positions + caption; report PCA variance
   fraction), **actor filter chips** (All / top actors / Everyone else) that dim non-matching stars **and**
   filter the Section III list, **hover tooltips** (memory text + meta), and **draggable stars** with a
   light spring sim (home-spring + edge-spring + damping) so neighbors tug along. If reduced motion, place
   stars statically. The **actor-filter chip row stays visible when this section is collapsed**. Add a
   small **mobile caption** beneath the scatter pointing to Section III (same memories as a readable list).
3. **The Memories We Made** — memories as a list, grouped by in-game day (each day heading shows its
   memory count and, if available, a one-line day summary). Each row: actor + `type · emotion` left, the
   memory text with location beneath in the middle, an **importance bar** (labelled `imp <score>`) right.
   Two chip rows: **type filters** (All types + one per `memory_type`, with counts) and a **sort toggle**
   (chronological / by importance); the Constellation's actor filter applies here too. Include a short
   **note that the mind filter above also filters this list** (the two views stay in sync). Show a live
   "N of M memories shown" note.
4. **Blood Leaderboard** — `leaderboard` as a ranked killer table: rank number (`01`, `02`, …), killer
   name (coloured by actor), **total kills**, a horizontal **stacked bar segmented by victim category**
   (People / Undead / Wildlife / Monsters / Dragon, each its colour) with a matching legend above, and a
   row of **victim chips** (`Bandit ×3`, `Draugr ×2`, …).
5. **The Diary** — each entry as a centered "journal page": a bordered/inset panel, header with author and
   place, a **drop-cap** first letter, paragraphs split on blank lines, `*starred*` phrases as italic
   emphasis. If several, present as a book-like stack that can be flipped through. If no diary entry, omit.
6. **OmniSight Field Notes** — open with a framed **"A Portrait of the Land"** summary panel: the
   `omni_summary` prose (drop-capped first paragraph, a mono footer naming capture count + date span),
   distilled from **all** captures and grounded only in them. Below, a grid of cards from `screenshots`:
   each with its matched `omnisight-images/` image, subject name, location/metadata line, and description,
   with a "show more"/"show all" control if there are many.

### IntelEngine tab — sections

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

> **Sections 2–3 are conditional on a war.** If no war (Step 2), **omit both the war panel and battles
> list entirely** — no empty panel, no placeholder — and renumber so the tab reads naturally (Relations,
> Provocations, Whispers).

2. **The War** *(only if a war exists)* — a panel for the active `war`: an intro note (who declared it and
   when, battles fought, victor status), then the two belligerents either side of a `Versus`, each with
   **morale** and **strength** gauges (0–100 bars) and their leaders.
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
   hardcoding. If no log was uploaded, hide this section and note why.

### Beyond the standard layout

The above is the standard build, but my save is my own. If while reading my data you find something
**genuinely notable that none of those sections capture** — a significant world event, completed quest
arc, dragon-soul or bounty milestone, distinctive populated table — **don't silently drop it or cram it
into an ill-fitting section.** Tell me what you found and **ask whether I'd like a new section for it.**
If yes, design it in the same aesthetic (roman-numeral header, shared palette/fonts, its own JSON
payload, collapsible) and add it to the right tab and the TOC. Only add sections I've approved.

### Footer

A short italic footer naming the source database files (actual uploaded filenames) and noting that
off-screen gossip is reconstructed from the saved model outputs.

---

## Phase 4 — Verify before handing over

- Confirm it opens offline; **render it** (headless browser ideal) and check every section populates
  (no empty panels, `undefined`, `NaN`, or console errors — a blocked Google Fonts request offline is
  fine). If OmniSight image embeds were requested, confirm the images are present in `omnisight-images/`
  and render in the cards.
- Confirm every interaction: tab switching; UMAP/PCA toggle; actor/type/sort filters and the live count;
  diary flip; OmniSight show-more; whisper filters; **every section collapses and re-expands**; and the
  **Constellation's actor chips stay visible and keep filtering the memory list while collapsed**.
- Confirm layout: **no horizontal overflow** at desktop and mobile; on desktop the **floating TOC never
  overlaps content** and content **re-centers on wide screens**; on narrow the **mobile TOC bar +
  hamburger overlay** work.
- Confirm the name, day count, "The Adventures of …" title, and every stat came from **my** data — not
  any example — and the OmniSight summary is grounded in my actual captures.
- Tell me plainly what you fell back on (e.g. "UMAP not installed, used t-SNE", "no log provided, gossip
  omitted", "no OmniSight images, text-only capture cards") rather than papering over gaps.
- Then give me the finished `.html` file.
