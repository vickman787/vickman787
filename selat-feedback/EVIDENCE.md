# Raw terminal evidence — SELAT session 2026-08-06T22:25:19Z

## Environment
```
harness: Claude Code (remote container, claude.ai/code)
os: Linux 6.18.5-fc-v18 x86_64
node: v22.22.2  npm: 10.9.7
selat-cli: 0.15.7
plugin: selat@selat-plugins 0.1.8
```

## selat doctor
```

selat doctor — checking your setup

Binaries:
  ✓ node v22.22.2
  ✓ npm 10.9.7
  ✓ git git version 2.43.0

Agent-payment skill:
  ✓ skill at /opt/node22/lib/node_modules/@selat-ai/selat-cli/node_modules/@selat-ai/selat-discovery

Circle CLI:
  ✗ circle CLI not installed — install Circle CLI, then run `selat init`

Agent Wallet:
  ✗ no agent wallet found — run `selat init`

Spending policy:
  ⚠ could not read the wallet spending policy (circle wallet limit budget)

selat-pay:
  ✓ selat-pay installed (bundled, v0.9.4) — /opt/node22/lib/node_modules/@selat-ai/selat-cli/node_modules/@selat-ai/selat-pay/bin/selat-pay.mjs
    only reachable via `selat run`; for a shell-callable copy: npm i -g @selat-ai/selat-pay

Config:
  ✗ /root/.config/selat-pay/.env missing or empty — run `selat init`

3 check(s) failed.
Most fixes: install missing tools, then run `selat init` again.
[exit 1]
```

## selat skill list --available  (WORKS offline-ish: registry is on raw.githubusercontent.com)
```

▸ Available skills
  ●  enrich-waterfall          (multi/mixed)     Cheapest-first B2B person+company enrichment waterfall across 14 MPP merchants. Paid per call via x402 + MPP.
  ●  comprehensive-enrichment  (multi/MPP on Tempo)  Deep multi-source person+company enrichment (Fiber, Hunter, Tomba, Sixtyfour, Brand.dev, Exa…). Paid per call via MPP on Tempo.
  ●  lead-enrichment           (multi/MPP on Tempo)  Lead enrichment via Hunter + Sixtyfour + Fiber. Paid per call via MPP on Tempo.
  ●  person-lookup             (single/MPP on Tempo)  Person lookup via Nyne. Paid per call via MPP on Tempo.
  ●  gtm-enrichment-smart      (multi/mixed)     Smart GTM enrichment (Apollo, Tomba, Brand.dev, Sixtyfour, Scrape Creators). Paid per call via x402 + MPP.
  ●  gtm-enrichment-deep       (multi/MPP on Tempo)  Deep GTM enrichment (Apollo + Sixtyfour). Paid per call via MPP on Tempo.
  ●  sales-prospecting         (multi/MPP on Tempo)  Sales prospecting list-build + enrich (Fiber, Hunter, Sixtyfour, Brand.dev). Paid per call via MPP on Tempo.
  ●  email-campaign            (multi/MPP on Tempo)  Email-campaign prospect + verify pipeline (Fiber, Hunter, Sixtyfour, Brand.dev). Paid per call via MPP on Tempo.
  ●  recent-funding-rounds     (single/MPP on Tempo)  Recent funding rounds via Fundable. Paid per call via MPP on Tempo.
  ●  find-twitter-influencers  (multi/mixed)     Find Twitter/X influencers (Scrape Creators, Fiber, Exa, Brand.dev, Hunter, Tomba). Paid per call via x402 + MPP.
...(19 skills listed, truncated)
```

## selat search "web search"  (FAILS: first host is api.apify.com)
```
Loading federated catalog (refresh=false)...
Fatal: 403 https://api.apify.com/v2/store?limit=500&offset=0&allowsAgenticUsers=true&sortBy=totalUsers: Host not in allowlist: api.apify.com. Add this host to your network egress settings to allow access.

Your agent host may be sandboxing network egress (e.g. Cursor 2.5+ denies it by
default). SELAT discovery needs api.circle.com + router.selat.ai. Allowlist them in
.cursor/sandbox.json (or ~/.cursor/sandbox.json):
  { "networkPolicy": { "default": "deny", "allow": ["api.circle.com","*.selat.ai","registry.npmjs.org","*.npmjs.org"] } }
…or run the selat command in a normal terminal outside the sandbox.
[exit 1]
```

