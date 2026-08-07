# PR to SELAT-AI/selat-skills — ready to open

**Branch:** `add-skill-destination-brief`
**Base:** `SELAT-AI/selat-skills` `main`
**Files:** `skills/destination-brief/` (4 files) + `index.json`
**Title:** `Add skill: destination-brief`

The `.selat/verify-receipt.json` is gitignored upstream by design
(`.gitignore:8 skills/*/.selat/`) — the receipt belongs in the PR body, below.

---

## PR body

Adds the **destination-brief** skill (multi/routed).

Trip-planning brief for any destination — fuses Tripadvisor's location index (POIs, ratings,
addresses), a live web-context search, and a Google SERP into one grounded pre-travel brief.
Three routed calls, ~$0.04 total, no API keys.

The hub's existing 18 skills cover enrichment, social, financial and research; none cover travel
planning, so this fills a gap rather than overlapping.

### Steps

| # | Step | Provider | Method | Rail | Price |
|---|---|---|---|---|---|
| 1 | points of interest | Tripadvisor (paysponge x402 gateway) | `GET /api/v1/location/search` | routed-x402 | $0.0105 |
| 2 | web context | Tavily (first-party x402) | `POST /search` | routed-x402 | $0.0105 |
| 3 | SERP | SerpApi (Tempo MPP gateway) | `GET /search` | routed-mpp | $0.01575 |

Full run ≈ $0.037. `maxAmount` `0.09` (per-step `0.03`) — headroom, not a price.

### Live verification (selat-verify/v1 · 2026-08-07T06:22:50.290Z · probe-only)

- step 1 [routed] routed-x402 $0.0105 / cap $0.03
- step 2 [routed] routed-x402 $0.0105 / cap $0.03
- step 3 [routed] routed-mpp $0.01575 / cap $0.03

`selat skill validate` ✓ · `selat skill verify` ✓ 3/3 within cap · `npm run validate` ✓ 19 skills,
0 errors, 0 warnings.

### Notes for reviewers

All three endpoints were exercised with **real settled calls** while authoring, not only probed,
so `references/endpoints.md` records live quote IDs, observed prices and actual request shapes
rather than catalogue claims. Two things surfaced that are worth flagging:

- **Tavily's declared `inputSchema.required` is `[]`, but the live API rejects a request without
  `query` — and still charges for it.** Confirmed by paying $0.0105 for a `400 Validation failed`
  (`quoteId=selatx202a7469-4800-464e-baa3-6e560ba6c471`). Documented as a gotcha in `SKILL.md`.
  This is the spec-drift case `meta/skill-creator/references/schema-enrichment.md` warns about.
- **Charged prices ran ~5% over the catalogue's listed `minAmountUsd` on both rails**
  (`$0.0150` → `$0.015750`), so `maxAmount` is set with headroom rather than at the listed figure.

Not paid-verified end-to-end — a maintainer should paid-re-verify before merge.

---

## How to open it

`SELAT-AI/selat-skills` is outside this session's repo scope, so the fork has to be created
from the GitHub UI first:

1. Fork https://github.com/SELAT-AI/selat-skills → `vickman787/selat-skills`
2. Push this branch to the fork:
   ```bash
   cd /workspace/selat-ai/selat-skills
   git remote add fork https://github.com/vickman787/selat-skills
   git push -u fork add-skill-destination-brief
   ```
3. Open the PR: https://github.com/SELAT-AI/selat-skills/compare/main...vickman787:add-skill-destination-brief
   — paste the body above.

Or, once the fork exists, `selat skill submit ./skills/destination-brief` does 2 and 3 in one
step (it needs a passing verify receipt, which is already written).
