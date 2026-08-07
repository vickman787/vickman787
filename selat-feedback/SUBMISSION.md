# SELAT bounty submission, vickman787

Full write-up: **https://github.com/vickman787/vickman787/blob/Json/selat-feedback/FINDINGS.md**
(24 findings, with source line numbers and reproductions)
Raw terminal output: **[`EVIDENCE.md`](https://github.com/vickman787/vickman787/blob/Json/selat-feedback/EVIDENCE.md)**

---

## 1. Setup, and what I tried

| | |
|---|---|
| Harness | Claude Code (remote container via claude.ai/code) |
| OS | Linux 6.18.5 x86_64 · Node v22.22.2 · npm 10.9.7 |
| Versions | `@selat-ai/selat-cli` **0.15.7** · bundled `selat-pay` 0.9.4 · plugin `selat@selat-plugins` 0.1.8 |
| Network | **Egress-restricted sandbox with a host allowlist.** This turned out to be the single most important fact about the run. |
| Duration | Three sessions over ~8 hours |
| Also tested | Windows, where `selat init` cannot complete at all in 0.15.7 (Finding 25) |

Ran the full flow: install → discovery → wallet → funding → paid calls → verify → skill contribution.
Completed steps 1 to 6 and the optional step 8.

**Sessions 1 and 2 produced no paid calls at all.** Both were consumed entirely by network egress
problems, not because the sandbox was unusually strict, but because SELAT's own documentation and
error messages named the wrong hosts to allow. That is Findings 1, 7, 8, 9 and 13, and it is worth
reading as one story rather than five bugs.

Session 3 got everything working and spent **$0.582750 USDC** across **6 distinct endpoints**.

---

## 2. Discovery experience

**The ranking is genuinely good, and better than I expected.** `selat search "web search"` merged
**2596 services from all 5 catalogs** (raw 2670 → deduped), reporting `690 matched a token,
88 on-target`, and returned a sensible top 5 ordered by name match blended with price.

Three things I'd single out as well-designed:

- **The `why:` line.** `why: matched web, search in tags+name · $0.0010/call` makes the ranking
  auditable instead of magic. I could tell at a glance when a match was name-only versus
  tag-corroborated. More products should do this.
- **Honest about the tail.** `(602 weaker description-only matches hidden — --json to see them.)`
  rather than pretending the top 5 is the whole answer.
- **`selat skill list --available`** with per-skill reliability dots and a `checked 9h ago`
  timestamp is the best surface in the CLI. The reliability indicator is the thing that would make
  me trust spending money.

### What was missing or ranked oddly

**Ranking is payability-blind, this is the big one.** Scoring is name-match × price. Nothing in it
accounts for whether a service can actually be paid *right now*. `--explain` is documented as
showing "why each match is or isn't payable right now", but the ranking itself doesn't consume
that, so the top result is routinely one that cannot be reached or settled.

I measured it. I free-probed 42 catalogue endpoints across every reachable merchant family:

| Family | probed | live 402 |
|---|---|---|
| `*.mpp.tempo.xyz` | 8 | **6** |
| `parallelmpp.dev` | 2 | **2** |
| `api.messari.io` | 2 | **2** |
| `api.nansen.ai` | 2 | **2** |
| direct x402 (`tavily`, `paysponge`, `stablesocial`) | 4 | **4** |
| `mpp.orthogonal.com` | 11 | 1 |
| `*.mpp.paywithlocus.com` | 13 | **0** |

**`*.mpp.paywithlocus.com` went 0 for 13.** CoinGecko, Brave Search, Wolfram|Alpha, Stability AI,
ScreenshotOne, Deepgram, Hunter, RentCast and Billboard are all listed, all on a host that answers
HTTP, and none of them served a payment challenge. Nothing in `selat search` output distinguishes
those listings from the ones that work, they rank identically.

Feeding the `skill compare` probe result, or the reliability dot `skill list` already has, back
into `search` ordering would be a large quality win.

**`1/5 catalogs` is near-constant and therefore carries little signal.** Almost nothing is
corroborated across registries. Where it *does* vary, `stableenrich.dev` showing
`[circle,agentic,mpp]`, that's genuinely useful information: a service three registries
independently list is a better bet. Worth surfacing more prominently than a name match.

**Prices in the catalogue are not the prices you pay.** Every call charged ~5% over the listed
`minAmountUsd` (`$0.0150` → `$0.015750`). Worse, Parallel's price depends on the *request body*
(`processor: pro` charged $0.105 against a listed $0.30; `ultra` charged $0.315) while
`--probe-only`, which sends no body, quoted `$0.300000` for both. An agent that budgets from
search output will misjudge every call.

---

## 3. Where I got stuck or confused

### Onboarding / network (cost me two full sessions)

- **The remediation text names the wrong hosts.** `selat search` failed on `api.apify.com`, a host
  absent from the allowlist its own error message told me to configure. (Finding 1)
- **One unreachable registry zeroes out all discovery.** `loadFederatedCatalog` fans out to five
  registries via `Promise.all` with only the fifth wrapped in a `.catch`, so one bad host discards
  the other four. I had three registries reachable and got **zero** results, with no indication
  which host was at fault. (Finding 7)
- **The shipped allowlist constant names 2 of the 5 catalogue hosts** and asserts those two are the
  complete set. (Finding 8)
- **The egress hint is gated on a probe of `api.circle.com` alone**, the host users allow *first*,
  so the hint goes silent for exactly the users who followed the docs partway. (Finding 9)

### Wallet setup

- **`selat init`'s documented headless path can't complete an init.** `--email` and `--wallet` are
  advertised "for agent shells / CI with no TTY", and both are honoured, then step 4 of 8 shells
  out to the Circle CLI, which opens its own TTY-only OTP prompt and exits 1. Its remediation then
  points you at raw `circle wallet login`, i.e. outside the abstraction SELAT is selling. I worked
  around it by running `selat init` under `script(1)` so it drove the Circle login itself.
  (Finding 15)
- **`selat init` cannot complete on Windows at all, in the current release.** Every path selat has
  for locating and running the Circle CLI is POSIX only:
  `command -v circle` gives ENOENT because `command` is a shell builtin; `which` only exists if
  Git Bash provides it; `spawn("circle")` gives ENOENT because Node does not apply PATHEXT and the
  shim is `circle.cmd`; and `spawn("C:\...\circle.cmd")` gives **EINVAL**, because Node 18.20 and
  later refuse to execute `.cmd` without `shell: true` (the CVE-2024-27980 hardening). So even
  correctly resolving the absolute path fails on any current Node.
  Worse, it presents as the wrong problem: under Git Bash the detection step passes, so `doctor`
  reports **"not authenticated"** and cannot read the spending policy. A Windows user goes looking
  for a broken login when selat simply never executed the binary. I moved to WSL rather than patch
  the installed package. (Finding 25)
- The **spending-policy prompt is the best safety design in the product.** Offered unprompted,
  before funding, explains *why* in one sentence ("the one hard ceiling the agent literally cannot
  bypass"), defaults to yes. Every agent-payments tool should do this. I set $0.50/tx · $2 per
  day/week/month.

### Funding

- **`selat fund --help` doesn't print help.** Both `--help` and `-h` hit the amount prompt before
  the flag is parsed, so they die on the no-TTY check or hang under a pty. It is the one command
  that moves USDC and the one whose documentation you cannot read. I learned its flags from error
  strings. (Finding 16)
- **`selat doctor`'s "did my deposit land?" advice is an invalid command.**
  `circle gateway balance --all` errors with `--address, --chain are required`. That message is
  shown at the exact moment a user is anxious about whether their money arrived. (Finding 19)
- Deposit-to-spendable took ~10 minutes, at the far end of the documented 5 to 10.

### Payment (this is where it cost money)

- **My first paid call failed and was charged in full.** I sent a POST without a body; Tavily
  returned `400 Validation failed`; I was billed **$0.0105**. Later, a wrong guess at Nansen's
  schema returned `422` and was billed another $0.0105. **$0.021 of my $0.583 bought two error
  messages.** (Finding 21)
- **SELAT's own generated command would have done the same thing.** Every POST record in the
  catalogue ships `exec_hints[].cmd` ready to paste, and every one of them says `--body '{}'`,
  while the *same record* carries `inputSchema.required`. `parallelmpp.dev` needs
  `["input","processor"]`, `stablesocial.dev` needs `["keywords"]`. Copy-pasting the hint SELAT
  generates is a paid 400. (Finding 20)
- **`--probe-only` doesn't predict payability.** It returns a clean 402 for a request the merchant
  will reject, because it never sends the body. My "10 verified endpoints" were verified only to
  the depth of *will quote a price*.
- **A network 403 is reported as a merchant defect.** When the direct probe is blocked, you get
  `no x402 or MPP challenge detected at <url>`, the actual response was
  `403 Host not in allowlist`. `probeUpstream()` captures `res.status` and the caller discards it.
  This fired on **19 of 23** candidates and made the catalogue look full of broken listings when
  the real cause was my egress policy. (Finding 13)

---

## 4. One thing I'd expect SELAT to do that it didn't

**Check my request against the schema it already has, before spending my money.**

The sequence in `selat-pay` is: sign → settle → send → *then* discover the request was malformed.
The ledger records the outcome accurately and impotently:

```json
{"upstreamUrl":"https://x402.tavily.com/search","amountUsd":0.0105,
 "httpStatus":400,"ok":false,"outcome":"failed"}
```

`ok:false`, `outcome:"failed"`, money gone. Nothing between "the user typed a command" and "the
USDC moved" inspects the body against the `inputSchema` SELAT already holds and already ships in
its catalogue payload. The pieces exist, `probeUpstream()` has a sample-value generator for
exactly this, and there's a `SELAT_PAY_VERIFIED_SCHEMAS_PATH` store, and neither runs on the
paying path. There is also no `--dry-run` on `selat-pay`, though `selat run` advertises one.

What makes this more than an oversight: **`meta/skill-creator/SKILL.md` warns contributors about
it in prose.**

> *…a wrong param name or shape still costs money: the SELAT Router settles the payment **before**
> the upstream validates the body, and a `verify` probe checks payability, not param correctness.*

So the hazard is understood well enough to document, and the mechanical check that would enforce
it isn't there. Documentation is not a mitigation when the failure costs money and the check is
trivial. (Findings 20, 21, 24)

### The structural counterpart, and the one I'd actually fix first

**Route the 402 probe through your own router.** `selat-pay` already builds
`${routerUrl}/proxy?target=<upstream>` at line 1339 and uses it for settlement, but the
*detection* probe at line 1097 fetches the merchant directly. So discovery requires direct egress
to every merchant domain in the catalogue, while settlement doesn't.

I verified the router reaches a host my environment blocks:

```
$ curl -s -o /dev/null -w '%{http_code}' https://api.agentstools.dev/search
403        Host not in allowlist: api.agentstools.dev

$ curl -sD - "https://router.selat.ai/proxy?target=https%3A%2F%2Fapi.agentstools.dev%2Fsearch"
HTTP/2 402
payment-required: eyJ4NDAyVmVyc2lvbiI6MiwicmVzb3VyY2Ui…
```

That decodes to a complete x402 v2 challenge, `amount "1050"` ($0.00105) across 10 chains. **The
blocked host is fully probeable through SELAT's own router.**

Why it matters at scale: I censused the catalogue across eight intents and found **526 distinct
merchant hosts, 477 of them (91%) outside any allowlist I could reasonably maintain**, only 34%
of catalogue entries were reachable. Every merchant registers its own domain, and the set grows
with every listing. No allowlist can track it. **Moving the probe to the router collapses SELAT's
egress requirement from the entire merchant long tail to one host.** It is a one-line change, and
it would have saved all three sessions of this run. (Findings 12, 14)

---

## 5. Gateway transactions, $0.582750 USDC across 6 distinct endpoints

Bar was ≥3 endpoints and ≥0.5 USDC. Gateway balance `1.000000` → **`0.417250`**.

### `selat history`

```
History file: /root/.local/state/selat-pay/gateway-history.jsonl
Wallet:       0x01224a287d5cbf9bfbd9cec6f93007a661062aac
Matched: 9   Showing: 9   Total shown: $0.582750 USDC

time                      chain  mode          amount     status method url
2026-08-07T06:09:38.445Z  base   routed-mpp    $0.315000  200    POST   parallelmpp.dev/api/task
2026-08-07T06:09:13.587Z  base   routed-mpp    $0.010500  422    POST   api.nansen.ai/api/v1/tgm/flows
2026-08-07T06:09:07.185Z  base   routed-x402   $0.010500  200    GET    tripadvisor.x402.paysponge.com/api/v1/location/search
2026-08-07T06:08:58.128Z  base   routed-mpp    $0.015750  200    GET    serpapi.mpp.tempo.xyz/search
2026-08-07T06:08:42.459Z  base   routed-mpp    $0.042000  200    POST   fal.mpp.tempo.xyz/xai/grok-imagine-image
2026-08-07T06:07:40.369Z  base   routed-mpp    $0.105000  200    POST   parallelmpp.dev/api/task
2026-08-07T06:07:28.075Z  base   routed-mpp    $0.063000  202    POST   stablesocial.dev/api/reddit/search
2026-08-07T06:07:05.090Z  base   routed-x402   $0.010500  200    POST   x402.tavily.com/search
2026-08-07T02:04:36.480Z  base   routed-x402   $0.010500  400    POST   x402.tavily.com/search
```

All signed by Circle MPC, SCA owner `0x9FC67499eB58AB608787a5C5F43F7230E9916523`.

### `selat spend`

```
Settled spend (money out of Gateway)           source: selat-pay ledger
  provider                   calls       settled  failed
  parallelmpp.dev                2     $0.420000       0
  stablesocial.dev               1     $0.063000       0
  fal.mpp.tempo.xyz              1     $0.042000       0
  serpapi.mpp.tempo.xyz          1     $0.015750       0
  tavily.com                     2     $0.010500       1
  tripadvisor.x402.paysponge.com 1     $0.010500       0
  nansen.ai                      1     $0.000000       1
                                       $0.561750   total
  ⚠ 2 failed call(s); $0.021000 charged-but-failed
    There is no automatic dispute or chargeback rail for these payments.
    To follow up, contact the provider directly with the quoteId/tx refs from `selat history`.
```

**Credit where it's due:** that warning is exactly right and rare. It states the loss, says plainly
there is no recourse, and points at the identifiers you'd need to chase it. No product wants to
write that sentence and writing it is the correct call. One nuance, the `$0.561750 total` above
it counts only successful settlements, so it understates actual Gateway outflow ($0.582750) by the
$0.021 it flags separately. (Finding 23)

### The errors, in full

**Charged 400**, POST without a body:

```
$ selat-pay POST https://x402.tavily.com/search --chain base --max-amount 0.02
[selat-pay] detected: x402=yes mpp=no; mode=routed-x402
[selat-pay] price=$0.010500 on eip155:8453
[selat-pay] resolved Circle SCA owner 0x9FC67499eB58AB608787a5C5F43F7230E9916523
[selat-pay] signed; submitting paid request
[selat-pay] status=400
{"error":{"message":"Validation failed","statusCode":400,
  "details":[{"msg":"Query must be a non-empty string with max 1000 characters",
              "path":"query","location":"body"}]}}
```

Balance before `1.000000` → after `0.989500`.

**Charged 422**, wrong body shape for Nansen: `"error": "Missing field"`, billed $0.010500.

**Discovery blocked (sessions 1 to 2)**, same call, two different reports:

```
$ selat search "web search"
Fatal: 403 api.cdp.coinbase.com

$ selat skill compare "web search" --limit 5
  3  web-search   ?  —  2103ms  ✗
     GET https://api.agentstools.dev/search
     · no x402 or MPP challenge detected at https://api.agentstools.dev/search
        ← actual response was 403 Host not in allowlist: api.agentstools.dev
```

**Free command blocked on config:**

```
$ selat skill compare "summarize a webpage" --limit 3
  1  Automaton Webpage Change Mon..   ?  —  1116ms  ✗  · missing --router-url or SELAT_ROUTER_URL
  2  Wikipedia Article Summary        ?  —  1135ms  ✗  · missing --router-url or SELAT_ROUTER_URL
  3  YouTube Summary & Transcript     ?  —  1218ms  ✗  · missing --router-url or SELAT_ROUTER_URL
```

`https://router.selat.ai` is documented as the default in three shipped files and written by
`selat init`, but `selat-pay.mjs:1192` has no fallback, so a free, wallet-free command is gated
behind wallet creation for no functional reason. (Finding 11)

---

## 6. Skill PR (step 8)

**https://github.com/SELAT-AI/selat-skills/pull/58**, `Add skill: destination-brief`

A 3-step routed skill: Tripadvisor location search → Tavily web context → SerpApi SERP,
~$0.037 per run. Picked travel because none of the hub's 18 existing skills cover it.

`selat skill validate` ✓ · `selat skill verify` ✓ 3/3 within cap · `npm run validate` ✓ 19 skills,
0 errors, 0 warnings.

All three endpoints were exercised with **real settled calls** while authoring, not only probed, so
`references/endpoints.md` records live quote IDs and observed prices rather than catalogue claims,
including the Tavily schema drift I paid $0.0105 to discover.

---

## Fix list, in the order I'd do them

1. **Finding 12**, route `probeUpstream()` through `${routerUrl}/proxy?target=…`, the URL the same
   file already builds 240 lines later. Collapses egress from 526 hosts to one.
2. **Finding 21**, validate `--body` against `inputSchema` before signing. $0.021 of my $0.58 went
   to requests the tool could have rejected for free.
3. **Finding 20**, stop emitting `--body '{}'` in `exec_hints[].cmd` for endpoints with required
   fields. It is a copy-paste path to a paid 400.
4. **Finding 25**, use `where.exe` on win32 and invoke Circle via `node dist/index.js` rather than
   the `.cmd` shim. Windows is currently a platform on which the product cannot be onboarded.
5. **Finding 11**, default `SELAT_ROUTER_URL`. One `??`.
6. **Finding 16**, handle `--help` in `selat fund` before the TTY check.

1 and 2 are the difference between "promising" and "usable". 3 is a paid-error generator.
4 blocks an entire operating system. 5 and 6 are half-hour fixes.