## same command, routed through an HTTPS CONNECT proxy — host name is lost
```
Loading federated catalog (refresh=false)...
Fatal: fetch failed

Your agent host may be sandboxing network egress (e.g. Cursor 2.5+ denies it by
default). SELAT discovery needs api.circle.com + router.selat.ai. Allowlist them in
.cursor/sandbox.json (or ~/.cursor/sandbox.json):
  { "networkPolicy": { "default": "deny", "allow": ["api.circle.com","*.selat.ai","registry.npmjs.org","*.npmjs.org"] } }
…or run the selat command in a normal terminal outside the sandbox.
```

## Hosts referenced by @selat-ai/selat-discovery
```
      4 https://api.apify.com
      2 https://catalog.selat.ai
      2 https://api.circle.com
      1 https://docs.apify.com
      1 https://agi.apify.com
```

---

# Session 2 — allowlist applied per the docs, discovery still dead

Same machine class, fresh container. The user set **Network access = Custom** and allowed the
hosts the previous session identified. `@selat-ai/selat-cli` 0.15.7 (unchanged), Node v22.22.2,
plugin `selat@selat-plugins` reinstalled.

## Host reachability (curl direct, bypassing the harness proxy)
```
api.circle.com           307   reachable
router.selat.ai          404   reachable
catalog.selat.ai         404   reachable
api.apify.com            404   reachable
agi.apify.com            200   reachable
api.cdp.coinbase.com     403   BLOCKED
mpp.dev                  403   BLOCKED
```
The two 403s are the egress filter, not the origin:
```
$ curl https://api.cdp.coinbase.com/
Host not in allowlist: api.cdp.coinbase.com. Add this host to your network egress settings to allow access.
$ curl https://mpp.dev/
Host not in allowlist: mpp.dev. Add this host to your network egress settings to allow access.
```

## `selat search` — 5 of 7 catalog hosts reachable, still zero results
```
$ selat search "web search"
Loading federated catalog (refresh=false)...
Fatal: 403 https://api.cdp.coinbase.com/platform/v2/x402/discovery/resources?limit=1000&offset=0:
       Host not in allowlist: api.cdp.coinbase.com. Add this host to your network egress settings to allow access.
$ echo $?
1
```
Note what is **not** printed: the sandbox remediation block that appeared in session 1. See below.

## `selat skill compare` — same failure, one layer down
```
$ selat skill compare "search the web for recent news" --limit 3
▸ discovering candidates for "search the web for recent news" (free — no spend)…
✗ discovery failed (rank.mjs exited 1)
Loading federated catalog (refresh=false)...
Fatal: 403 https://api.cdp.coinbase.com/... Host not in allowlist: api.cdp.coinbase.com.
```

## `selat skill list --available` — works (Finding 4 reproduced)
19 skills returned with reliability dots, "checked 9h ago". Reads `raw.githubusercontent.com` only.

## `selat doctor` — still no network section (Finding 5 reproduced on a *working* network)
```
Binaries:            ✓ node ✓ npm ✓ git
Agent-payment skill: ✓
Circle CLI:          ✗ not installed
Agent Wallet:        ✗ none
Spending policy:     ⚠ unreadable
selat-pay:           ✓ bundled v0.9.4
Config:              ✗ /root/.config/selat-pay/.env missing
3 check(s) failed.
```
Zero mention of the two unreachable catalog hosts that are the actual blocker.

