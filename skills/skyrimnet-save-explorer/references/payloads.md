# Phase 2 — Transform the data

Build the JSON payloads to embed in the page. Derive everything from the user's files.

## SkyrimNet payload (`sndata`)

- **`player`** — the player character's name (from `uuid_mappings`).
- **`stats`** — headline numbers for the header: total events, total memories, number of distinct minds
  (actors that have memories), OmniSight captures, in-game day count, and a one-line subtitle (character
  name + the in-game date span, and playtime hours if derivable from `playtime`).
  **Total Spend is opt-in and user-supplied:** the header can include a "Total Spend" stat — the
  real-world money the user spent running this playthrough (API/LLM calls and the like), **not** any
  in-game gold. There is no way to derive this from the files. Before finalizing the header, **ask the
  user whether they'd like to include a Total Spend statistic**. If yes, **literally ask how much (in
  real dollars) they've spent** on API calls etc. for this run, and use the figure they give you
  **verbatim** (whatever they type, formatted as they type it). If no, skip the stat entirely.
- **`events_by_day`** — a map keyed `"<in-game day label>|<event_type>" -> count`. Parse the day/date out
  of `events.game_time_str` (e.g. `"20th of Last Seed"`), and count events per `(day, event_type)`. Also
  emit an ordered list of day labels. The page buckets `event_type`s into five categories — build the
  same mapping:
  - **Dialogue** (gold): `dialogue`, `dialogue_background`, `dialogue_npc`, `dialogue_player`,
    `dialogue_player_stt`, `dialogue_player_text`, `gamemaster_dialogue`
  - **Death & combat** (blood red): `death`
  - **Thoughts & narration** (steel blue): `player_thoughts`, `direct_narration`, `diary_entry_created`
  - **Trade, rest, world** (moss green): `trade_start`, `trade_complete`, `sleep_start`, `sleep_stop`,
    `book_read`, `furniture_used`, `item_given`
  - **Followers & tasks** (grey): anything else
- **`memories`** — an array; for each memory: `content`, `actor` (resolved name via `uuid_mappings`),
  `emotion`, `location`, `imp` (= `importance_score`), `type` (= `memory_type`), `day` (an in-game day
  label from `game_time`/related event), `gt` (raw `game_time`, for chronological sorting), and 2-D
  coordinates `x, y` (PCA) and `ux, uy` (UMAP) — see the projection recipe below.
- **`edges`** — an array of `[i, j, similarity]`: for each memory, its **2 nearest neighbors** by cosine
  similarity in the full embedding space, with the similarity value.
- **`leaderboard`** — kills aggregated per killer, ranked by total. Each entry:
  `{ killer, total, cats: { <category>: count }, victims: [[victim_name, count], …] }`. Parse `death`
  events (killer/victim from `event_data`, falling back to `originating_actor_UUID` /
  `target_actor_UUID`), resolve names via `uuid_mappings`, and bucket each victim into a small fixed set
  of categories — **People, Undead, Wildlife, Monsters, Dragon** — for the stacked bar.
- **`diary`** — the diary as `{ actor, loc, content, emotion }` (an array if there are several entries).
  Keep `content`'s paragraph breaks and any `*emphasis*` markers intact.
- **`screenshots`** — OmniSight captures as `{ name, loc, ts, desc }` from `omnisight_screenshots`
  (`ts` = the capture date/time string; `loc` = readable location/cell). Match each capture to the
  corresponding local image in `omnisight-images/` and embed that image in the card.
- **`omni_summary`** — **a synthesized "portrait of the land."** Read **all** of the OmniSight
  descriptions and distil them into one cohesive prose description of the whole landscape the run passed
  through: a `{ title, paragraphs:[…2-3 paragraphs…], foot }` object. Ground it strictly in what the
  captures actually describe — the real named places, terrain, structures, wildlife, weather, and mood
  (do a quick frequency pass over the text to find the recurring places and atmosphere) — and **never
  invent** locations or events that aren't in the captures. Make `title` something like `"A Portrait of
  the Land"` and `foot` a derived line like `"Distilled from all <N> OmniSight captures — <first day> to
  <last day>, 4E 201."`

