# Tribo Bahia Mirim Capoeira Practice Platform — Agent Implementation Brief

> Self-contained brief for whichever LLM picks up the work (Claude CLI, OpenCode/DeepSeek, OpenCode/BigModel, etc.). The spec PDF (`tribo mirim bahia capoeira spec.pdf`) is the canonical product definition. **This file is the operator's overlay** — paths, conventions, authorization, hand-off checkpoints. Read both before touching anything.

---

## TL;DR

Build a static-site capoeira practice tool at `capoeira.agroverse.shop` that pairs Bico Duro's individual-move video clips with curated berimbau-tempo music to drive 45-minute solo sessions, plus a public donation flow whose net proceeds (after Stripe fees) go to Tribo Bahia Mirim's after-school program in Itacaré. A separate transparency dashboard at `mirim-bahia.truesight.me` shows every dollar in/out from the existing TrueSight DAO `treasury-cache` ledger.

The project is **Shaolin-temple-shaped**: contemplative practice (`oracle.truesight.me`) and embodied practice (`capoeira.agroverse.shop`) are two halves of the same values-aligned worldview.

---

## Repository layout

| Repo | Path | Purpose |
|---|---|---|
| `tribomirimbahia` | `~/Applications/tribomirimbahia/` | **This repo.** Project container — spec, this brief, future planning docs, partner agreements, transparency-dashboard assets if separable. No production code. |
| `capoeira` | `~/Applications/capoeira/` | Static practice-platform site that deploys to `capoeira.agroverse.shop`. Currently empty — Phase 2 starts here. |
| `truesight_me` | `~/Applications/truesight_me/` | Existing static site. Transparency dashboard subdirectory `mirim-bahia/` lives here, deployed via the existing `truesight_me_prod` flow. |
| `agroverse_shop` | `~/Applications/agroverse_shop/` | Existing e-commerce site. Phase 4 cross-link from `farms/baia-itacare/` lands here. |

Raw video footage lives at `~/Downloads/capoeira/` (filesystem permission may need to be granted to the agent — `ls` returned `Operation not permitted` on first attempt).

---

## Required reading before doing anything

1. **The spec PDF** in this repo's root — every feature decision traces back to it.
2. **`agentic_ai_context/OPERATING_INSTRUCTIONS.md`** — workspace-wide rules.
3. **`agentic_ai_context/PROJECT_INDEX.md`** — repo-by-repo summary; lookup pattern for any project name you encounter.
4. **`agentic_ai_context/DAPP_PAGE_CONVENTIONS.md`** — meta tags, Open Graph, favicon, nav, body/container layout. **Mandatory** for every page in the `capoeira` site.
5. **`agentic_ai_context/DOWNLOADS_MEDIA_TO_AGROVERSE.md`** — heuristics for handling video drops in `~/Downloads/`. Phase 1 leans on this.
6. **`~/.claude/CLAUDE.md`** — git authorization scope (commit/push/PR/merge in `~/Applications` is pre-authorized; do NOT pause for confirmation in those repos).
7. **`agroverse_shop/scripts/sync_blog_listing_thumbnails.py`** and **`agroverse_shop/scripts/sync_post_open_graph_images.py`** — pattern for thumbnails / social images you'll mirror in Phase 2.
8. **`truesight_me/`** README + `treasury-cache/dao_offchain_treasury.json` (public on GitHub) — Phase 4 transparency dashboard reads from this ledger.

---

## Authorization (durable, no need to ask)

- **All `git` operations** in `~/Applications/` repos: commit, push, branch, PR, merge — pre-authorized by Gary in `~/.claude/CLAUDE.md`.
- **PRs default to draft** (per `truesight_autopilot#21`) and DO NOT auto-merge unless Gary types something equivalent to "merge it" / "go ahead and merge."
- **DAO contribution after each completed discrete task**: log via `dao_client/truesight_dao_client/modules/report_ai_agent_contribution.py` with `--contributors "Gary Teh"` (NOT auto-derived) and `--type "Time (Minutes)"`.
- **No `Co-Authored-By: Claude` trailer** in commit messages — AI attribution is covered by the DAO ledger.
- **Standing exceptions** (still ask first): force-push to main/master, `reset --hard` over uncommitted state, `--no-verify`, rewriting published shared-branch history.