## The five catalog registries, from the shipped source
`node_modules/@selat-ai/selat-discovery/scripts/`:
```
discover_circle_catalog.mjs    https://api.circle.com/v2/x402/discovery/resources
discover_agentic_catalog.mjs   https://api.cdp.coinbase.com/platform/v2/x402/discovery/resources
discover_mpp_catalog.mjs       https://mpp.dev/api/services
discover_apify_catalog.mjs     https://api.apify.com/v2/store   (+ agi.apify.com for tokens)
discover_selat_catalog.mjs     https://catalog.selat.ai
```

## The fan-out — `discover_federated_catalog.mjs:1102`
```js
export async function loadFederatedCatalog({ refresh = false, warn = console.warn } = {}) {
  const [circle, agentic, mpp, apify, selat] = await Promise.all([
    loadOrFetchCircleCatalog({ refresh }),
    loadOrFetchAgenticCatalog({ refresh }),
    loadOrFetchMppCatalog({ refresh }),
    loadOrFetchApifyCatalog({ refresh }),
    // 5th registry: SELAT-native first-party catalog (catalog.selat.ai).
    // Isolated so an outage here can't sink the other four.
    loadOrFetchSelatCatalog({ refresh }).catch((err) => {
      warn(`SELAT catalog load failed: ${err.message}`);
      return { services: [] };
    }),
  ]);
```
Source 5 is isolated. Sources 1–4 are not.

## The allowlist constant — `lib/host.mjs:83`
```js
// The catalog hosts SELAT discovery must reach. Mirrors guides/cursor.md +
// install.md so the in-CLI hint and the docs stay in lockstep.
const SANDBOX_ALLOW = ["api.circle.com", "*.selat.ai", "registry.npmjs.org", "*.npmjs.org"];
```
`api.cdp.coinbase.com`, `mpp.dev`, and `*.apify.com` are absent. Same list verbatim in
`selat-plugins/install.md:57` and `guides/cursor.md:48`, where the prose asserts:
> `api.circle.com` + `*.selat.ai` are the catalog hosts discovery needs

## Why the remediation hint went silent — `lib/host.mjs:109`
```js
export async function egressLikelyBlocked() {
  const ac = new AbortController();
  const timer = setTimeout(() => ac.abort(), 4000);
  try {
    const r = await fetch("https://api.circle.com/v2/x402/discovery/resources", {
      method: "GET", signal: ac.signal
    });
    return r.status === 403 || r.status === 407;
  } catch {
    return true;
  } finally { clearTimeout(timer); }
}
```
Callers — all three gate the hint on this one probe:
```
lib/commands/search.mjs:109  if (await egressLikelyBlocked()) console.error("\n" + fmt.dim(sandboxHintText()));
lib/commands/skill.mjs:518   if (await egressLikelyBlocked()) console.error("\n" + fmt.dim(sandboxHintText()));
lib/commands/run.mjs:124     if (await egressLikelyBlocked()) console.error("\n" + fmt.dim(sandboxHintText()));
```
`api.circle.com` is allowlisted here, so it answers 307 → probe returns `false` → hint suppressed,
while discovery is dead on two other hosts.

## Full host inventory referenced by @selat-ai/selat-discovery (session 2, v0.15.7)
```
     10 https://github.com              6 https://api.exa.ai
      9 https://api.apify.com           6 https://agentskills.io
      6 https://api.nansen.ai           4 https://x402.alchemy.com
      4 https://mpp.dev                 4 https://developers.circle.com
      4 https://api.cdp.coinbase.com    3 https://agi.apify.com
      2 https://www.x402.org            2 https://docs.cdp.coinbase.com
      2 https://catalog.selat.ai        2 https://api.coingecko.com
      2 https://api.circle.com          1 https://router.selat.ai
```
(Session 1 saw only 5 hosts here; the count grew with the x402-bazaar rename.)

---

# Session 3 — allowlist held; discovery ran end to end

`@selat-ai/selat-cli` 0.15.7 (unchanged) · Node v22.22.2 · npm 10.9.7 · 2026-08-06

## Host reachability — all six critical hosts open (curl direct, bypassing the harness proxy)