**Actor colours:** rank actors by memory count; the **top ~5 get distinct colours**, everyone else is
grey. Reuse this mapping everywhere an actor is coloured (constellation dots, memory rows, leaderboard
names).

## Computing the projection (`x,y` / `ux,uy`) and edges

Do all of this **offline in Python**; the browser only ever receives coordinates, edge indices, and
similarity weights, never the raw vectors.

1. **Decode.** Open the DB with `con.text_factory = bytes`. Join `memory_embeddings` to `memories` on
   `memory_id` and decode each embedding with `np.frombuffer(blob, dtype=np.float32)`. Expect
   `len(vec) == dimension` (384). If lengths are `dimension × 2` bytes it's float16; anything
   smaller/irregular suggests a compressed store — but standard SkyrimNet DBs are plain float32. Stack
   into an `N × dimension` matrix **in memories order**, so row *i* is memory *i* everywhere downstream
   (tooltips, filters, and edges all index this one shared array).
2. **PCA → `x,y`.** Mean-center the matrix and take the top-2 principal components via SVD (top two
   singular vectors scaled by their singular values). Report the fraction of variance those two
   components capture (sum of top-2 squared singular values / total) in the section caption — usually
   ~20%, an honest "blurry linear shadow" of the space.
3. **UMAP → `ux,uy`.** Run UMAP on the same matrix with **`metric='cosine'`**, `n_neighbors=10`,
   `min_dist=0.15`, and a fixed `random_state` for a reproducible layout. Cosine is deliberate — it's the
   metric the mod's memory recall actually uses. If `umap-learn` isn't installed, try to `pip install`
   it; failing that fall back to cosine t-SNE (sklearn); failing that reuse the PCA coords for `ux,uy`.
   **Tell the user plainly which you used.** Normalise both projections into a comparable 0–1 range.
4. **Edges (computed in the full space, not the projection).** L2-normalize every row, compute the full
   cosine-similarity matrix (`Xn @ Xn.T`), and for each memory keep its top-2 most-similar neighbors
   above a similarity floor (~0.75). Dedupe symmetric pairs into an undirected list; ship each as
   `[index_a, index_b, similarity]`, with similarity driving line opacity. Because edges are truth from
   the 384-d space while positions are projected, long edges in PCA are exactly the neighbor pairs the
   linear projection tore apart — a built-in diagnostic.
5. **Serialize.** Round coordinates to ~4 decimals and embed as JSON. When writing into the
   `<script type="application/json">` tag, escape `</` as `<\/` so no memory text can accidentally close
   the script block.

## IntelEngine payload (`inteldata`)

- **`relations`** — from `faction_relations`: each pair with its signed `relation_score`, whether a war
  is active, and readable faction display names (via `political_state.json` if present, else
  de-camelCased).
- **`factions`** — from `political_state.json` if present: `{ name, leaders, hold }` for each, used for
  readable labels, hover detail, faction colours, and whisper classification.
- **`war`** — from `faction_wars`, **only if a war exists**: the belligerents, `battles_fought`, the
  war's start, both sides' morale & strength, victor status, each side's leaders, and who declared it
  (from the `war_declaration` `faction_event`). Omit this key entirely when there's no war.
- **`battles`** — from `war_battles`: location, attacker/defender, result, losses on each side,
  narrative. Empty or omitted when there's no war.
- **`events`** — from `faction_events`: the provocation chronicle — faction pair, `event_type`,
  `description`, signed `relation_delta`, ordered by `game_time`.

## Gossip payload (`gossipdata`)

An array of `{ speaker, listener, rumor, scene }` from the successful `npc_gossip` records in the output
log (`speaker`=`npc`, `listener`=`npc2`, `scene`=`narration`, `rumor`=`gossip`). Keep only **distinct**
exchanges (dedupe repeats). Empty if no log was provided. Displayed gossip must come **only** from these
saved outputs — the dialogue/input logs are corroborating provenance, not a source of new rumors.
