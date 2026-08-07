# Handoff — resume the SELAT bounty run here

**Session 3 completed the bounty run.** Steps 1-6 are **done**: install, discovery, wallet,
funding, paid calls, and verification. **6 distinct endpoints, $0.582750 USDC settled** against a
bar of >=3 endpoints and >=0.5 USDC. Findings 1-23 are written up in `FINDINGS.md` with raw
output in `EVIDENCE.md`.

**Step 8 is done too** — `destination-brief` submitted as SELAT-AI/selat-skills#58.

What remains is submitting the feedback itself; the channel for that was never identified.

Gateway balance left: **0.417250 USDC**, and it is **not** lost when this container dies.
Earlier notes in this file said otherwise; that was wrong. The wallet
`0x01224a287d5cbf9bfbd9cec6f93007a661062aac` is a Circle Agent Wallet on the user's own Circle
account — `selat init` found it among 3 pre-existing wallets on that account — so the balance is
reachable from any machine after `selat init` with the same email. What actually dies with the
container is only `/root/.config/selat-pay/.env` and the local
`/root/.local/state/selat-pay/gateway-history.jsonl` ledger. The `selat history` / `selat spend`
captures are committed to `EVIDENCE.md`, so nothing needed for the submission depends on this
container.

## First thing to do: confirm the policy still holds

```bash
for h in api.circle.com catalog.selat.ai api.apify.com agi.apify.com \
         api.cdp.coinbase.com mpp.dev router.selat.ai; do
  printf '%-24s %s\n' "$h" "$(curl -sS --noproxy '*' -o /dev/null -w '%{http_code}' --max-time 10 https://$h/)"
done
```

`403` means blocked (body reads `Host not in allowlist: <host>`). Anything else is fine —
session 3 saw `307/404/404/200/404/200/404` and everything worked. **All seven must be non-403.**
Don't lean on `selat doctor` for this: post-init it probes `router.selat.ai` and nothing else, so
it goes green while all five catalog registries are unreachable (Finding 17).

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
| `@selat-ai/selat-cli` | 0.15.7 — unchanged across all three sessions |
| `@selat-ai/selat-pay` | `npm i -g @selat-ai/selat-pay` for a shell-callable copy |
| Steps 1-2 install + discovery | ✅ done |
| Step 3 wallet | ✅ done — needs a pty (Finding 15) |
| Step 4 funding | ✅ done — 1 USDC deposited, ~10 min to settle |
| Step 5 paid calls | ✅ **done — 6 endpoints, $0.582750** |
| Step 6 verify | ✅ done — `selat history` / `selat spend` captured in `EVIDENCE.md` |
| Step 8 skill | ✅ **submitted — SELAT-AI/selat-skills#58** |
| Wallet | `0x01224a287d5cbf9bfbd9cec6f93007a661062aac` |
| Spending policy | $0.50/tx · $2/day · $2/wk · $2/mo, on BASE |
| Gateway balance | **0.417250 USDC** |
| `FINDINGS.md` | Findings 1-23, all three sessions |
| `EVIDENCE.md` | raw output, with source line numbers |

## Paid calls, as settled

```
$0.315000  200  POST  parallelmpp.dev/api/task                      (processor=ultra)
$0.105000  200  POST  parallelmpp.dev/api/task                      (processor=pro)
$0.063000  202  POST  stablesocial.dev/api/reddit/search
$0.042000  200  POST  fal.mpp.tempo.xyz/xai/grok-imagine-image
$0.015750  200  GET   serpapi.mpp.tempo.xyz/search
$0.010500  200  GET   tripadvisor.x402.paysponge.com/…/location/search
$0.010500  200  POST  x402.tavily.com/search
$0.010500  400  POST  x402.tavily.com/search                        ✗ charged, no body
$0.010500  422  POST  api.nansen.ai/api/v1/tgm/flows                ✗ charged, wrong body
```

**If you run more paid calls, pass `--body`.** `selat-pay` settles payment *before* the upstream
validates (Finding 21), so a malformed request is billed in full with no refund path. $0.021 of
the $0.583 went to two such calls. Get the required fields from the catalog's `inputSchema`:

```bash
selat search "<intent>" --top 50 --json | \
  jq -r '.results[] | select(.endpoint.url=="<URL>") | .endpoint.inputSchema'
```

Do **not** paste `exec_hints[].cmd` — it emits `--body '{}'` even when the schema has required
fields (Finding 20).

## Step 8 — optional skill contribution (+5 USDC)

User chose **scaffold only, do NOT open a PR**. Author a skill per
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
- This container is ephemeral, but **the wallet is not** — it lives on the user's Circle account
  and survives. Only the local config and `selat history` ledger die with the container, so
  capture spend output before the session goes idle.
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