```
$ for h in api.circle.com catalog.selat.ai api.apify.com agi.apify.com \
           api.cdp.coinbase.com mpp.dev; do
    printf '%-24s %s\n' "$h" "$(curl -sS --noproxy '*' -o /dev/null \
      -w '%{http_code}' --max-time 10 https://$h/)"
  done
api.circle.com           307
catalog.selat.ai         404
api.apify.com            404
agi.apify.com            200
api.cdp.coinbase.com     404      <- was 403 in session 2
mpp.dev                  200      <- was 403 in session 2
```

No 403 anywhere. The session-2 GAP hosts were the whole blocker.

## Merchant hosts from the session-2 allowlist — all reachable

```
x402.ottoai.services  200    api.messari.io    200    api.nansen.ai        200
x402.alchemy.com      200    x402.tavily.com   200    x402.api.agentmail.to 302
stabledomains.dev     200    stablesocial.dev  200    parallelmpp.dev      200
mpp.orthogonal.com    200    api.exa.ai        404    brave.mpp.paywithlocus.com 404
flightapi.mpp.tempo.xyz 404  api.apify.com     404    mainnet.base.org     405
```

## `selat search "web search"` — works, full 5-catalog merge

```
Intent: "web search"
Catalog: 2596/2596 merged services (raw 2670); 690 matched a token; 88 on-target.

 1. web-search                        score 0.915
    $0.0010  1/5 catalogs (agentic)
    GET   https://api.agentstools.dev/search
    why: matched web, search in name · $0.0010/call
 2. web-search.api.klymax402.com      score 0.899
    $0.0030  1/5 catalogs (agentic)
    POST  https://web-search.api.klymax402.com/api/search
 3. Web Search, News & Page Reader    score 0.881
    $0.0100  1/5 catalogs (agentic)
    GET   https://websearch.use.x402atlas.com/search
 4. Brave Search                      score 0.736
    $0.0350  1/5 catalogs (mpp)
    POST  https://brave.mpp.paywithlocus.com/brave/web-search
 5. search.reversesandbox.com         score 0.715
    $0.0020  1/5 catalogs (agentic)
    GET   https://search.reversesandbox.com/web/search

(602 weaker description-only matches hidden — --json to see them.)
```

## `selat skill compare` on a clean install — dies on config, not on network (Finding 11)

```
$ selat skill compare "summarize a webpage" --limit 3
▸ discovering candidates for "summarize a webpage" (free — no spend)…
▸ probing 3 candidates at their catalog serviceUrl (free — never settles)…

  #  SERVICE                         PRICE  RAIL  LATENCY  PROBE  REL
  1  Automaton Webpage Change Mon..      ?  —      1116ms  ✗      ○
     · missing --router-url or SELAT_ROUTER_URL
  2  Wikipedia Article Summary           ?  —      1135ms  ✗      ○
     · missing --router-url or SELAT_ROUTER_URL
  3  YouTube Summary & Transcript        ?  —      1218ms  ✗      ○
     · missing --router-url or SELAT_ROUTER_URL
```

Supplying the value the shipped README calls the default makes the command work:

```
$ SELAT_ROUTER_URL=https://router.selat.ai selat skill compare "summarize a webpage" --limit 3
  1  YouTube Summary & Transcript   ?  —  1336ms  ✗   · no x402 or MPP challenge detected at …
  2  Wikipedia Article Summary      ?  —  1433ms  ✗   · no x402 or MPP challenge detected at …
  3  Automaton Webpage Change Mon.. ?  —  1725ms  ✗   · no x402 or MPP challenge detected at …
```

Source, `@selat-ai/selat-pay/bin/selat-pay.mjs`:

```
1192:  const routerUrl = (args.routerUrl ?? process.env.SELAT_ROUTER_URL ?? "").replace(/\/$/, "");
1210:  if (!routerUrl) throw new Error("missing --router-url or SELAT_ROUTER_URL");
```

Documented as a default in: `selat-cli/README.md:198`, `selat-pay/README.md:36`,
written by `selat-cli/lib/commands/init.mjs:248`.

## The direct probe vs the routed settlement (Finding 12)

