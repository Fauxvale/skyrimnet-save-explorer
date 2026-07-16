# Phase 3 — Build the page

Produce **one self-contained `.html` file**. No build step, no frameworks, no external JavaScript or CSS
libraries. The only external references allowed are a Google Fonts `<link>` and any fonts/images embedded
as base64 data URIs. Embed the payloads as `<script type="application/json">` blocks and render
everything with **vanilla JS**. It must open by double-clicking the file offline. Honor
`prefers-reduced-motion`, keep visible keyboard focus, and be responsive down to mobile.

## Structure (applies to every theme)

**The page is an app shell: a sidebar and one view at a time.** This is the most important structural
rule here, and the reason is worth stating. A save explorer holds hundreds of records — a real
playthrough runs to ~255 memories, ~163 captures, ~54 gossip records. Printing every table onto one
scroll, expanded, means the reader scrolls past hundreds of rows to reach anything, and the page ends up
organised **by database table rather than by anything a reader cares about**. Don't do that. Give each
section its own screen and let the reader choose one.

- **Sidebar (desktop, ≳900px)** — pinned full height on the left, with its own scroll.
  - **Brand block** at the top: a small italic serif kicker **"The Adventures of"**, the character's name
    in the uppercase display face beneath it, then a mono line with the in-game date span. A thin rule
    closes the block.
  - **Nav entries** below, in groups: an ungrouped **Overview** first, then a **SkyrimNet** group, then an
    **IntelEngine** group (label each group with a small mono all-caps header). Each entry shows its
    **record count** on the right — the counts are the reader's map of where the substance is.
  - **Active entry** takes an accent left rule, an accent label, and an accent count.
  - A small footer notes the source database filenames.
- **Views and routing** — each section is a view, addressed by a **hash route** (`#/memories`,
  `#/captures`). Only the active view is in the DOM. The browser back button must work. Landing with no
  hash shows Overview.
- **View header** — sticky at the top of the content column: a small accent kicker naming the half of the
  journal this view belongs to (e.g. "A SkyrimNet Field Journal" / "An IntelEngine Dispatch"), the view
  title, and an italic caption saying what the reader is looking at. Overview is the exception — it
  carries its own masthead, so it needs no view header.
- **Overview is the landing view.** It must answer "what happened in this run?" without scrolling far:
  - A **masthead** — the monumental title card the journal deserves: italic "The Adventures of", the
    character's name set **large** (uppercase display face), then the italic subtitle. Overview is the
    only place this appears; on every view it would cost a screenful each time.
  - A **stat row** — the headline numbers from `stats` (including Total Spend if the user opted in).
  - Any **data notices** (see "Be honest about gaps").
  - A grid of **panels**, each summarising one section and linking into it: activity-by-day bars, the
    busiest minds, the heaviest memories, a few captures, and the war if there is one. Every panel needs a
    way through to its full view.
- **Conditional views** — a view whose data is empty is **omitted from the nav entirely**. No empty
  panels, no "no war found" placeholders. This already applied to war, battles, diary and gossip; it now
  applies to every view. Nothing renumbers, because nothing is numbered.
- **Narrow screens (≲900px)** — the sidebar becomes an **off-canvas drawer**: a sticky top bar shows the
  current view name and a hamburger; tapping it slides the sidebar in over a scrim. Tapping the scrim,
  choosing an entry, or pressing Escape closes it.
- Confirm there is **no horizontal overflow** at any width (watch long captions — let them wrap).

### What this replaces, and why

Earlier versions of this skill specified a two-tab bar, a floating roman-numeral table of contents that
reflowed into a mobile bar + overlay, and collapsible sections on one long scroll. **All of that is
gone.** If you are working from an older example page, do not copy its shell.

- **Tabs → sidebar groups.** Two tabs hid half the journal behind a click and still left each tab a long
  scroll. SkyrimNet and IntelEngine are groups in one nav, so every section is one click from everywhere.
- **Floating TOC → the sidebar.** The TOC existed to navigate a scroll that no longer exists. The sidebar
  *is* the table of contents — and being a real column rather than a floating overlay, it can't overlap
  the content or need re-centering, which the old spec had to work around.
- **Collapsible sections → routing.** Collapsing was a workaround for too much on one page. With one view
  at a time there is nothing to collapse. Drop the chevrons and the `aria-expanded` toggles.
- **Roman numerals → record counts.** Numerals implied a fixed reading order. A routed shell has none —
  the reader picks. A count ("255", "163") tells them something the numerals never did.
