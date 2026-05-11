## Claude prompt — Tribo Bahia Mirim Capoeira Donation Ledger publishing

When you are ready to publish the Tribo Bahia Mirim donation ledger data, write it to this file:

```
treasury-cache/managed-ledgers/TBM.json
```

The repo is `TrueSightDAO/treasury-cache` (HTTPS remote, already cloned at `~/Applications/treasury-cache/`).

### Schema to follow

The file uses this shape (the Transparency Explorer at `https://TrueSightDAO.github.io/tribomirimbahia/` reads from it):

```json
{
  "ledger_name": "TBM",
  "program_name": "Tribo Bahia Mirim Capoeira",
  "description": "Donation ledger for Tribo Bahia Mirim's after-school capoeira program in Itacare, Bahia.",
  "schema_version": 1,
  "generated_at": "<ISO timestamp when this snapshot was generated>",
  "source": "<what produced this: e.g. Stripe-reconciliation-apps-script, manual, claude-agent>",
  "summary": {
    "total_donations_usd": 123.45,
    "total_stripe_fees_usd": 12.34,
    "total_transferwise_fees_usd": 5.67,
    "total_net_to_program_brl": 500.00,
    "transaction_count": 5
  },
  "transactions": [
    {
      "id": "stripe_ch_123",
      "date": "2026-05-10T14:30:00Z",
      "type": "stripe_donation",
      "description": "Donation via capoeira.agroverse.shop",
      "amount": 50.00,
      "fees": 1.75,
      "net": 48.25,
      "currency": "USD",
      "reference": "stripe_ch_123",
      "status": "completed"
    },
    {
      "id": "tw_transfer_456",
      "date": "2026-05-10T15:00:00Z",
      "type": "transferwise_fx",
      "description": "USD → BRL conversion, rate 5.12",
      "amount": 48.25,
      "fees": 3.50,
      "net": 229.12,
      "currency": "BRL",
      "reference": "tw_transfer_456",
      "status": "completed"
    },
    {
      "id": "pix_e2e_789",
      "date": "2026-05-10T16:00:00Z",
      "type": "pix_outbound",
      "description": "Monthly disbursement to program",
      "amount": 229.12,
      "fees": 0,
      "net": 229.12,
      "currency": "BRL",
      "reference": "pix_e2e_789",
      "recipient": "Bico Duro / Tribo Bahia Mirim",
      "status": "completed"
    }
  ]
}
```

### How to publish

**If you are pushing directly (Claude agent commit):**

```bash
cd ~/Applications/treasury-cache
git pull origin main
# Edit managed-ledgers/TBM.json
git add managed-ledgers/TBM.json
git commit -m "chore: update tribomirimbahia donation ledger snapshot"
git push origin main
```

**If an Apps Script is publishing (the long-term plan):**

The Apps Script should:
1. Pull from Google Sheet (Stripe CSV → Sheet tab)
2. Construct the JSON per the schema above
3. PUT to GitHub Contents API: `TrueSightDAO/treasury-cache/managed-ledgers/TBM.json`
4. The explorer auto-fetches via `raw.githubusercontent.com`

### The explorer page

Once the JSON is published, the transparency dashboard at `https://TrueSightDAO.github.io/tribomirimbahia/` (or `tribomirimbahia.truesight.me` when CNAME is active) will display:
- Summary cards (total inflow, fees, net to program)
- Filterable transaction table (search + type dropdown)
- Each transaction row: date, type badge, description, amounts, reference, status

The explorer normalizes field names flexibly — if the schema evolves, it will still render. The canonical fields above are the preferred shape.

### When there are no transactions yet

The file at `managed-ledgers/TBM.json` already exists with an empty `transactions` array and zeroed `summary`. The explorer shows "Ledger coming soon" until the first transaction is added. Simply append to the array and update `summary` and `generated_at`.
