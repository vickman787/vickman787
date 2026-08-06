# Handoff — resume the SELAT bounty run here

**Session 3 unblocked discovery.** The user's widened allowlist held: all six critical hosts
answer, `selat search` merges all 2596 services from all 5 catalogs, and `selat skill compare`
runs. Bounty steps 1 and 2 are **done**. Findings 11–14 are written up.

**What's left is steps 3–6, and none of it is network-blocked.** It is blocked on the user:
an email, a 6-digit OTP they type, and ~1 USDC they send. Ask for the email first thing.

## First thing to do: confirm the policy still holds

```bash
for h in api.circle.com catalog.selat.ai api.apify.com agi.apify.com \
         api.cdp.coinbase.com mpp.dev router.selat.ai; do
  printf '%-24s %s\n' "$h" "$(curl -sS --noproxy '*' -o /dev/null -w '%{http_code}' --max-time 10 https://$h/)"
done
```

`403` means blocked (body reads `Host not in allowlist: <host>`). Anything else is fine —
session 3 saw `307/404/404/200/404/200/404` and everything worked. **All seven must be non-403.**
Don't trust `selat doctor` for this; per Findings 9, 10 and 14 it does not probe anything.

## Set this before running anything

```bash
export SELAT_ROUTER_URL=https://router.selat.ai
```

Without it `selat skill compare` fails on every candidate with
`missing --router-url or SELAT_ROUTER_URL`, even though it's a free command and the value is the
documented default (Finding 11). `selat init` writes it to `~/.config/selat-pay/.env`, so once
step 3 is done this is redundant — until then, export it.

## Reinstall (container is fresh each session)

```bash
npm i -g @selat-ai/selat-cli          # 0.15.7 — unchanged across all three sessions
claude plugin marketplace add SELAT-AI/selat-plugins
claude plugin install selat@selat-plugins
```

## State as of this handoff

| Item | Status |
|---|---|
| `@selat-ai/selat-cli` | reinstall — still 0.15.7, no release since session 1 |
| `selat doctor` | runs; 3 wallet failures; still no network section |
| `selat skill list --available` | ✅ 19 skills |
| `selat search` | ✅ **works** — 2596/2596 merged, all 5 catalogs |
| `selat skill compare` | ✅ works **with `SELAT_ROUTER_URL` exported** |
| Circle CLI | not installed; `selat init` installs it |
| wallet / funding / paid calls | **not started — waiting on the user** |
| `FINDINGS.md` | Findings 1–14. Sessions 1–3 complete. Needs a payment section if steps 3–6 run. |
| `EVIDENCE.md` | raw output from all three sessions, with source line numbers |

## Steps 3–6 — the remaining bounty work

**3. Wallet** — get the user's email, then `selat init`, which prompts for a 6-digit OTP.
Relay the prompt verbatim; the user types the code. Never run `circle` directly; never ask for a
private key. `api.circle.com` is reachable.

**4. Funding** — read the address back to the user, they send 1 USDC (Base recommended), then
`selat fund --chain base --amount 1 --method eco`. **Confirm before depositing.** Credits take
5–10 min; `--wait` blocks until spendable.

**5. Paid calls** — bar is **≥3 distinct endpoints and ≥0.5 USDC total**. `--dry-run` first,
surface the price, get an explicit OK per call. `selat freeze` is the kill switch.

**Pick targets from the reachable list below, not from `selat search`'s top 5.** Ranking is
payability-blind (Finding 14) and its top result usually sits on a host this environment blocks.
Two endpoints already probed clean in session 3:

```
Otto AI          $0.0010  GET  https://x402.ottoai.services/crypto-news
Apify person-enrichment  $1.05  POST https://api.apify.com/v2/actors/ryanclinton~person-enrichment-lookup/run-sync-get-dataset-items
```