- **The "keep-visible" chip row → per-view filters.** That exception existed only because the
  constellation's actor chips also drove the memory list further down the same scroll. Those are separate
  views now, so the coupling is gone: the constellation owns its filter, Memories owns its own. Delete the
  exception, and the "the filter above also filters this list" note with it.

## Look & feel — ask the user which theme

**Before you settle on any colors, fonts, or overall styling, ask the user to pick one of two:**

1. **Preset theme — Illuminated Manuscript** (details below). If they pick this, apply it exactly; no
   need to ask anything further about design.
2. **Their own theme** — the user describes a color scheme and/or overall design they'd prefer. If they
   pick this, **invoke your design skill** (if available) and design the site to their specification —
   palette, typography, and styling — while keeping the same structure, views, and interactions described
   here. Show them the resulting palette/type choices before building the full page. If no design skill is
   available, apply solid visual-design fundamentals to their brief. **If the user supplies their own
   fonts, images, or other assets** (see `data-sources.md`, Step 4), embed and use them directly as base64
   data URIs rather than approximating them.

### Theme option 1 — Illuminated Manuscript (preset)

- **Fonts:** `Oswald` (uppercase, letter-spaced headings/labels), `Crimson Pro` (serif body),
  `JetBrains Mono` (numbers/metadata), via a Google Fonts `<link>`.
- **Palette (CSS `:root`):**
  `--bg:#14161a; --panel:#1a1d23; --panel2:#20242b; --text:#e8e4da; --dim:#98938a; --gold:#c9a45c;
  --steel:#7da3b8; --blood:#b0584e; --moss:#8ba06b; --violet:#a98cc0; --amber:#cf9a54;` plus subtle
  hairlines like `rgba(232,228,218,.16)`.
- **Category colours:** Dialogue = gold, Death = blood, Thoughts = steel, Trade/world = moss,
  Followers = dim/grey. Reserve **amber** for data-gap notices, so a gap never reads as content.
- **Motifs:** the monumental Overview masthead, "double rule" style hairlines, uppercase letter-spaced
  view titles with a trailing hairline, and the accent-marked active sidebar entry.

*(Everything below is the structural baseline for whichever theme the user picks.)*

## Rules that apply to every view

These are not garnish; they are the difference between a page a reader can use and a data dump with a
nice font.

- **Any list longer than ~25 records is paged or virtualized.** Never print 255 rows at once. Show a live
  "N of M shown" count so the reader always knows what they're looking at.
- **Any collection of images is lazy-loaded** (`loading="lazy"`) and rendered on demand. Do not build 163
  `<img>` tags up front in order to show 12 of them.
- **Every image opens in a lightbox.** An image the reader can't enlarge is a thumbnail of nothing.
- **Any filter over a long list is searchable**, not a fixed row of chips. Chips are fine for a handful of
  categories; they cannot address 80 actors.
- **Escape every piece of model text before it reaches the DOM.** Memory content, rumors and descriptions
  are model output and may contain `<`, `&`, or markup. When highlighting a search hit, escape **first**
  and then wrap the match — otherwise a crafted query becomes live markup.
- **Never silently drop a record.**
- **One file means one CSS namespace — name every class for what it is.** This page puts a full-height
  sidebar and a war panel of two belligerents in the same stylesheet, and generic names collide. Calling
  both `.side` is a real bug that has shipped here: the belligerent columns inherited the sidebar's
  `height:100vh; display:flex; border-right`, which blew the war panel to ~3.5× its proper height, stranded
  the "Versus" mark in 800px of dead space, and painted the sidebar's border down the middle of the panel —
  with no console error and no horizontal overflow to give it away. Prefer `.sidebar` and `.belligerent`
  over `.side`. The same trap is waiting in `.side`/`.stat`/`.page`/`.view`/`.card`/`.row`.

### Be honest about gaps

Some records cannot be placed. A memory whose timestamp is unusable has no day; a capture whose date
matches no known day has nowhere to sit (see `payloads.md`, "Timestamp hygiene"). **Do not drop these and
do not guess.**

- Give them a visible home — an "Undated" group at the end of the memory list, an explicit bucket for
  unplaceable captures.
- Where a gap is worth explaining, render a short **notice** (amber accent, visually distinct from
  content) on Overview or at the head of the affected list, saying plainly how many records are affected
  and why. **Derive every number from the data — never state a count you haven't computed.**
- Tell the user about the same gaps in chat when you hand the page over (Phase 4).

## SkyrimNet views