```
selat-pay.mjs:1097   const res = await fetch(upstreamUrl, {…});      // probeUpstream — DIRECT
selat-pay.mjs:1339   const targetForPayment =
                       `${routerUrl}/proxy?target=${encodeURIComponent(upstreamUrl)}`;
selat-pay.mjs:1394   routerProbe = await fetch(targetForPayment, {…});  // ROUTED
selat-pay.mjs:1561   paidRes     = await fetch(targetForPayment, {…});  // ROUTED
```

Proof the router reaches what the environment blocks:

```
$ curl -sS --noproxy '*' -o /dev/null -w '%{http_code}\n' https://api.agentstools.dev/search
403
$ curl -sS --noproxy '*' https://api.agentstools.dev/search
Host not in allowlist: api.agentstools.dev. Add this host to your network egress settings…

$ curl -sSD - --noproxy '*' \
    "https://router.selat.ai/proxy?target=https%3A%2F%2Fapi.agentstools.dev%2Fsearch"
HTTP/2 402
payment-required: eyJ4NDAyVmVyc2lvbiI6MiwicmVzb3VyY2UiOnsidXJsIjoiL3Byb3h5P3RhcmdldD1odHRwcyUzQSUyRiUyRmFwaS5hZ2VudHN0b29scy5kZXYlMkZzZWFyY2gi…
{}
```

Decoded `payment-required` (x402 v2): `amount "1050"` (= $0.00105 USDC, 6dp),
`payTo 0x1E5Be7e87A876C04AF0ffd725adccf02e998c5C2`,
`extra.name GatewayWalletBatched`, `verifyingContract 0x77777777dcc4d5a8b6e418fd04d8997ef11000ee`,
offered on `eip155:` 1, 8453, 43114, 42161, 10, 137, 130, 146, 480, 1329, 999.

## Same 403, as the operator sees it (Finding 13)

```
  3  web-search   ?  —  2103ms  ✗   ○
     GET https://api.agentstools.dev/search
     · no x402 or MPP challenge detected at https://api.agentstools.dev/search
```

```
selat-pay.mjs:1303   if (!hasX402 && !hasMpp && !upstreamFree) {
selat-pay.mjs:1304     throw new Error(`no x402 or MPP challenge detected at ${upstreamUrl}`);
```

`probeUpstream()` returns `{ status, x402, mpp, body }` (line ~1117) — the status is available
and dropped.

## The two candidates that did probe clean

```
"enrich a person by name and company"
  1  Person Data Enrichment — Ema..  $1.05  routed-x402   7617ms  ✓
     POST https://api.apify.com/v2/actors/ryanclinton~person-enrichment-lookup/run-sync-get-dataset-items

"crypto market news"
  1  Otto AI                       $0.0010  routed-x402  10416ms  ✓
     GET https://x402.ottoai.services/crypto-news
```

Both on allowlisted hosts. Two non-network failures on reachable hosts:

```
  Brave Search  · expected 402 from router, got 502:
                  {"error":"expected upstream 402 challenge, got 400"}   (brave.mpp.paywithlocus.com)
  Company Enrich · no x402 or MPP challenge detected                     (mpp.orthogonal.com)
```

`mpp.orthogonal.com` is named in selat-pay's own source comment (~line 1085) as a gateway that
validates the body before issuing 402 — the retry path exists but did not surface a challenge here.

## Merchant-host census (Finding 14)

Method — dump the catalog across eight broad intents, tally `endpoint.url` hostnames, match
against the full session-2 allowlist (wildcards as suffix matches):

```
for q in search enrich price news data image weather token; do
  selat search "$q" --top 400 --json > /tmp/c_$q.json
done
```