$1.05 for the Apify one is most of the budget in a single call — check price before committing.
Other reachable, well-populated hosts to shortlist from: `mpp.orthogonal.com` (78 entries),
`*.mpp.paywithlocus.com` (apollo, alphavantage, coingecko, diffbot, hunter, brave, openweather,
clado, perplexity, deepl, replicate, …), `*.mpp.tempo.xyz` (serpapi, googlemaps, firecrawl,
kicksdb, flightapi, spyfu, oxylabs, fal), `x402.alchemy.com`, `x402.tavily.com`, `api.nansen.ai`,
`api.messari.io`, `api.exa.ai`, `parallelmpp.dev`, `stablesocial.dev`, `stabledomains.dev`,
`tripadvisor.x402.paysponge.com`.

Probe before paying — free and never settles:
```bash
selat skill compare "<intent>" --limit 5      # PROBE ✓ column = payable right now
```

**6. Verify** — `selat history` / `selat spend`, capture the output for the submission
**before the session goes idle**, not at the end.

**8. (optional, +5 USDC)** — user chose **scaffold only, do NOT open a PR**. Author a skill per
`/workspace/selat-ai/selat-skills/meta/skill-creator/SKILL.md`, run `selat skill validate` and the
free `selat skill verify`, then stop and hand it to them.

Reference clones (re-clone if missing):
`git clone --depth 1 https://github.com/selat-ai/selat-plugins /workspace/selat-ai/selat-plugins`
`git clone --depth 1 https://github.com/selat-ai/selat-skills  /workspace/selat-ai/selat-skills`

## The allowlist, for reference

The user applied all of this and it worked. Add `router.selat.ai` if it isn't already there.

```
# the five catalog registries — all required, Promise.all means any miss is fatal
*.circle.com
selat.ai  *.selat.ai            # covers catalog.selat.ai and router.selat.ai
*.apify.com
api.cdp.coinbase.com
mpp.dev                         # bare domain; NOT covered by *.mpp.tempo.xyz etc.

# infrastructure
raw.githubusercontent.com  registry.npmjs.org  *.npmjs.org

# merchant endpoints for paid calls
*.mpp.paywithlocus.com  *.mpp.tempo.xyz  mpp.orthogonal.com  parallelmpp.dev
*.x402.paysponge.com  x402.alchemy.com  x402.tavily.com  x402.ottoai.services
x402.api.agentmail.to  api.exa.ai  api.messari.io  api.nansen.ai
stabledomains.dev  stablesocial.dev

# chain RPCs for funding
mainnet.base.org  mainnet.optimism.io  arb1.arbitrum.io
```

Bare domains need their own line — the user's matcher treats `selat.ai` and `*.selat.ai` as
distinct. `mpp.dev` and `parallelmpp.dev` are the two that look covered and aren't.

**Do not ask the user to extend this further.** Session 3 measured the merchant host set at
**526 distinct hosts, 477 of them outside this list** (Finding 14). It is unbounded and grows
with every service listed. If a paid call 403s on a host not above, that is Finding 12, not a
policy gap — report it and pick a different endpoint from the reachable list.

## Standing constraints from the user

- Confirm before anything that spends or moves funds. Every time, not once.
- This container is ephemeral — the wallet and local `selat history` die with it. The user was
  told and chose to proceed here anyway. Get the screenshots before the session goes idle.
- Don't route around the egress proxy. If a host is blocked, report it.

## Decisions carried forward

- **Proceed with the wallet and paid calls.** Steps 3–5 are approved, target ≥3 endpoints /
  ≥0.5 USDC. Ask for the email at the start of the session; relay the OTP prompt verbatim.
- Scaffold the optional skill, hand it over, **no PR**.

## If the run stops here permanently

The write-up is a complete submission for the feedback half of the bounty: 14 findings across
three sessions, with source line numbers and reproductions. Findings 11 and 12 are the two worth
leading with — 12 especially, since it is a one-line change (route `probeUpstream()` through
`${routerUrl}/proxy?target=…`, which `selat-pay.mjs:1339` already builds) that would collapse
SELAT's egress requirement from the entire merchant long tail to a single host.
