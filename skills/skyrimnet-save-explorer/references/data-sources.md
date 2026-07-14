# Phase 1 — Collect the files (one at a time)

Ask the user for **one file at a time**, in the order below. After each upload, **inspect it and report
what you found** (row counts, the character's name, the date range, anything missing) before asking for
the next. Wait for each upload. While inspecting, also **flag anything notable the standard layout
doesn't cover** — a major world event, quest arc, milestone, unusual populated table — and ask whether
they'd like a dedicated section for it (see "Beyond the standard layout" in `page-build.md`).

The databases are **SQLite**. Inspect them with Python's `sqlite3` module (the CLI may be absent).
**Read-only SELECTs only — never modify the user's files.** If a read-only URI handle
(`file:...?mode=ro`) fails to open (e.g. a read-only mount), copy the DB to a writable temp location and
open the copy, still issuing SELECTs only.

## Step 1 — SkyrimNet database

Ask the user to upload their **SkyrimNet database** (`SkyrimNet-<numbers>.db`). Read its schema; confirm
these tables exist and their row counts. This DB feeds most of the **SkyrimNet tab**:

- **`memories`** — first-person memories. Columns: `content` (the memory text), `actor_uuid`, `location`,
  `game_time` (a float in-game clock), `importance_score` (~0.5–0.85), `emotion`, `memory_type`
  (`EXPERIENCE`, `RELATIONSHIP`, `KNOWLEDGE`, `TRAUMA`, `SKILL`), `tags` (JSON array), `related_event_ids`
  (JSON array).
- **`memory_embeddings`** — one row per memory: `memory_id`, `embedding` (raw vector bytes), `dimension`
  (e.g. 384). These power the "Constellation" scatter. **Decoding gotcha:** the value is served as TEXT,
  not BLOB, so set `con.text_factory = bytes` before reading — otherwise Python's sqlite3 tries to
  UTF-8-decode it and will either throw ("Could not decode to UTF-8") or report a wrong `LENGTH()`
  (character count of mangled multibyte text, not real byte count). Even with `text_factory = bytes`,
  SQLite's `LENGTH()` may still report a mangled character count — **`np.frombuffer(blob,
  dtype=np.float32)` on the raw column is authoritative**, and yields exactly `dimension` floats
  (`dimension × 4` bytes; 384 → 1536) of little-endian IEEE-754 float32, no header, no compression — see
  the projection recipe in `payloads.md`.
- **`events`** — the full event log (often 1000+ rows). Columns: `event_type`, `event_data` (JSON),
  `location`, `game_time`, and **`game_time_str`** — a human date like
  `"7:45 AM, Middas, 20th of Last Seed, 4E 201"`. This drives the day timeline and (via `death` events)
  the kill ledger. Also read `originating_actor_UUID` / `target_actor_UUID` and `playtime` where present.
- **`uuid_mappings`** — maps the big numeric `uuid` values to a readable `actor_name`. Use it to turn
  every `actor_uuid` / actor UUID into a name. **The player character's name comes from here** (the actor
  flagged as the player, e.g. bio template `player_special`).
- **`omnisight_screenshots`** — captured location/scene notes: `subject_name`, `description` (rich prose),
  `location_name` / `location_cell`, and a `capture_time_game` date. The actual screenshot pixels are
  **not** in the DB logs; use the text + metadata, plus any matching files in `omnisight-images/` when
  embedding the OmniSight cards.
- **`diary_entries`** — long-form first-person journal `content` with an `emotion` and `entry_date`.

Tell the user the character's name and the in-game date span you detected, then proceed.

The `text_factory = bytes` trick applies to any SkyrimNet DB you inspect with Python.

## Step 2 — IntelEngine database

Ask the user to upload their **IntelEngine database** (`IntelEngine-<numbers>.db`), the source for the
**IntelEngine tab**. Confirm these tables:

- **`faction_relations`** — pairwise standings: `faction_a`, `faction_b`, `relation_score` (−100…+100),
  `war_active`, `trade_active`.
- **`faction_wars`** — active/past wars: `faction_a`, `faction_b`, `battles_fought`, `faction_a_morale`,
  `faction_b_morale`, `faction_a_strength`, `faction_b_strength`, `victor`, `start_time`. **May be
  empty** — many saves never trigger a war at all.
- **`war_battles`** — `location_name`, `attacker`, `defender`, `result`, `attacker_losses`,
  `defender_losses`, `narrative`. Also empty when no war has broken out.
- **`faction_events`** — provocations/incidents: `faction_a`, `faction_b`, `event_type`
  (`war_declaration`, `border_skirmish`, `espionage`, `assassination_attempt`, `battle_result`, `tariff`,
  `trade_dispute`, `insult`), `description`, `relation_delta` (a signed integer), `game_time`.

If the user also has a small **`political_state.json`**, use it for **faction identity** — display names,
leaders, and holds — which give readable faction labels for the relations/chronicle sections. It also
mirrors the current war/relations state as a convenience, but the IntelEngine DB is authoritative for the
numbers.

Then **tell the user whether this save has a war** — i.e. whether there are any rows in `faction_wars` /
`war_battles`, or a `war_declaration` in `faction_events`. If there is no war, the war and battle
sections are simply left out; the rest of the IntelEngine tab still gets built.

## Step 3 — SkyrimNet logs

Ask the user for their **SkyrimNet logs**. Explain what each log offers so they can upload the right
one(s), and take only what you actually need rather than demanding all of them:

- **`openrouter_output.log`** (~1 MB — *recommended*) — the model's structured outputs. This is the only
  source for the **gossip** section, and a good source for diary/OmniSight text. Keep OmniSight images
  sourced from the local `omnisight-images/` files.
- **`conversation_log.log`** (~78 KB) — the raw player↔NPC / NPC↔NPC dialogue transcript (corroboration
  only).
- **`openrouter_input.log`** and its dated siblings (~4–13 MB each) — mostly the *prompts* sent to the
  model; large and largely redundant with the DBs. Usually **skip these**; they may be too big to upload.

These logs are **not** JSON-lines. Each entry is a header line followed by a body. Split on the header
pattern:

```
^\[.*\] \[.*\] (Generate request|Generate response|GenerateStreaming complete response) \[req_.*\]:
```

then JSON-parse the block that follows (guard for plain-text and ```json-fenced bodies; a body may be a
single object or an array of objects). **Discard** the startup health-check pings that read
`"Hello! Yes, I'm working"`. Expect a large fraction of blocks to be un-parseable request echoes or
streaming fragments — that's fine; keep the ones that parse.