```
distinct merchant hosts seen: 526
  covered by handoff allowlist: 49
  NOT covered (would 403):     477
service-entries: total 1174  reachable 399  (34.0%)

top unreachable by entry count:
   8  agent402.tools                [agentic]
   7  gateway.apiosk.com            [agentic]
   7  api.delx.ai                   [agentic]
   7  x402.forgemesh.io             [agentic]
   7  api.strale.io                 [agentic]
   7  api.x402node.dev              [agentic]
   6  np.orthogonal.com             [circle,agentic]
   6  nano.blockrun.ai              [circle,agentic]
   6  api.24klabs.ai                [agentic]
   6  archtools.dev                 [agentic]
   6  clonecho.builda.company       [agentic]
   6  api.agentstools.dev           [agentic]
   6  2s.io                         [agentic]
   5  stableenrich.dev              [circle,agentic,mpp]
   5  vibesprings.net               [agentic]
   …477 total, long flat tail

reachable, by entry count:
 223  api.apify.com                 [apify]
  78  mpp.orthogonal.com            [mpp]
   9  x402.ottoai.services          [circle,agentic]
   4  apollo.mpp.paywithlocus.com   [mpp]
   4  alphavantage.mpp.paywithlocus.com [mpp]
   4  coingecko.mpp.paywithlocus.com    [mpp]
   3  x402.alchemy.com              [circle,mpp]
   3  api.nansen.ai                 [agentic,mpp]
   3  serpapi.mpp.tempo.xyz         [mpp]
   3  parallelmpp.dev               [circle,mpp]
   …49 total
```

## `selat doctor` — third session, still no network section (Finding 10, re-confirmed)

```
Binaries:            ✓ node v22.22.2   ✓ npm 10.9.7   ✓ git 2.43.0
Agent-payment skill: ✓ skill at …/@selat-ai/selat-discovery
Circle CLI:          ✗ circle CLI not installed — install Circle CLI, then run `selat init`
Agent Wallet:        ✗ no agent wallet found — run `selat init`
Spending policy:     ⚠ could not read the wallet spending policy
selat-pay:           ✓ selat-pay installed (bundled, v0.9.4)
Config:              ✗ /root/.config/selat-pay/.env missing or empty — run `selat init`
3 check(s) failed.
```

Run on a network where every SELAT host resolves and answers. `doctor` reports nothing about it —
and it does not check `SELAT_ROUTER_URL`, which is the one config value Finding 11 shows will
break a free command.

## `selat init` — non-interactive attempt fails at step 4 (Finding 15)

```
$ selat init --email <addr>            # stdin a pipe, no TTY
[1/8] Checking prerequisites          ✓ node v22.22.2
[2/8] Checking agent-payment skill    ✓ skill at …/@selat-ai/selat-discovery
[3/8] Checking Circle CLI             Circle CLI not found — installing @circle-fin/cli…
                                      added 164 packages in 26s
                                      ✓ Circle CLI installed
[4/8] Circle Agent Wallet login
      ✗ not logged in, and this shell has no TTY for the email/OTP login.
      Log in from an interactive terminal first: circle wallet login <addr> --type agent
      (the Circle CLI prompts for the 6-digit code), then re-run `selat init` here.
[[EXIT 1]]
```

## `selat init` under a pty — completes (Finding 15 workaround)

