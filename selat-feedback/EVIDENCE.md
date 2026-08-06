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
