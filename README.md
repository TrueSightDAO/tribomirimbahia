# Tribo Bahia Mirim — Ledger & Transparency Layer

Static transparency explorer at [mirim-bahia.truesight.me](https://mirim-bahia.truesight.me),
plus the project's spec PDF, agent brief, and the workflow for publishing the donation
ledger to the public `treasury-cache` repo.

## What

Two things live here:

1. **`index.html`** — the transparency explorer. Reads `treasury-cache/managed-ledgers/tribomirimbahia.json`
   and renders the monthly donation table: inflow, Stripe fees, net to program, Pix transfer ref.
   Deploys via GitHub Pages from this repo (`CNAME` → `mirim-bahia.truesight.me`).

2. **`CLAUDE_PROMPT_LEDGER.md`** — the workflow for publishing/updating the donation JSON to
   [`TrueSightDAO/treasury-cache`](https://github.com/TrueSightDAO/treasury-cache) so the
   explorer has data to read.

**Practice content (move clips, music, the session-generator site) lives in a separate repo:**
[`TrueSightDAO/capoeira`](https://github.com/TrueSightDAO/capoeira), deploys to
[`capoeira.agroverse.shop`](https://capoeira.agroverse.shop). Don't put practice content
here — the split is intentional:

- **`tribomirimbahia/`** = ledger & transparency (this repo)
- **`capoeira/`** = practice tool

## Develop

Open `index.html` directly in a browser, or:

```bash
python3 -m http.server 8000
# → http://localhost:8000
```

The page fetches from `https://raw.githubusercontent.com/TrueSightDAO/treasury-cache/main/managed-ledgers/tribomirimbahia.json`.
If you want to test against local ledger data, swap the `DATA_URL` in `index.html`.

## Files

```
├── index.html                          # Transparency explorer (deploys via CNAME)
├── CNAME                               # mirim-bahia.truesight.me
├── CLAUDE_PROMPT_LEDGER.md             # Workflow: publish donation JSON to treasury-cache
├── AGENT_BRIEF.md                      # Operator overlay — Phase 4 (donations/ledger) focused
├── tribo mirim bahia capoeira spec.pdf # Canonical project spec (covers both repos)
└── README.md
```

## Deploy

Deploys via GitHub Pages from the `main` branch root, custom domain via `CNAME`.

| Setting | Value |
|---|---|
| GitHub Pages source | Deploy from a branch |
| Branch | `main` / `(root)` |
| Custom domain | `mirim-bahia.truesight.me` (see `CNAME`) |

## Data flow

```
Donor → Stripe Checkout (linked from capoeira.agroverse.shop landing)
      ↓
   Stripe CSV export (monthly)
      ↓
   Google Sheet "Tribo Bahia Mirim Donations" (manual reconciliation)
      ↓
   Apps Script → treasury-cache/managed-ledgers/tribomirimbahia.json
      ↓
   index.html (this repo) reads the JSON and renders the table
```

Schema for the JSON is in `CLAUDE_PROMPT_LEDGER.md`.

## Related repos

| Repo | Purpose | Deploys to |
|---|---|---|
| [`tribomirimbahia`](https://github.com/TrueSightDAO/tribomirimbahia) | **This repo** — ledger & transparency layer | `mirim-bahia.truesight.me` |
| [`capoeira`](https://github.com/TrueSightDAO/capoeira) | Practice platform — moves, music, session generator | `capoeira.agroverse.shop` |
| [`treasury-cache`](https://github.com/TrueSightDAO/treasury-cache) | Public JSON ledger (`managed-ledgers/tribomirimbahia.json`) | (raw GitHub) |
| [`tokenomics`](https://github.com/TrueSightDAO/tokenomics) | Apps Script reference for Sheet → treasury-cache snapshot pattern | — |

## Conventions

- **Transparency credibility is the bar.** Donors should be able to trace every dollar. Keep numbers + math + Pix refs visible and auditable.
- **No marketing language.** "Net to program after fees" — not "100% goes to the kids" or "transformative."
- **Bico Duro framing:** mestre / lineage holder; never "founder" or "celebrity."
- **Gary framing:** "long-time supporter" — never "founder of Tribo," never "founding member."

## License

No license yet. Contact Gary for permissions.

Phase 4 (donations + transparency) — see `AGENT_BRIEF.md` for open questions and operator overlay.
