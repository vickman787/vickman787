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
