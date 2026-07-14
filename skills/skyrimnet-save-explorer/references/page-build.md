# Phase 3 — Build the page

Produce **one self-contained `.html` file**. No build step, no frameworks, no external JavaScript or CSS
libraries. The only external references allowed are a Google Fonts `<link>` and any fonts/images embedded
as base64 data URIs. Embed the payloads as `<script type="application/json">` blocks and render
everything with **vanilla JS**. It must open by double-clicking the file offline. Honor
`prefers-reduced-motion` throughout, keep visible keyboard focus, and be responsive down to mobile.

## Structure (applies to every theme)

- **Header** — a centered header that reads **"The Adventures of &lt;name&gt;"**: a small gold eyebrow
  label (which switches per tab, e.g. "A SkyrimNet Field Journal" / "An IntelEngine Dispatch"), then a
  small italic serif kicker **"The Adventures of"**, then the character's name set **large and
  monumental** (uppercase display face). Below it, the italic subtitle and a row of stat blocks divided
  by thin rules (including Total Spend if the user opted in).
- **Two tabs** at the top (`SkyrimNet` / `IntelEngine`) that swap the visible content and the table of
  contents.
- **No divider ornaments between sections** — section headers alone do the work: a roman numeral + an
  uppercase title + a trailing hairline. (Don't add decorative "knot"/rule divider strips between
  sections; they waste vertical space.)
- **Every section is collapsible.** The section header is a real toggle (`role="button"`, `tabindex="0"`,
  `aria-expanded`, keyboard Enter/Space) with a chevron that rotates when collapsed; collapsing hides the
  section body. **Exception:** in the Constellation section, the actor-filter chip row stays **outside**
  the collapsible body ("keep-visible") so those chips remain on screen and keep filtering the memory
  list (Section III) even when the constellation itself is collapsed.
- **Floating table of contents (desktop, ≳1120px):** pinned to the left, **vertically centered**, with a
  **roman-numeral index badge** beside each entry and a **clean active-section highlight** (the active
  badge fills with the accent colour, its label takes the accent colour, and a thin accent rule marks
  it); a scroll-spy keeps it in sync. The main page content must **shift clear of the pinned sidebar so
  they can never overlap**, and **re-center on wide screens** — e.g. give each centered column
  `margin-left: max(<sidebar-clearance>, calc((100vw − <content-max>)/2)); margin-right:auto;` so it hugs
  clear of the sidebar when the viewport is tight and returns to true center once there's room.
- **Table of contents (narrow, ≲1120px):** the sidebar is replaced by a **gradient top bar** (sticky,
  just below the tab bar) showing the current tab + roman badge + current section name, with a
  **hamburger** that opens a **full-screen overlay list** of all sections (roman badges, larger tap
  targets); tapping an entry scrolls to it and closes the overlay. The scroll-spy updates the bar's
  current-section label.
- Confirm there is **no horizontal overflow** at any width (watch section-header captions — let them
  wrap).

## Look & feel — ask the user which theme

**Before you settle on any colors, fonts, or overall styling, ask the user to pick one of two:**

1. **Preset theme — Illuminated Manuscript** (details below). If they pick this, apply it exactly; no
   need to ask anything further about design.
2. **Their own theme** — the user describes a color scheme and/or overall design they'd prefer. If they
   pick this, **invoke your design skill** (if available) and design the site to their specification —
   palette, typography, and styling — while keeping the same structure, sections, and interactions
   described here. Show them the resulting palette/type choices before building the full page. If no
   design skill is available, apply solid visual-design fundamentals to their brief. **If the user
   supplies their own fonts, images, or other assets** (see `data-sources.md`, Step 4), embed and use
   them directly as base64 data URIs rather than approximating them.

### Theme option 1 — Illuminated Manuscript (preset)

- **Fonts:** `Oswald` (uppercase, letter-spaced headings/labels), `Crimson Pro` (serif body),
  `JetBrains Mono` (numbers/metadata), via a Google Fonts `<link>`.
- **Palette (CSS `:root`):**
  `--bg:#14161a; --panel:#1a1d23; --text:#e8e4da; --dim:#98938a; --gold:#c9a45c; --steel:#7da3b8;
  --blood:#b0584e; --moss:#8ba06b;` plus subtle hairlines like `rgba(232,228,218,.16)`.
- **Category colours:** Dialogue = gold, Death = blood, Thoughts = steel, Trade/world = moss,
  Followers = dim/grey.
- **Motifs:** centered header, "double rule" style hairlines, roman-numeral section headers with trailing
  hairlines, the collapsible chevron, and the floating TOC / mobile bar described above.