---

## Model / tool hand-off matrix

Each pipeline stage names the **owner tool** so a future agent (or DeepSeek-driven session) knows where to stop and where to hand off. The capoeira project intentionally mixes models — cultural fidelity matters in some stages, cost matters in others.

| Stage | Owner | Reason for the choice |
|---|---|---|
| File enumeration + ffprobe metadata | shell (`ls`, `ffprobe`) | Deterministic, no model needed |
| Frame stills for vision/dedupe | shell (`ffmpeg`) + `imagehash` pHash | Deterministic; matches `analyze_incoming_videos.py` |
| **PT audio → text** (raw transcription) | `faster-whisper` via `agroverse_shop/scripts/analyze_incoming_videos.py --language pt` | Speech recognition is **not** an LLM task. Whisper handles PT natively. **Hand-off in:** raw clip files. **Hand-off out:** per-file Whisper segments in `manifest.json` |
| Transcript cleanup (filler removal) | local rules (`transcript_publish_helpers.clean_transcript` from agroverse_shop) | Rule-based, free |
| Transcript polish (optional) | Grok via `grok_transcript_polish.py` | Cheap, decent style; cached |
| Move-name extraction from PT intro | **Claude** | PT → canonical capoeira move name needs cultural literacy ("Bênção", "Meia-Lua de Compasso"); a wrong name on a public site is embarrassing |
| Vision pass on representative still | **Claude** | Confirms move identity if PT intro is ambiguous; visual capoeira-form recognition |
| Pedagogy notes (`notes` field on `moves.json`) | **Claude** (drafts only) | Heritage tone + Bico Duro framing; **Gary reviews every entry** before publish |
| **EN translation** of Bico Duro's PT speech | **Claude** | Lineage / mestre voice fidelity — not a literal translation, faithful rendering |
| BPM detection on music tracks | `librosa` (shell), or DeepSeek for batch tagging | Numeric task, no cultural nuance |
| Music tempo/style tags | **DeepSeek** | Cheap bulk tagging; Gary spot-checks |
| ffmpeg per-move clip trim + compile | shell `ffmpeg` (cut points from Whisper segment timestamps) | Deterministic |
| YouTube upload | reuse `agroverse_shop/scripts/upload_video_to_youtube.py` (OAuth bound to **admin@truesight.me**) | Don't re-mint credentials; reuse working OAuth |
| Title / description sync | mirror `youtube_update_video_titles.py` pattern with a tribomirimbahia-scoped JSON (`scripts/youtube_videos.json`) | Same source-of-truth idiom as agroverse_shop |
| Pull-request narrative + commit messages | **Claude** | Reviewer-readable; no Co-Authored-By trailer |

**Rule of thumb:** if a stage touches Bahia heritage, capoeira terminology, or Bico Duro's voice, route it through Claude. If it's bulk metadata munging, route through DeepSeek or local scripts.

---

## Phase 1 — Data Preparation (`tribomirimbahia/data/`)

### 1A — Catalog Bico Duro's move clips → `data/moves.json`

**Source layout (recorded chronologically by Gary; ~80 .MOV files in `~/Downloads/capoeira/`):**
- Each video typically opens with Bico Duro **announcing the move name in Portuguese**, then demonstrating it.
- A small number of videos are **not move demos** — at least one is Bico Duro speaking about the **history of capoeira / his training system / lineage**. Identify and route these separately (treated as bonus context videos for the landing page, not entries in `moves.json`).
- Some moves may have **multiple takes** across files — the "compiled clip" per move is the canonical edit (best take, trimmed dead air, optionally cross-faded). One compiled clip = one entry in `moves.json` = one YouTube upload.

