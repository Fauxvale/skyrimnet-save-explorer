# SkyrimNet Save Explorer

**🔎 [Live example — "The Adventures of Jules"](https://fauxvale.github.io/The-Adventures-of-Jules.html)** — see a full explorer built from a real playthrough.

A [Claude Code](https://claude.com/claude-code) skill that turns a Skyrim playthrough using the
**SkyrimNet** and **IntelEngine** AI mods into a single, self-contained, offline HTML document — an
*illuminated field journal* for one character's entire run, transcribed straight from your own save data.

Everything on the page is derived from the files you provide. Nothing is templated or hardcoded, and
your databases are never modified — only read.

## What it produces

One double-clickable `.html` file with **two tabs**:

### SkyrimNet — the lived experience
- **Event timeline** — a chronological record of the character's run
- **Memory constellation** — an interactive visualization of the character's memories, laid out from
  their embedding projections
- **Memories** — the memories themselves
- **Kill ledger** — combat and death records
- **Diary** — the character's diary entries
- **OmniSight field notes** — a distilled "portrait of the land" plus matched capture images

### IntelEngine — the political layer
- **Faction relationships** — how the world's factions regard one another
- **Provocation chronicle** — the history of provocation events
- **Off-screen gossip** — non-visible gossip records
- **War & battles** — an ongoing war and its battles, shown **only when the save actually has one**

### Throughout
- A single offline HTML file — no build step, no frameworks, no external libraries
- Collapsible sections and a floating table of contents that reflows into a mobile bar on narrow screens
- Optional embedding of your OmniSight screenshots

## Requirements

- **Claude Code** installed
- A Skyrim playthrough with both the **SkyrimNet** and **IntelEngine** mods
- Your character's save data:
  - `SkyrimNet-<numbers>.db` — source for most of the SkyrimNet tab
  - `IntelEngine-<numbers>.db` — source for the IntelEngine tab
  - `openrouter_output.log` (recommended) — the only source for gossip
  - Optionally, OmniSight screenshots and any custom fonts/images

No additional dependencies or build tools are needed.

## Installation

1. Clone or download this repository.
2. Copy `skills/skyrimnet-save-explorer` into your Claude Code skills directory:
   - **macOS / Linux:** `~/.claude/skills/`
   - **Windows:** `%USERPROFILE%\.claude\skills\`
3. Restart Claude Code so the skill is auto-loaded.
4. Invoke it with a prompt like *"build me a SkyrimNet save explorer"*, or run
   `/skyrimnet-save-explorer`.

## How it works

The workflow is collaborative — Claude builds the page with you, not from a template:

1. **Share your files.** Upload your SkyrimNet and IntelEngine databases and logs; Claude inspects them
   read-only, one file at a time, and reports what it found (row counts, character name, date range).
2. **Make the calls that are yours to make.** Claude asks about the visual theme (an "Illuminated
   Manuscript" preset or your own), any optional sections for notable events it discovers, whether to
   include an opt-in "Total Spend" stat, and whether to embed OmniSight screenshots. It also detects and
   tells you whether the save has a war.
3. **Get one self-contained page.** Claude generates a single offline HTML file, verifies every section
   populates and every interaction works, and hands it over ready to open and share.

## Design principles

- **Everything from your data** — the character's name, in-game day count, every stat, label, and count
  come from your uploaded files, never from an example.
- **Read-only** — your databases are inspected with `SELECT`s only and are never modified.
- **Portable and shareable** — a single, framework-free HTML file that opens offline by double-click.