*(Everything below is the structural baseline for whichever theme the user picks.)*

## SkyrimNet tab — sections

1. **The Timeline** *(e.g. "Seven Days in Whiterun Hold" — derive the hold from the data if one
   dominates)* — one vertical stacked bar per in-game day, each segment a category colored per the theme,
   bar height ∝ that day's event count, with a day label + total and a color legend. Built from
   `events_by_day`.
2. **Constellation of Memory** — an SVG scatter of every memory. Position from the projection; **dot size
   ∝ importance**; **color by actor** (top ~5 actors distinct, the rest grey). Draw `edges` as faint
   lines (opacity ∝ similarity). Interactions: a **UMAP/PCA toggle** (updates positions and the caption;
   report the PCA variance fraction), **actor filter chips** (All / top actors / Everyone else) that dim
   non-matching stars **and** filter the Section III memory list, **hover tooltips** showing the memory
   text + meta, and **draggable stars** with a light spring simulation (home-spring + edge-spring +
   damping) so neighbors tug along. If reduced motion is set, place stars statically. The **actor-filter
   chip row stays visible when this section is collapsed** (see structure). Add a small **caption for
   mobile readers** beneath the scatter pointing them to Section III, where the same memories appear as a
   readable list.
3. **The Memories We Made** — the memories as a list, grouped by in-game day (each day gets a heading with
   its memory count and, if available, a one-line summary of that day). Each row: actor + `type · emotion`
   on the left, the memory text with its location beneath in the middle, and an **importance bar**
   (labelled `imp <score>`) on the right. Two chip rows drive it — **type filters** (All types + one per
   `memory_type`, with counts) and a **sort toggle** (chronological / by importance) — and the
   Constellation's actor filter applies here too. Include a short **note stating that the mind filter from
   the Constellation above also filters this list** (the two views stay in sync). Show a live "N of M
   memories shown" note.
4. **Blood Leaderboard** — the `leaderboard` payload as a ranked table of killers: a rank number
   (`01`, `02`, …), the killer's name (colored by their actor color), their **total kills**, a horizontal
   **stacked bar segmented by victim category** (People / Undead / Wildlife / Monsters / Dragon, each its
   own color) with a matching legend above, and a row of **victim chips** (`Bandit ×3`, `Draugr ×2`, …).
5. **The Diary** — render each diary entry as a centered "journal page": a bordered/inset panel, a header
   with the author and place, a **drop-cap** first letter, paragraphs split on blank lines, and
   `*starred*` phrases shown as italic emphasis. If several entries exist, present them as a book-like
   stack that can be flipped through. If there is no diary entry, omit the section.
6. **OmniSight Field Notes** — open with a framed **"A Portrait of the Land"** summary panel: the
   synthesized `omni_summary` prose (drop-capped first paragraph, a mono footer line naming the capture
   count and date span), distilled from **all** captures and grounded only in what they describe. Below
   it, a grid of cards from `screenshots`: each card should include the matched image from
   `omnisight-images/`, the subject name, the location/metadata line, and the description prose, with a
   "show more"/"show all" control if there are many.

## IntelEngine tab — sections

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

> **Sections 2–3 are conditional on there being a war.** If the data has no war (see `data-sources.md`,
> Step 2), **omit both the war panel and the battles list entirely** — no empty panel, no "no war found"
> placeholder — and let the remaining IntelEngine sections renumber so the tab reads naturally
> (Relations, then Provocations, then Whispers).

2. **The War** *(only if a war exists)* — a panel for the active `war`: an intro note (who declared it and
   when, battles fought so far, victor status), then the two belligerents either side of a `Versus`, each
   with **morale** and **strength** gauges (0–100 bars) and their leaders.
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
   `political_state.json`) rather than hardcoding any character. If no log was uploaded, hide this section
   and note why.

## Beyond the standard layout

The sections above are the standard build, but each save is the user's own. If, while reading their data,
you find something **genuinely notable that none of those sections capture** — a significant world event,
a completed quest arc, a dragon-soul or bounty milestone, a distinctive populated table — **don't
silently drop it, and don't cram it into an ill-fitting section.** Tell the user what you found and **ask
whether they'd like a new section built for it.** If yes, design it in the same aesthetic (roman-numeral
header, the shared palette and fonts, its own JSON payload, collapsible like the rest) and add it to the
appropriate tab and the table of contents. Only add sections the user has approved — never invent one
unprompted.

## Footer

A short italic footer naming the source database files (use the actual uploaded filenames) and noting
that off-screen gossip is reconstructed from the saved model outputs.