1. **Overview** — see Structure above.
2. **The Timeline** *(title it from the data, e.g. "Fourteen Days")* — one vertical stacked bar per
   in-game day, each segment a category colored per the theme, bar height ∝ that day's event count, with
   a day label + total and a color legend. Built from `events_by_day`.
3. **Constellation of Memory** — an SVG scatter of every memory. Position from the projection; **dot size
   ∝ importance**; **color by actor** (top ~5 actors distinct, the rest grey). Draw `edges` as faint lines
   (opacity ∝ similarity). Interactions: a **UMAP/PCA toggle** (updates positions and the caption; report
   the PCA variance fraction), **actor filter chips** (All / top actors / Everyone else) that dim
   non-matching stars, **hover tooltips** showing the memory text + meta, and **draggable stars** with a
   light spring simulation (home-spring + edge-spring + damping) so neighbors tug along. If reduced motion
   is set, place stars statically. Add a caption pointing narrow-screen readers to **Memories**, where the
   same records appear as a readable list.
4. **The Memories We Made** — the memories as a list. This is the densest view in the journal, and it
   needs real tools:
   - **Full-text search** over memory content (match actor and location too), with **highlighted hits**
     and a live "N of M memories shown" count.
   - **Type filters** — chips, one per `memory_type`, with counts, plus "All types".
   - **A mind filter that reaches every actor** — a searchable picker over the full actor list with
     per-actor memory counts, not a chip row of the top 5. A run has ~80 minds; five chips address 6% of
     them. Build the filter input **once** and re-render only the list beneath it as the reader types:
     rebuilding the input on each keystroke re-creates the element with the caret at index 0, so typing
     "nazeem" yields "meezan" and matches nothing.
   - **A sort toggle** — chronological or by importance.
   - **Paging** (~25 rows). Page over rows, not day groups, or a 50-memory day blows past the page size.
   - **Grouping** — when sorted chronologically, group by in-game day with a heading and that day's count,
     in day-list order, sorted within a day by `gt`. Undated memories form their own group **at the end**,
     never mixed into a real day.
   - Each row: actor + `type · emotion` on the left, the memory text with its location beneath in the
     middle, and an **importance bar** (labelled `imp <score>`) on the right.
5. **Blood Leaderboard** — the `leaderboard` payload as a ranked table of killers: a rank number
   (`01`, `02`, …), the killer's name (colored by their actor color), their **total kills**, a horizontal
   **stacked bar segmented by victim category** (People / Undead / Wildlife / Monsters / Dragon, each its
   own color) with a matching legend above, and a row of **victim chips** (`Bandit ×3`, `Draugr ×2`, …).
6. **The Diary** — each entry as a "journal page": a bordered/inset panel, a header with the author and
   place, a **drop-cap** first letter, paragraphs split on blank lines, and `*starred*` phrases as italic
   emphasis. **Render the entries as a simple stack of pages** — do not build a page-flip widget with
   prev/next/indicator chrome. Saves typically hold a handful of entries; flip controls cost more than
   they give and hide entries behind clicks. If there is no diary entry, omit the view.
7. **OmniSight Field Notes** — open with a framed **"A Portrait of the Land"** summary panel: the
   synthesized `omni_summary` prose (drop-capped first paragraph, a mono footer line naming the capture
   count and date span), distilled from **all** captures and grounded only in what they describe. Below
   it, a grid of cards from `screenshots`: the matched image from `omnisight-images/`, the subject name,
   the location/metadata line, and the description prose. Then:
   - **Lazy-load every image** and render an initial batch (~24) with a "show all N captures" control.
   - **A lightbox.** Clicking or keyboard-activating a card opens the full image with its name, metadata
     and description. **Left/Right arrows** move through the set, **Escape** closes, and focus returns to
     the card that opened it. Cards are real controls (`role="button"`, `tabindex="0"`, Enter/Space).
   - If some captures could not be placed on a day, give them their own labelled group rather than
     dropping them.

## IntelEngine views

1. **The Great Powers** — the `relations`, rendered as **bars that slide along a track**:
   - Sort the rows by score **ascending** (most hostile / the war at the top).
   - Each row is a grid: **faction A name** (right-aligned, coloured by faction, with its leaders + seat
     shown on hover) · a **track** (a thin full-width rule with a centre tick at 0) · **faction B name**
     (left-aligned, coloured, with an inline **"⚔ At War"** tag when the pair is at war) · the **score**
     on the far right (coloured by sign: red negative / green positive / dim zero).
   - On the track, a **colored bar segment slides from the centre to the score position**: negative
     scores extend **left in blood-red** (hostility); positive scores extend **right, tinted by faction
     A's colour** (alliance). Width ∝ |score|/100 of each half.
   - Flag any at-war pair (e.g. a subtle red glow on the track).
   - Caption in the spirit of: *relations from −100 (open war) to +100 (alliance); each pair drifts as
     the world generates off-screen politics; hover a name for its leaders and seat.*