```
$ tail -f /tmp/otp_feed | script -qfe -c "selat init --email <addr>" /tmp/init2.raw
[4/8] Circle Agent Wallet login
      A login code will be sent to <addr>.
      The Circle CLI will prompt for the 6-digit code.
Enter the 6-digit OTP from your email after D3N-: ******
Logged in as <addr>
      ✓ logged in as <addr>
[5/8] Creating agent wallets
      ✓ 3 agent wallets found on this Circle account
      Checking Gateway balances…
      1. 0x01224a287d5cbf9bfbd9cec6f93007a661062aac  Gateway: 0.000000 USDC  (default)
      2. 0x0af826448d204ee0f990d4518a6b5d6797e3fc93  Gateway: 0.000000 USDC
      3. 0x7056a1ccc589d6c2fccc23e61360a0745bdfce32  Gateway: 0.000000 USDC
▸ Wallet to use [1-3/new] (1) 1
      ✓ wallet 0x01224a287d5cbf9bfbd9cec6f93007a661062aac
      across 3 Circle-supported chains
[6/8] Checking selat-pay              ✓ selat-pay installed (bundled with selat-cli)
[7/8] Writing config                  ✓ /root/.config/selat-pay/.env (mode 0600)
      SELAT_ROUTER_URL=https://router.selat.ai
      SELAT_AGENT_WALLET_ADDRESS=0x01224a287d5cbf9bfbd9cec6f93007a661062aac
[8/8] Funding check
      No USDC yet — fund before paid calls: selat fund --amount 2

⚠ This wallet has NO spending caps — a runaway agent could spend the full balance.
      Circle wallet policy (per-tx/daily/weekly/monthly) is the one hard ceiling
      the agent literally cannot bypass. Strongly recommended before funding.
▸ Set spending caps now? [Y/n] (y) y

▸ Per-transaction cap (USDC) (5) 0.50
▸ Daily cap (USDC) (50) 2
▸ Weekly cap (USDC) (200) 2
▸ Monthly cap (USDC) (500) 2

⚠ Circle will send a one-time code to your email.
Enter the code at the prompt below. This is Circle's policy-write security layer.
Enter the 6-digit OTP from your email after CQJ-: ******
Policy updated for 0x01224a287d5cbf9bfbd9cec6f93007a661062aac on BASE.
┌─────────────────────┬────────┬───────┬────────┬─────────┬────────┐
│ Policy Type         │ Per-Tx │ Daily │ Weekly │ Monthly │ Origin │
├─────────────────────┼────────┼───────┼────────┼─────────┼────────┤
│ STABLECOIN_TRANSFER │   0.50 │     2 │      2 │       2 │ CUSTOM │
└─────────────────────┴────────┴───────┴────────┴─────────┴────────┘
✓ policy set

You're ready.
Script done — [COMMAND_EXIT_CODE="0"]
```

Two distinct email OTPs: `D3N-` to log in, `CQJ-` to write the policy.
Policy write reports `on BASE` only, though the wallet spans 3 chains.

## `selat fund --help` never prints help (Finding 16)

```
$ selat fund --help
✗ no TTY to prompt for the deposit amount — re-run with --amount <usdc> (and --yes to confirm the deposit)
$ selat fund -h
✗ no TTY to prompt for the deposit amount — re-run with --amount <usdc> (and --yes to confirm the deposit)
$ script -qec "selat fund --help" /dev/null      # under a pty
<hangs on the amount prompt; killed at 180s>
```

`search`, `skill compare`, `init` and `doctor` all handle `--help` correctly.

## `selat doctor` after init — gains a network section (Finding 17)

```
Binaries:            ✓ node v22.22.2   ✓ npm 10.9.7   ✓ git 2.43.0
Agent-payment skill: ✓ skill at …/@selat-ai/selat-discovery
Circle CLI:          ✓ circle binary on PATH
                     ✓ authenticated as Vickmancrypto@gmail.com
Agent Wallet:        ✓ wallet 0x01224a287d5cbf9bfbd9cec6f93007a661062aac
                     ⚠ Gateway balance: 0 USDC (run `selat fund`)
                     ⚠ on-chain USDC base: 0.00
                     ⚠ on-chain USDC optimism: 0.00
                     ⚠ on-chain USDC arbitrum: 0.00
                     ⚠ on-chain USDC polygon: 0.00
                     ⚠ No on-chain USDC and Gateway is empty — run `selat fund`
Spending policy:     ✓ capped at $0.5/tx · $2/day · $2/wk · $2/mo
selat-pay:           ✓ selat-pay installed (bundled, v0.9.4)
Config:              ✓ /root/.config/selat-pay/.env present
                     ✓ SELAT_ROUTER_URL=https://router.selat.ai
                     ✓ SELAT_AGENT_WALLET_ADDRESS=0x01224a287d5cbf9bfbd9cec6f93007a661062aac
Router reachability: ✓ https://router.selat.ai/healthz returns 200

All checks passed.
```

`Router reachability` is the only network probe, and it is absent pre-init. Six ⚠ lines above
`All checks passed.` The same wallet-absent condition printed `3 check(s) failed.` before init.
