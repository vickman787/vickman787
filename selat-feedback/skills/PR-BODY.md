# PR to SELAT-AI/selat-skills — OPENED

**https://github.com/SELAT-AI/selat-skills/pull/58** — `Add skill: destination-brief`
Opened 2026-08-07 by vickman787, `vickman787:add-skill-destination-brief` → `SELAT-AI:main`.
Head commit `c36da52` (unsigned). 1 commit, 5 files, 317 insertions.

The record below is kept for reference.

---

# Original submission notes

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

## How to open it — one click left

The branch is **already pushed** to your fork, authored as `vickman787`:

    https://github.com/vickman787/selat-skills/tree/add-skill-destination-brief

Opening the PR itself has to happen from the browser — creating it writes to
`SELAT-AI/selat-skills`, which cannot be added to a Claude Code session that already
has `vickman787` repos (cross-owner adds are unsupported), so both `fork_repository`
and `create_pull_request` return "not configured for this session".

**Open it here, then paste the body above:**

    https://github.com/SELAT-AI/selat-skills/compare/main...vickman787:selat-skills:add-skill-destination-brief?expand=1

Title: `Add skill: destination-brief`

`selat skill submit ./skills/destination-brief` would also work from a machine with
`gh` authenticated — it pushes and opens the PR in one step, using the verify receipt
that is already written.