Two families of objects matter:

1. **Gossip** — objects with `type:"npc_gossip"`:

```json
{ "type": "npc_gossip", "npc": "Arivanya", "npc2": "Ulundil",
  "narration": "leaned close and murmured about the Stormcloak uproar...",
  "gossip": "Ulfric called Thalmor justiciars parasites on Nord soil — loudly, in the square" }
```

   `npc` is the speaker, `npc2` the listener, `narration` the scene, `gossip` the rumor. (Some are
   `type:"npc_interaction"` with `fact1`/`fact2` instead — you may include those as memories exchanged.)

**If the user doesn't provide logs, don't invent gossip** — hide or annotate that section and tell them
it was skipped for lack of a log file.

## Step 4 — external assets (optional)

Beyond the databases and logs, the user may hand you **external assets** to use in the build — fonts
(`.ttf`/`.otf`/`.woff2`), images or textures (borders, panels, icons, backgrounds), or a batch of
OmniSight capture images. **When the user supplies an asset, use it directly** rather than recreating or
approximating it: inspect what they gave you and report what you found (dimensions, format, what it looks
suited for). There are **two ways to wire an asset in**:

- **Embed (default, for a single self-contained file):** inline the asset as a **base64 data URI** —
  fonts via `@font-face`, images via `url(data:...)` / `border-image` / `<img src="data:...">` — so the
  whole build is one file that opens offline.
- **Reference directly:** keep the asset as a real file and point the page at it by **relative path**.
  Deliver those asset files **alongside** the `.html` in an `assets/` folder when that is the intended
  packaging, so the page still opens offline from the local folder, just not as a lone file.

Keep a CSS/SVG fallback ready for anything the user *doesn't* supply, and tell them plainly which assets
you used (and how you wired them) and which you fell back on. Use **only** assets the user actually
provides through the chat — never fetch, download, or invent them. For the user's **own theme**, assets
are optional: ask only when they fit what was described, and don't demand them. For OmniSight image
embeds, ask first whether they want to upload the screenshots (specify **one archive**, not individual
images); if yes, place the provided image files in `omnisight-images/` and wire them into the cards.
