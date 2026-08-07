# Handoff — resume the SELAT bounty run here

**Session 3 got through discovery and the wallet.** The widened allowlist held, `selat search`
merges all 2596 services from all 5 catalogs, and `selat init` completed — wallet live with a
Circle spending policy on it. **Bounty steps 1-3 are done.** Findings 11-18 are written up.

**The only thing left blocking steps 4-6 is money.** Nothing is network-blocked and nothing needs
another policy change. The user has to send ~1 USDC to the wallet; everything after that is
mechanical and the endpoint plan is already verified below.

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

**Steps 1-3 are DONE.** Step 4 (funding) is the only thing blocking the rest, and it needs the
user to send USDC. Nothing else is in the way.

| Item | Status |
|---|---|
| `@selat-ai/selat-cli` | reinstall — still 0.15.7, no release since session 1 |
| `@selat-ai/selat-pay` | `npm i -g @selat-ai/selat-pay` for a shell-callable copy (needed for `--probe-only`) |
| `selat search` / `skill compare` | works (export `SELAT_ROUTER_URL` first) |
| `selat init` | **done** — needs a pty, see Finding 15 |
| Circle login | Vickmancrypto@gmail.com |
| Wallet | `0x01224a287d5cbf9bfbd9cec6f93007a661062aac` (existing default of 3, spans 3 chains) |
| Spending policy | **$0.50/tx · $2/day · $2/wk · $2/mo**, written on BASE |
| Config | `/root/.config/selat-pay/.env` (0600) |
| Gateway balance | **0 USDC — waiting on the user** |
| `selat doctor` | all green except balance; gains a Router reachability check post-init |
| `FINDINGS.md` | Findings 1-18. Sessions 1-3. Needs a payment section once step 5 runs. |
| `EVIDENCE.md` | raw output from all three sessions, with source line numbers |

## Step 4 — funding (THE BLOCKER)

The user sends USDC on **Base** to `0x01224a287d5cbf9bfbd9cec6f93007a661062aac`, then:

```bash
selat fund --amount <usdc> --yes --wait     # --wait blocks until spendable
```

**Confirm before running it** — it moves funds into Circle Gateway. Deposits take 5-10 min.
Note `selat fund --help` does not work (Finding 16); those flags come from error strings.
Watch out for gas: the Gateway deposit is an on-chain tx from the wallet, and it is untested
whether Circle sponsors it. If it fails for gas, the wallet needs a little ETH on Base.

**Send ~1 USDC.** The verified plan below costs $0.5475; 1 USDC leaves headroom for retries and
stays under the $2 daily cap.

## Step 5 — the paid-call plan (verified, do not re-derive)

All ten free-probed live with a real 402 and a live price. Bar is >=3 endpoints and >=0.5 USDC;
this is 10 endpoints and **$0.5475**, every call under the $0.50/tx cap.

```
$0.3000  POST  https://parallelmpp.dev/api/task
$0.0788  GET   https://googlemaps.mpp.tempo.xyz/solar/v1/dataLayers
$0.0600  POST  https://stablesocial.dev/api/reddit/search
$0.0420  POST  https://fal.mpp.tempo.xyz/xai/grok-imagine-image
$0.0158  GET   https://serpapi.mpp.tempo.xyz/search
$0.0105  GET   https://spyfu.mpp.tempo.xyz/apis/serp_api/v2/seo/*
$0.0105  GET   https://goflightlabs.mpp.tempo.xyz/flight-prices
$0.0100  POST  https://x402.tavily.com/search
$0.0100  POST  https://api.nansen.ai/api/v1/tgm/flows
$0.0100  GET   https://tripadvisor.x402.paysponge.com/api/v1/location/search
```

Saved at `/home/user/selat-run/plan.tsv` (method, url, price, name — tab separated). Re-probe
before paying, prices drift:

```bash
selat-pay <METHOD> <URL> --chain base --max-amount 0.50 --probe-only
```

Do **not** substitute endpoints from `selat search`'s top 5 without probing — ranking is
payability-blind (Finding 14), `*.mpp.paywithlocus.com` is 0-for-13 despite being reachable, and
`mpp.orthogonal.com` is 1-for-11 (Finding 18). Messari is live at $0.55 but exceeds the per-tx cap;
raising the cap costs another email OTP.

Drop `--probe-only` to settle. **Get the user's OK before each call**, surface the live price
first, and remember `selat freeze` is the kill switch.

## Step 6 — verify

`selat history` / `selat spend`, captured **before the session goes idle**. The container is
ephemeral and takes the local history with it.

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