**Pipeline (reuse agroverse_shop infrastructure rather than re-inventing):**

1. **Run analyze on the capoeira folder** (writes manifest with ffprobe + Whisper PT segments + pHash):
   ```bash
   cd ~/Applications/agroverse_shop
   python3 scripts/analyze_incoming_videos.py \
     --input ~/Downloads/capoeira \
     --output ~/Applications/tribomirimbahia/data/incoming_capoeira \
     --glob '*.MOV' \
     --language pt \
     --model small   # 'tiny' is faster; 'small' is more accurate for Portuguese
   ```
2. **Classify each video** (Claude): read `manifest.json`'s `transcript_segments[0..2]` per file. Three buckets:
   - `move_demo` — opens with a move-name announcement → goes into `moves.json` pipeline
   - `lineage_talk` — Bico Duro speaking about history/training/lineage → goes into a separate `data/lineage_videos.json` for the landing-page narrative
   - `unknown` — flag for Gary review
3. **Group `move_demo` videos by move name** — multiple takes of the same move get merged at the compile stage (1D below).

**Output:** `data/moves.json` per the spec's §3 schema. Every move object MUST have:
- `id` (snake_case, e.g. `ginga_basico`)
- `name_pt` (canonical Brazilian Portuguese, from the spoken intro — verify against capoeira-community standard spelling)
- `name_en` (one-line English description, NOT a translation — explain what the move IS)
- `theme` ∈ `Foundation | Defense | Attacks | Flow | Aerials | Floreios`
- `difficulty` ∈ `Beginner | Intermediate | Advanced`
- `duration_minutes` (typical practice time, 15–25)
- `youtube_clip_url` (filled in by 1E once uploaded)
- `tempo_range` ∈ `Slow | Medium | Fast`
- `notes` — pedagogy: precision focus, common mistakes, progression cues. **Honor Bico Duro's "precision over chaining" philosophy.**
- `pairs_well_with` — array of move IDs (optional)
- `source_videos` — array of source filenames (e.g. `["IMG_5283 2.MOV", "IMG_5284 2.MOV"]`) so re-edits are reproducible
- `transcript_pt` — verbatim PT intro (Bico Duro's own words)
- `transcript_en` — EN translation (Claude; lineage-faithful, not literal)

**Voice for `notes`:** terse, instructional, second-person. Reference body parts and ground angles, not abstract energy. Example:
> "Plant the back foot at 45°. Hip stays square — if you rotate, the kick loses power. Beginners over-extend the leg; keep it short and crisp until the foot retraction is automatic."

**Acceptance (hand-off back to Gary):** human reviewer reads each entry and corrects PT names + pedagogy notes. First-pass output is a *draft*, not the source of truth.

### 1B — Curate music tracks → `data/music_library.json`

12 tracks per the spec's §5 mix:
- 3–4 slow berimbau-heavy (Foundation / warm-up)
- 4–5 medium-tempo drum-heavy (Defense / Attacks)
- 2–3 fast energetic (Aerials / Floreios)

Each track: `id`, `title`, `youtube_url`, `duration_seconds`, `bpm` (estimated via `librosa` or DeepSeek tap), `tempo_category`, `style_notes`.

**Hand-off:** Gary curates the URLs; agent's job is the metadata + tagging pass. **Owner:** DeepSeek for tagging; Gary spot-checks.

### 1C — Compile per-move canonical clips → `data/compiled_clips/`

For each move identified in 1A, produce **one compiled clip** that goes to YouTube. Source files are in `~/Downloads/capoeira/` (~80 .MOV files; some moves have multiple takes).

**Process:**
1. **Trim points** from Whisper segment timestamps in `manifest.json`:
   - Start of demo = end timestamp of the PT-intro segment (where Bico Duro stops speaking and starts moving)
   - End of demo = end of last segment OR explicit silence detection via `ffmpeg -af silencedetect`
2. **Multi-take selection:** if a move has multiple source videos, choose the longest clean take by default; mark the others as `alternate_takes` in `moves.json` source list. **Do NOT cross-fade automatically** — capoeira demo videos read best as single takes; concatenation is a Gary-review decision.
3. **Compile command** (per move):
   ```bash
   ffmpeg -ss <start> -to <end> -i "<source>" \
     -c:v libx264 -preset slow -crf 20 \
     -c:a aac -b:a 128k \
     -movflags +faststart \
     "data/compiled_clips/<move_id>.mp4"
   ```
   `-movflags +faststart` matters for YouTube + web playback.
4. **Output naming:** `<move_id>.mp4` (snake_case). Example: `meia_lua_de_compasso.mp4`.
5. **Resolution + portrait/landscape:** preserve source aspect ratio; do NOT force-rotate. iPhone vertical videos stay vertical.

**Owner:** shell `ffmpeg` (deterministic). Whisper timestamps drive the cut points.

**Hand-off out:** one .mp4 per move under `data/compiled_clips/`, ready for 1D translation + 1E upload.

### 1D — English translation per compiled clip → `data/transcripts/`

For each compiled clip, produce two artifacts:
- `data/transcripts/<move_id>.pt.txt` — verbatim PT (from Whisper + `clean_transcript`)
- `data/transcripts/<move_id>.en.txt` — EN translation (Claude)

**EN translation rules (these are mestre's words on a public site):**
- Faithful, not literal. "Bênção" stays "Bênção" (gloss as "*the blessing kick*" parenthetically on first mention).
- Preserve cultural / lineage references — if Bico Duro mentions a teacher or capoeira regional vs. angola, keep those proper nouns.
- Second-person where Bico Duro is teaching ("you plant the foot…").
- No marketing-speak. No "transformative." No "journey."
- Footer line on the EN transcript: `*Translated from Bico Duro's instruction. PT source: tribomirimbahia/data/transcripts/<move_id>.pt.txt*`

**Owner:** Claude (drafts only). Gary reviews EN translation before any clip publishes publicly.

### 1E — Upload to YouTube (admin@truesight.me) → fill `youtube_clip_url`

**Reuse existing OAuth** in `agroverse_shop/scripts/`:
- `youtube_credentials.json` — Desktop client for admin@truesight.me
- `youtube_token.json` — refresh token with scopes `youtube.upload` + `youtube.force-ssl`

**Per-clip upload** (one move = one video):
```bash
cd ~/Applications/agroverse_shop
python3 scripts/upload_video_to_youtube.py \
  ~/Applications/tribomirimbahia/data/compiled_clips/<move_id>.mp4 \
  --title "<title-format-below>" \
  --description "<description-format-below>" \
  --privacy unlisted    # default unlisted until Gary approves public flip
```

**Title format (≤ 100 chars):**
```
<name_pt> — <name_en> | Tribo Bahia Mirim Capoeira
```
Example: `Meia-Lua de Compasso — half-moon compass kick | Tribo Bahia Mirim Capoeira`

**Description format:**
```
<one-line move summary in EN>

— Bico Duro, Tribo Bahia Mirim, Baia Itacaré (Bahia, Brazil)

INSTRUCTION (translated from Portuguese):
<EN transcript body>

ORIGINAL (Portuguese):
<PT transcript body>

About Tribo Bahia Mirim:
After-school capoeira program for children in Baia Itacaré, led by Bico Duro. Support: https://capoeira.agroverse.shop

Lineage: <if Bico Duro mentioned in the intro, repeat the lineage line he used>
```

**Source-of-truth JSON** (mirrors agroverse_shop convention): `tribomirimbahia/scripts/youtube_videos.json` maps `<move_id>` → `{video_id, canonical_title, description_sha}`. After every upload OR title change, run a `youtube_update_video_titles.py`-style script so the live YouTube title matches the JSON.

**Privacy default:** **unlisted** until Gary reviews. Flip to **public** only on Gary's explicit say-so per move (cultural risk: a wrong PT name on a public video is hard to walk back).

**Owner:** existing `upload_video_to_youtube.py` (no new OAuth). **Hand-off back to Gary:** YouTube URLs for each move, listed in `youtube_videos.json` + populated into `moves.json[*].youtube_clip_url`.

### 1F — Bonus lineage / history video → `data/lineage_videos.json`

Any video classified as `lineage_talk` in 1A.2 (Bico Duro speaking about capoeira history or his training system, not demoing a move) gets a separate pipeline:
- Same Whisper PT transcript + Claude EN translation
- Same YouTube upload (unlisted, admin@truesight.me)
- Recorded in `data/lineage_videos.json` (not `moves.json` — these are landing-page narrative material, not practice content)
- Title format: `<topic in PT> — <topic in EN> | Tribo Bahia Mirim Capoeira` (e.g. `A história da capoeira — the history of capoeira | Tribo Bahia Mirim Capoeira`)

These videos feed Phase 2's `index.html` narrative section.

---

## Phase 2 — Core Site Build (`~/Applications/capoeira/`)

The `capoeira` repo is currently empty. Everything below is greenfield, but **mirror `agroverse_shop` conventions** rather than inventing new ones.

### Files to create

```
capoeira/
├── index.html                  # Landing page (§4 narrative, donate CTA)
├── library.html                # Move library (searchable/filterable per §4.4)
├── practice.html               # Session generator + practice flow (§4.1, §4.2)
├── transparency.html           # Light summary; full dashboard at mirim-bahia.truesight.me
├── assets/
│   ├── css/styles.css
│   ├── js/session-generator.js # §4.1 logic (history-aware theme selection, weighted move pick)
│   ├── js/practice-flow.js     # §4.2 (move display, music timer, "Rest" prompt, next-move handoff)
│   ├── js/move-library.js      # §4.4 filters
│   ├── js/session-history.js   # §4.3 localStorage persistence + streak counter
│   └── images/                 # Hero + thumbnails
├── data/                       # Symlink or copy of tribomirimbahia/data/{moves,music_library}.json
└── README.md                   # How to develop locally + deploy
```

### Conventions (mandatory)

- Every page MUST follow `agentic_ai_context/DAPP_PAGE_CONVENTIONS.md` for meta / OG / favicon / nav / body container. Do NOT skip these — they're what makes the page render correctly when shared on WhatsApp / Telegram / etc.
- For UX patterns (loading, errors, comboboxes) follow `dapp/UX_CONVENTIONS.md`.
- No frontend frameworks. Static HTML/CSS/vanilla JS. The spec says so (§8) and the rest of the agroverse_shop family follows the same convention.
- Video embeds: YouTube unlisted/public; use `youtube-nocookie.com` for the privacy-preserving embed.
- Music: also YouTube; honor the move's `tempo_range` when sequencing.

### Session Generator logic (`assets/js/session-generator.js`)

Algorithm per §4.1:
1. Load last 3-4 sessions from `localStorage.session_history`.
2. Pick a theme that wasn't used in the last 2 sessions (avoid repetition).
3. From that theme's moves, pick 4-6 weighted by:
   - Difficulty bias toward Beginner/Intermediate (configurable via UI later)
   - Total `duration_minutes` summing to ~45 (±10)
   - Recency penalty: a move practiced in the last session has lower selection weight
4. Pick music tracks matching the theme's tempo arc: slow intro → medium-fast middle → optional cool-down. Sequence by `tempo_category`.
5. Render a "Session Card" with theme, ordered move list, ordered music list, total estimated time.

### Practice Flow logic (`assets/js/practice-flow.js`)

Per §4.2:
- Display first move: `name_pt` (large) / `name_en` (subtitle), embedded YouTube clip, `notes`, music-track countdown timer.
- "Play Music" button starts the YouTube music embed AND starts the timer.
- When music ends → "Rest" prompt with 30s countdown → next move auto-loads.
- Final move ends → "Log session" CTA writes to `localStorage`.

---

## Phase 3 — Persistence & Analytics (optional MVP, ship in Phase 2 if time allows)

`assets/js/session-history.js`:
- Sessions completed this week (rolling 7d)
- Theme frequency breakdown (count per theme over last 30d)
- Most/least practiced moves (top 5 / bottom 5 by count)
- Streak counter (consecutive days with ≥1 session)
- All client-side, `localStorage` only. No backend.

If a backend becomes necessary later, the schema is per spec §3 (`session_history.json`) — leave the door open by keeping the same field names.

---

## Phase 4 — Donation & Ledger Integration

### 4A — Stripe checkout (in `capoeira/index.html` + footer)

- Stripe Checkout (hosted page), not Stripe Elements. Lower implementation surface, less PCI scope.
- Donation amounts: preset $10 / $25 / $50 / $100 + custom field.
- Success URL → `transparency.html?just_donated=1` which renders a thank-you message + link to the dashboard.
- Webhook is **optional** for MVP (per spec §8) — manual reconciliation is acceptable initially.

### 4B — Manual monthly reconciliation

Per spec §4.5 + §9:
1. Stripe CSV export → 2. Google Sheet (Tribo Bahia Mirim Donations tab on the existing TrueSight DAO ledger spreadsheet) → 3. Apps Script writes JSON to `treasury-cache` repo.

Mirror the existing pattern in `tokenomics/google_app_scripts/` (sales/inventory snapshot scripts publish to `treasury-cache` similarly). Add a new script: `tribomirimbahia_donations_snapshot.gs`.

### 4C — Transparency dashboard at `mirim-bahia.truesight.me`

Lives in `truesight_me/mirim-bahia/index.html`. Pulls from `https://raw.githubusercontent.com/TrueSightDAO/treasury-cache/main/tribomirimbahia_donations.json`.

Shows monthly table:
| Month | Total inflow (USD) | Stripe fees | Net to program | Pix transfer ref |

Plus a single sentence above the table: "Every donation, minus processing fees, goes directly to Bico Duro and the kids."

---

## Conventions

- **Tone:** humble, specific, never marketing-speak. Don't write "transformative" or "journey." Do write "45-minute session, six moves, one berimbau."
- **Portuguese fidelity:** `Bahia` (proper noun, capitalized), `mirim` (lowercase except in proper nouns like "Tribo Mirim Bahia"), `capoeira` (lowercase). Move names in `data/moves.json` use lowercase Portuguese spelling (e.g. `meia-lua de compasso`, not `Meia-Lua De Compasso`).
- **Bico Duro framing:** he's the lineage holder + teacher, not a "founder" or "celebrity." Public copy treats him with the deference owed to a mestre.
- **Gary framing:** "long-time supporter" / "student of the lineage," NOT "founder of Tribo" or "leader of the program." Per memory: never use "founding member" — use "long time contributor" instead.
- **Cross-link from Agroverse:** `agroverse_shop/farms/baia-itacare/index.html` gets a single line linking to `capoeira.agroverse.shop` per spec §10. Don't redesign the farm page.

---

## What NOT to do

- **Don't invent move names.** If you can't identify a move from the video, write `name_pt: "TODO_REVIEW"` and let Gary fill it in. A wrong PT name on a public site is embarrassing in the capoeira community.
- **Don't add features beyond the spec.** No leaderboards, no user accounts, no AI-generated pedagogy notes (Bico Duro's notes are the source of truth; LLMs draft them, Gary reviews).
- **Don't touch `truesight_autopilot` or `dapp/` for this project.** Capoeira platform is its own surface.
- **Don't write copy that implies TrueSight DAO takes a cut.** It doesn't (spec §9). 100% of donations net of Stripe fees go to Bico Duro / Tribo.
- **Don't ship without `DAPP_PAGE_CONVENTIONS.md` compliance.** Pages without proper meta / OG tags break sharing.
- **Don't push Bahia-specific cultural copy without Gary's review.** This is the highest-stakes part of the project — heritage is unforgiving of clumsy phrasing.

---

## Hand-off checkpoints (where the human reviews)

| Checkpoint | What Gary reviews | Why |
|---|---|---|
| End of Phase 1A | Every entry in `data/moves.json` (PT names + pedagogy notes) | Cultural fidelity; LLM first-pass is a draft |
| End of Phase 1B | Music track tags (BPM, tempo_category) | BPM detection can drift; tempo arc affects practice feel |
| Landing page first draft | All copy on `index.html` | Heritage tone, Bico Duro framing |
| Pre-Stripe-live | Stripe account config, fee math | Real money; double-check before going live |
| Pre-launch | Full site walkthrough on a phone | Practice flow needs to feel right on mobile |

---

## Output expected per phase

Each phase ends with a short status message to Gary in this format:

```
phase: <1A|1B|2|3|4A|4B|4C>
artifacts: <files added or changed>
PRs: <URLs, draft status>
verified: <what you ran/tested>
needs review: <specific human checkpoints>
follow-ups: <any open items appended to agentic_ai_context/OPEN_FOLLOWUPS.md>
```

No prose celebration. Gary reads diffs.

---

## DAO contribution logging (after each phase)

Run from any directory:
```bash
python3 -m truesight_dao_client.modules.report_ai_agent_contribution \
  --title "Capoeira platform — Phase <N>: <one-line summary>" \
  --body-file /tmp/phase_N_body.txt \
  --pr <PR URL> \
  --type "Time (Minutes)" \
  --minutes <rounded to nearest 30> \
  --contributors "Gary Teh"
```

Reference: `agentic_ai_context/DAO_CLIENT_AI_AGENT_CONTRIBUTIONS.md`.

---

## References

- Spec: `tribo mirim bahia capoeira spec.pdf` (this repo's root)
- DApp page conventions: `agentic_ai_context/DAPP_PAGE_CONVENTIONS.md`
- Downloads pipeline: `agentic_ai_context/DOWNLOADS_MEDIA_TO_AGROVERSE.md`
- Treasury cache pattern: `tokenomics/google_app_scripts/` for snapshot scripts; `treasury-cache` repo for the published JSON
- Existing Agroverse e-commerce site (convention source): `agroverse_shop/`
- DAO contribution flow: `agentic_ai_context/DAO_CLIENT_AI_AGENT_CONTRIBUTIONS.md`
- Operating instructions: `agentic_ai_context/OPERATING_INSTRUCTIONS.md`
- Standing git/script authorizations: `~/.claude/CLAUDE.md`

---

## Open questions (answer before Phase 2)

1. **Site deployment target:** does `capoeira.agroverse.shop` deploy from the `capoeira` repo via GitHub Pages, OR is the site copied into `agroverse_shop/capoeira/` and deployed via the existing Agroverse Pages flow? Default assumption: standalone repo with GitHub Pages, custom domain via DNS CNAME.
2. **Stripe account:** new Stripe account for Tribo, or sub-account / restricted-key under Gary's existing Stripe? Affects KYC + fee structure + reporting flow.
3. **Bico Duro consent:** does he know his image / footage is going public? Spec §11 step 1 says to confirm. Phase 1A clip cataloging can proceed in parallel but **don't push public until consent is recorded.**
4. **Initial donation goal:** is there a fundraising target to display ("first $X funds quarter 1 of after-school program")? Affects landing page copy.