> **The War and Battles views are conditional.** If the data has no war (see `data-sources.md`, Step 2),
> **omit both entirely** — no empty panel, no placeholder — and leave them out of the sidebar. The rest of
> the IntelEngine group is unaffected.

2. **The War** *(only if a war exists)* — one panel per entry in `war`, each with an intro note (who
   declared it and when, battles fought so far, victor status), then the two belligerents either side of a
   `Versus`, each with **morale** and **strength** gauges (0–100 bars) and their leaders.
   - **`war` is a list, and a save really can hold several.** Don't assume one. If there is more than one,
     **title the view "The Wars"**, sort any **ongoing war first** (it's the live situation), and give each
     panel its own header naming its belligerents plus an ongoing/victor flag — otherwise several wars read
     as one undifferentiated blur.
   - **The declaration is model prose and usually has no terminal punctuation.** Appending the outcome to
     it produces run-ons like *"…drive the 'Elven puppeteers' from Skyrim Concluded after 3 battles"*.
     Strip trailing punctuation and add your own separator. The same applies anywhere you concatenate
     model text with your own sentence.
   - Phrase the outcome from the actual numbers: a war with no battles yet is *"No battles fought yet"*,
     not *"the war rages on — 0 battles fought"*.
   - On a narrow screen the two belligerents **stack**; side by side leaves ~150px each, which wraps a
     faction name onto two lines and squashes the gauges.
3. **Battles** *(only if a war exists)* — `battles` rows: location, attacker/defender, result, each
   side's losses, and the narrative.
4. **Chronicle of Provocations** — `events` rows in time order: an event-type tag, the description, the
   faction pair, and the signed `relation_delta` (red for negative, green for positive).
5. **Whispers Beyond the Road** — the `gossipdata` as a **responsive card grid**. Above the grid, a row
   of **summary stat blocks** (total recorded whispers, distinct voices involved, political count,
   player-rumor count) and a row of **filter chips**. Each **card** shows:
   - one or more **colour-coded category tags** (see classification below), and a top border tinted to
     the card's category;
   - a **speaker → listener** header (uppercase, the listener in the accent colour);
   - the **scene** as **capitalized italic context ending in a colon** (take `narration`, capitalize the
     first letter, strip trailing punctuation, append `:`);
   - the **rumor as a pull-quote** anchored at the card's foot (larger serif, a left accent rule, a
     decorative quotation mark).

   **Classify each whisper** (at render time, by scanning the rumor text) into:
   - **About the player** *(gold tag)* — references the player character (by name, an alias/epithet, or a
     closely associated NPC).
   - **Political** *(steel tag)* — references factions, leaders, war, tariffs, named political locations,
     or related terms.
   - **Personal** *(green tag)* — everything else.

   The chips are **All** + those three buckets, each showing its count. Derive the matching terms from the
   data (the player's name from `uuid_mappings`; faction names/leaders/holds from IntelEngine +
   `political_state.json`) rather than hardcoding any character. If no log was uploaded, omit this view.

> **IntelEngine keeps its own clock.** `faction_events` carry IntelEngine's own numeric day (e.g. `3.3` …
> `15.4`), which is **not** the in-game calendar (`"20th of Last Seed"`) the timeline and memories use.
> There is no reliable mapping between the two in the data. Never join them, never place IntelEngine
> events on the SkyrimNet timeline, and never render a numeric day as though it were a calendar date. If
> you surface the number at all, label it as IntelEngine's own.

## Beyond the standard layout

The views above are the standard build, but each save is the user's own. If, while reading their data,
you find something **genuinely notable that none of those views capture** — a significant world event, a
completed quest arc, a dragon-soul or bounty milestone, a distinctive populated table — **don't silently
drop it, and don't cram it into an ill-fitting view.** Tell the user what you found and **ask whether
they'd like a view built for it.** If yes, design it in the same aesthetic (the shared palette and fonts,
its own JSON payload, its own sidebar entry in the appropriate group). Only add views the user has
approved — never invent one unprompted.

## Footer

A short italic footer in the sidebar naming the source database files (use the actual uploaded filenames)
and noting that off-screen gossip is reconstructed from the saved model outputs.
