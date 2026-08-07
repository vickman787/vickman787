# SELAT — session feedback (install → discovery, blocked before payment)

**Harness:** Claude Code (remote container via claude.ai/code)
**OS:** Linux 6.18.5 x86_64 · Node v22.22.2 · npm 10.9.7
**Versions:** `@selat-ai/selat-cli` 0.15.7 · plugin `selat@selat-plugins` 0.1.8 · bundled `selat-pay` 0.9.4
**Date:** 2026-08-06

## TL;DR

Install worked first try. `selat skill list --available` worked. **`selat search` did not**,
and the error message told me to fix it in a way that would not have fixed it. I never got to a
paid call, for two independent reasons: (a) this environment's egress allowlist blocks
`*.selat.ai`, `api.circle.com`, and `api.apify.com`; (b) the wallet/funding steps need a human's
email OTP and real USDC, which an ephemeral container should not be holding anyway.

What follows is the part I think is actually useful to you: five things that surprised me, one of
which is a straightforward doc/code bug you can fix today.

---

## What worked

`claude plugin marketplace add SELAT-AI/selat-plugins` + `claude plugin install selat@selat-plugins`
— clean, no prompts, no ambiguity. `npm i -g @selat-ai/selat-cli` likewise. `selat --help` is
genuinely well-written: the distinction between `search` (free) and `run` (pays) is stated where
you actually read it, and `--dry-run` is advertised next to `run` rather than buried. Good.

`selat skill list --available` returned 19 skills with per-skill reliability dots and a
"checked 8h ago" timestamp. The reliability indicator is the single best thing I saw in the whole
CLI — it is the thing that would make me trust spending money.

---

## Finding 1 — the remediation text names the wrong hosts (reproducible)

`selat search "web search"` fails on **`api.apify.com`**, which is the *first* host the federated
catalog loader touches:

```
Loading federated catalog (refresh=false)...
Fatal: 403 https://api.apify.com/v2/store?limit=500&offset=0&allowsAgenticUsers=true&sortBy=totalUsers:
       Host not in allowlist: api.apify.com.
```

The remediation block printed directly underneath says:

> SELAT discovery needs api.circle.com + router.selat.ai. Allowlist them in .cursor/sandbox.json:
> `{ "networkPolicy": { "default": "deny", "allow": ["api.circle.com","*.selat.ai","registry.npmjs.org","*.npmjs.org"] } }`

**`api.apify.com` is not in that list.** A user who follows this advice verbatim re-runs the
command and hits the exact same error. The same incomplete allowlist is in
`selat-plugins/install.md` (Cursor section) and `guides/cursor.md`.

Grepping the shipped `@selat-ai/selat-discovery` package, the catalog hosts are:

```
4  https://api.apify.com
2  https://catalog.selat.ai
2  https://api.circle.com
1  https://agi.apify.com
```

So the allowlist should be at minimum
`["api.circle.com","*.selat.ai","*.apify.com","registry.npmjs.org","*.npmjs.org"]`.
Also note the prose says `router.selat.ai` while the code calls `catalog.selat.ai` — the wildcard
covers both, but the prose is describing a host the discovery path doesn't actually use.

**Fix:** derive the printed allowlist from the same constant list the loader reads, so it can't
drift again.

## Finding 2 — the error is much worse behind a CONNECT proxy

Same command, same block, but routed through an HTTPS proxy instead of a transparent filter:

```
Loading federated catalog (refresh=false)...
Fatal: fetch failed
```

That's it. No host, no URL, no status. I only knew which host to blame in Finding 1 because the
transparent filter happened to return an HTTP 403 with a readable body. Behind a normal corporate
proxy the CONNECT is rejected at the tunnel and `undici` throws a bare `TypeError: fetch failed`,
and SELAT surfaces it unchanged.

**Fix:** wrap catalog fetches and re-throw with the host you were dialing — `Fatal: could not reach
api.apify.com (fetch failed)`. One line, and it turns a dead end into a diagnosable failure. This is
the difference between "SELAT is broken" and "my network blocks apify".

## Finding 3 — the sandbox advice is Cursor-specific, but prints in every harness

I am running in Claude Code. The error told me to create `.cursor/sandbox.json`. There is no Cursor
here. The plugin already knows which harness it's installed under (there are separate
`.claude-plugin/`, `.cursor-plugin/`, `.codex-plugin/` manifests in the repo), so the hint could be
harness-aware — and for hosted/remote harnesses the honest advice is "your environment has an egress
policy; these are the hosts to allow," not a Cursor config file path.

## Finding 4 — "free discovery" is two commands with very different network footprints

This isn't documented anywhere I could find, and it's worth a sentence in the README:

| Command | Reads from | Worked here |
|---|---|---|
| `selat skill list --available` | `raw.githubusercontent.com` (`SELAT_SKILLS_RAW_BASE`) | ✅ yes |
| `selat search <intent>` | `api.apify.com`, `catalog.selat.ai`, `api.circle.com` | ❌ blocked |

Both are described as free, no-wallet discovery, so I expected them to succeed or fail together.
That one worked and the other didn't sent me reading package internals for ten minutes trying to
figure out whether I'd mis-installed something. Saying "skill list works anywhere GitHub is
reachable; search additionally needs the catalog hosts" would have saved that entirely.

## Finding 5 — the one thing I expected SELAT to do and it didn't

**`selat doctor` does not check network reachability.**

It checks binaries, the skill package, the Circle CLI, the wallet, `selat-pay`, and config — all
local — then reports "3 check(s) failed" about wallet setup. Every one of those is a *wallet* problem.
Meanwhile the actual thing standing between me and a working `selat search` — three unreachable
hosts — got no mention at all, because `doctor` never dials anything.

I ran `doctor` first, exactly as `install.md` tells you to, and it gave me a clean bill of health on
everything except the wallet I hadn't set up yet. It pointed me at `selat init`, which would have
failed on `api.circle.com` for the same reason. The command whose entire job is "diagnose setup
problems" is blind to the most common class of setup problem in sandboxed and corporate environments.

**Fix:** add a `Network:` section that HEADs each required host and prints per-host reach/block. That
one change would have resolved this whole session in a single command, and it would have produced
Finding 1's correct allowlist as a by-product.

## Finding 6 — the skill-contribution path is unreachable from a locked-down network

For step 8: `meta/skill-creator` is well written and the `serviceUrl` vs `url` warning ("read this
twice") is clearly hard-won. The merge gate is `selat skill verify`, which probes each step's live
402 challenge. Credit where due: **the gating receipt comes from the free probe, not `--pay`** — a
contributor needs no funded wallet to produce a mergeable PR, and the skill-creator frontmatter says
so explicitly. Good call, and worth advertising louder than it currently is.

The remaining constraint is purely network: verify must reach the merchant `serviceUrl` hosts, and
authoring at all requires the catalogue (`api.apify.com` / `catalog.selat.ai`) to choose endpoints
from. So behind an egress policy a contributor can scaffold and statically validate but cannot
verify — and the failure will surface as the same nameless `fetch failed` from Finding 2, at which
point they cannot tell an unreachable merchant from a genuinely dead endpoint. Fixing Finding 2
largely fixes this one too.

---

## Where I stopped, and why

I did not set up a wallet, fund it, or make a paid call. Two reasons, both worth you knowing:

1. **Network.** `api.circle.com`, `*.selat.ai`, `api.apify.com` are all blocked by this
   environment's egress policy. `selat init` would fail at the Circle login step.
2. **Judgment.** This is an ephemeral container that gets reclaimed on idle. Creating a
   Circle Agent Wallet here means the wallet's session state dies with the container, and any
   funded balance would be stranded behind an email OTP I can't complete on the user's behalf.
   Sending real USDC to a wallet living in a disposable sandbox is a bad idea regardless of
   whether SELAT works, and self-custody makes it *more* the user's problem, not less.

On the "what made you nervous about letting your agent spend money" question, honestly: the
nervousness isn't about SELAT's confirmations, which are fine. It's that `selat run "<intent>"`
compresses discover → rank → pick → settle into one command, and the thing being paid is chosen by
a ranker I can't inspect before it charges me. `--dry-run` exists and is advertised, which helps a
lot. What I'd want on top of that is a persisted per-session cap that is on by default rather than
opt-in via `setup-policy` — `freeze`/`unfreeze` are a kill switch after the fact, and `budget`
reports caps that `setup-policy` has to have set first. Default-deny above some trivial amount would
let me stop thinking about it.

## Reproduction

See `EVIDENCE.md` in this directory for raw terminal output of every command above.

```bash
npm i -g @selat-ai/selat-cli          # 0.15.7
selat doctor                          # local-only checks, no network probe
selat skill list --available          # works (raw.githubusercontent.com)
selat search "web search"             # fails on api.apify.com
```

---

# Session 2 — the allowlist was fixed, and discovery still failed

The user reconfigured the environment's egress policy to allow the hosts session 1 identified,
and I re-ran from a fresh container. `selat search` still fails. That turned out to be more
interesting than a working run would have been: it produced a much sharper version of Finding 1,
and two new findings that I think are the most actionable things in this document.

Nothing below is speculative — every claim has a line number in the shipped 0.15.7 package or
raw terminal output in `EVIDENCE.md`.

## Finding 7 — one unreachable registry zeroes out all discovery, and the fix is already in the file

`selat search` fans out to **five** independent catalog registries. Four of them are bare entries
in a `Promise.all`, so the first rejection sinks the entire catalog — including the four registries
that answered fine.

`discover_federated_catalog.mjs:1102`:

```js
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

You already know this is a problem — that comment says so, and the `.catch` is exactly the right
shape. It's just applied to one of five. In my run, `api.circle.com`, `api.apify.com`, and
`catalog.selat.ai` were all reachable and `api.cdp.coinbase.com` was not, and I got **zero**
results rather than three registries' worth.

This is not only a sandbox problem. Any one of five third-party registries having a bad ten
minutes takes `selat search` and `selat run` down for every user, everywhere. Five dependencies
wired in series is five times the outage surface of one, and none of the four is load-bearing
enough to deserve that.

**Fix:** give sources 1–4 the same `.catch` source 5 already has (or `Promise.allSettled` and
partition), then print what degraded — `catalog: 4/5 sources (agentic unreachable)`. A partial
catalog is enormously more useful than a `Fatal:`, and the warn line keeps it honest. This is the
single highest-leverage change in this document; it's a handful of lines and it converts a hard
failure into a soft one.

## Finding 8 — Finding 1, confirmed the hard way: the documented allowlist names 2 of 5 catalog hosts

Session 1 reported that the printed allowlist was missing `api.apify.com`. I flagged the risk of
drift. Session 2 is that drift, observed: I ran on a network configured to the documented
allowlist and discovery still died, on a host the docs have never mentioned.

The constant, `lib/host.mjs:83`:

```js
// The catalog hosts SELAT discovery must reach. Mirrors guides/cursor.md +
// install.md so the in-CLI hint and the docs stay in lockstep.
const SANDBOX_ALLOW = ["api.circle.com", "*.selat.ai", "registry.npmjs.org", "*.npmjs.org"];
```

The actual registries, from the discovery scripts:

| Registry | Host | In `SANDBOX_ALLOW`? |
|---|---|---|
| Circle | `api.circle.com` | ✅ |
| SELAT-native | `catalog.selat.ai` | ✅ (via `*.selat.ai`) |
| x402 Bazaar | `api.cdp.coinbase.com` | ❌ |
| MPP | `mpp.dev` | ❌ |
| Apify | `api.apify.com`, `agi.apify.com` | ❌ |

Two of five. And `install.md:64` states the wrong pair as fact:

> `api.circle.com` + `*.selat.ai` are the catalog hosts discovery needs

The comment above the constant says it "mirrors guides/cursor.md + install.md so the in-CLI hint
and the docs stay in lockstep" — and it does. All three are in lockstep and all three are wrong,
which is what a hand-maintained mirror buys you. The most recent commit on `selat-plugins` is
`chore/x402-bazaar-rename`; the Bazaar registry got renamed and the allowlist that has to know
about it didn't move. That's the drift, one commit old.

**Fix:** the same one as Finding 1, and it now has a second reason to happen. Export the host list
from the discovery scripts that own it, build `SANDBOX_ALLOW` from that, and generate the docs
snippet from the same export. The correct list today is:

```json
["api.circle.com", "*.selat.ai", "api.cdp.coinbase.com", "mpp.dev",
 "*.apify.com", "registry.npmjs.org", "*.npmjs.org"]
```

## Finding 9 — the egress hint is gated on a probe that goes silent exactly when it's needed

This is the one I'd fix first after Finding 7, because it actively misleads.

The remediation block from Finding 1 didn't print in session 2. Not because it was fixed —
because it was suppressed. `lib/host.mjs:109`:

```js
export async function egressLikelyBlocked() {
  const r = await fetch("https://api.circle.com/v2/x402/discovery/resources", { ... });
  return r.status === 403 || r.status === 407;
}
```

One host. `api.circle.com`. And all three call sites gate the hint on it:

```
lib/commands/search.mjs:109   if (await egressLikelyBlocked()) console.error(sandboxHintText());
lib/commands/skill.mjs:518    if (await egressLikelyBlocked()) console.error(sandboxHintText());
lib/commands/run.mjs:124      if (await egressLikelyBlocked()) console.error(sandboxHintText());
```

`api.circle.com` is the first host in `SANDBOX_ALLOW`, so it is the host a user is *most likely
to have already allowed*. Once they do, it answers 307, the probe returns `false`, and the hint
disappears — while discovery stays broken on `api.cdp.coinbase.com` and `mpp.dev`.

So the diagnostic is designed to fail in precisely the population it exists to serve: users who
read the docs and followed them partway. Do nothing and you get the hint. Follow the instructions
and you lose it. Session 1 got the hint (nothing was allowed); session 2 followed the guidance and
got a bare `Fatal:` line.

The docstring's stated principle is right — "fail-safe: any ambiguity resolves to `false` (don't
cry wolf)" — but a single sentinel host isn't a fail-safe, it's a coin flip on which host the
user happened to allow first.

**Fix:** probe all five catalog hosts concurrently and report per-host, rather than reducing them
to one boolean:

```
Catalog hosts:
  ✓ api.circle.com          ✓ catalog.selat.ai       ✓ api.apify.com
  ✗ api.cdp.coinbase.com    ✗ mpp.dev
Allowlist the two blocked hosts in your egress policy.
```

That is strictly more informative than the current boolean, it can't go silent when it's needed,
and it can't drift from the real host list if it's derived from Finding 8's shared export.

## Finding 10 — `selat doctor` is still network-blind, now confirmed against a live network

Session 1 reported this from a fully-blocked network, where it could be argued the probe would
have failed anyway. Session 2 removes that caveat: on a network where five of seven SELAT hosts
answer and two don't, `doctor` reports the same three wallet failures and says nothing about the
two blocked hosts that are the only thing actually stopping me.

What makes this worth repeating rather than just re-filing: **`lib/host.mjs` already contains a
working reachability probe**. `egressLikelyBlocked()` is the mechanism `doctor` is missing. It's
imported by `search`, `skill`, and `run` — every command except the one whose entire job is
diagnosing setup. Generalize it to the full host list per Finding 9 and call it from `doctor`, and
Findings 8, 9 and 10 all close with one shared function.

I want to be clear about the cost of it being missing, because it's the whole story of this run:
two sessions, two container rebuilds, and a round-trip through the user to change an environment
policy — and both times the thing that told me which host to allow was `curl` and `grep` through
`node_modules`, not SELAT. A `Network:` section in `doctor` would have collapsed both sessions
into one command.

## What this changes about the earlier findings

Finding 2 (bare `fetch failed` behind a CONNECT proxy) stands, and Finding 7 raises its stakes:
with sources 1–4 in a `Promise.all`, a nameless `fetch failed` from any one of them takes down
discovery with no indication of which of five hosts to investigate. Fix Finding 7 and even an
unnamed failure degrades to a warning line next to four working registries.

Finding 3 (Cursor-specific advice in every harness) also stands, and Finding 9 adds a wrinkle:
the advice is not just harness-wrong, it's conditional on a probe that misfires. On this run the
Cursor-flavoured hint was *correctly* suppressed, but for the wrong reason — not "you're not in
Cursor", just "`api.circle.com` happened to answer".

## Status

Discovery (bounty step 2) is blocked pending `api.cdp.coinbase.com` and `mpp.dev` being added to
the environment's egress allowlist. Wallet, funding, and paid calls (steps 3–6) are not started;
`api.circle.com` is reachable, so `selat init` is not itself blocked, but it needs the user's
email and an OTP they have to type, and funding needs their USDC.

---

# Session 3 — the allowlist finally held, discovery ran, and the real bug surfaced

**Date:** 2026-08-06 · `@selat-ai/selat-cli` **still 0.15.7** (no release between sessions 1–3)

All six hosts from the session-2 handoff are reachable this run, including the two that were
missing (`api.cdp.coinbase.com`, `mpp.dev`). `selat search` works. Findings 7–10 are therefore
confirmed-and-closed from the outside: the fan-out is fatal exactly as described, and widening
the allowlist is what fixed it.

That unblocked step 2 — and step 2 immediately produced a better bug than anything in sessions
1–2. **Findings 11 and 12 are the two I would actually fix.** Finding 12 in particular is a
one-line change that would have prevented all three sessions of this bounty run from stalling.

## Finding 11 — `selat-pay` documents a default router URL and then doesn't apply it

`selat skill compare` is advertised as free and wallet-free: *"free-probes each candidate's live
402 at its catalog serviceUrl (never settles)"*. On a clean install it fails on every candidate:

```
1  Automaton Webpage Change Mon..   ?  —  1116ms  ✗   · missing --router-url or SELAT_ROUTER_URL
2  Wikipedia Article Summary        ?  —  1135ms  ✗   · missing --router-url or SELAT_ROUTER_URL
3  YouTube Summary & Transcript     ?  —  1218ms  ✗   · missing --router-url or SELAT_ROUTER_URL
```

Every column the command exists to fill — PRICE, RAIL — renders `?` and `—`. The cause is two
lines apart in `selat-pay.mjs`:

```js
1192:  const routerUrl = (args.routerUrl ?? process.env.SELAT_ROUTER_URL ?? "").replace(/\/$/, "");
1210:  if (!routerUrl) throw new Error("missing --router-url or SELAT_ROUTER_URL");
```

There is no fallback. Meanwhile the value is documented as a *default* in three shipped files:

```
selat-cli/README.md:198            SELAT_ROUTER_URL=https://router.selat.ai   # default SELAT Router
selat-pay/README.md:36             SELAT_ROUTER_URL=https://router.selat.ai
selat-cli/lib/commands/init.mjs:248   SELAT_ROUTER_URL: routerUrl,
```

`init.mjs` writes it into `~/.config/selat-pay/.env` — so the "default" only materialises after
`selat init`, which requires a wallet. The result is that a **free, no-spend, no-wallet command is
gated behind wallet creation for no functional reason.** Exporting the documented value by hand
fixes it completely:

```
SELAT_ROUTER_URL=https://router.selat.ai selat skill compare "summarize a webpage" --limit 3
```

**Fix:** `?? "https://router.selat.ai"` at line 1192. Or have `skill compare` fall back to it,
since it never settles.

## Finding 12 — the 402 probe goes direct to the merchant, but the router already proxies it

This is the important one.

`selat-pay` talks to two different places for the same call. Settlement goes through the router:

```js
1339:  const targetForPayment = `${routerUrl}/proxy?target=${encodeURIComponent(upstreamUrl)}`;
1394:  routerProbe = await fetch(targetForPayment, …);   // discovery of the challenge, routed
1561:  paidRes     = await fetch(targetForPayment, …);   // settlement, routed
```

But the *detection* probe that decides whether a service is payable at all goes straight to the
merchant:

```js
1097:  const res = await fetch(upstreamUrl, …);   // probeUpstream() — direct, unrouted
```

So `selat search` / `selat skill compare` require **direct egress to every merchant domain in the
catalog**, even though SELAT operates a proxy that already reaches them and is already in the code
path a few lines later.

I verified the router proxy reaches a host this environment blocks:

```
$ curl -s -o /dev/null -w '%{http_code}' https://api.agentstools.dev/search
403        Host not in allowlist: api.agentstools.dev

$ curl -sD - "https://router.selat.ai/proxy?target=https%3A%2F%2Fapi.agentstools.dev%2Fsearch"
HTTP/2 402
payment-required: eyJ4NDAyVmVyc2lvbiI6MiwicmVzb3VyY2UiOnsidXJsIjoiL3Byb3h5P3Rhcmdl…
```

That decodes to a complete x402 v2 challenge — `amount: "1050"` ($0.00105 USDC) across 10 chains,
`payTo 0x1E5B…c5C2`. **The blocked host is fully probeable through SELAT's own router.**

**Fix:** route `probeUpstream()` through `${routerUrl}/proxy?target=…` — the URL the function 240
lines below it already builds. It is free (402 challenges never settle) and it collapses SELAT's
egress requirement from *the entire merchant long tail* to **one host: `router.selat.ai`**.

Three sessions of this bounty were spent grepping `node_modules` for hostnames to hand the user
for their allowlist. Finding 12 makes that entire exercise unnecessary.

## Finding 13 — a network 403 is reported as a merchant protocol defect

When the direct probe from Finding 12 is blocked, this is what the operator sees:

```
3  web-search   ?  —  2103ms  ✗
   GET https://api.agentstools.dev/search
   · no x402 or MPP challenge detected at https://api.agentstools.dev/search
```

The actual response was `403 Host not in allowlist: api.agentstools.dev` — a network policy
decision, from the proxy, about the *operator's* environment. The message blames the merchant for
not speaking x402.

`probeUpstream()` captures `res.status` and returns it, but the caller discards it:

```js
1303:  if (!hasX402 && !hasMpp && !upstreamFree) {
1304:    throw new Error(`no x402 or MPP challenge detected at ${upstreamUrl}`);
```

An inline comment three lines up shows this is deliberate — *"401/403 are auth answers, not body
validation"* — which is true for the retry logic and wrong for the error message. 403 is the one
status that most often is not the merchant's fault.

Across five intents this misattribution fired on **19 of 23 candidates**. An operator reading that
table concludes the SELAT catalog is full of broken listings. It isn't; their egress policy is
narrow. This is Findings 2 and 9 (bad network diagnostics) recurring a third time, one layer down.

**Fix:** include the status: `` `no x402 or MPP challenge detected at ${url} (HTTP ${status})` ``,
and special-case 403 with the response body, which in this case names the exact host to allow.

## Finding 14 — the merchant host set is unbounded, so allowlisting is not a viable strategy

To size Finding 12's blast radius I dumped the catalog across eight broad intents
(`selat search "<q>" --top 400 --json`, q ∈ search/enrich/price/news/data/image/weather/token) and
tallied `endpoint.url` hostnames against the full allowlist from the session-2 handoff:

```
distinct merchant hosts seen:      526
  covered by the allowlist:         49
  NOT covered (would 403):         477   (91%)
service entries:  1174 total,  399 reachable  (34.0%)
```

The 49 reachable hosts are almost entirely three wildcard families — `api.apify.com` (223 entries),
`mpp.orthogonal.com` (78), `*.mpp.paywithlocus.com` and `*.mpp.tempo.xyz` (~45 combined). The 477
unreachable ones are a flat long tail, mostly from the `agentic` (x402 Bazaar) source, with a
median of ~2 entries each: `agent402.tools`, `api.delx.ai`, `x402.forgemesh.io`, `api.strale.io`,
`api.x402node.dev`, `api.24klabs.ai`, `clonecho.builda.company`, `2s.io`, …

Every merchant registers its own domain. The set grows every time someone lists a service. No
allowlist can track it, and the top-ranked result for a given intent is more likely than not to sit
on a host nobody has ever heard of. **Any egress-restricted environment — a corp network, a CI
runner, an agent sandbox — cannot use SELAT discovery as currently built.** That is the same
conclusion as Finding 12, arrived at from the data instead of the source.

## Step 2 (discovery) — what the ranking actually surfaced

This is the graded part of the bounty, so: **the ranking is good.** Better than I expected.

`selat search "web search"` merged **2596 services from all 5 catalogs** (raw 2670 → deduped),
`690 matched a token, 88 on-target`, and returned a sensible top 5 ordered by a blend of name
match and price. The `why:` line on each row (`matched web, search in tags+name · $0.0010/call`)
is the right idea — it makes the ranking auditable instead of magic, and I could tell at a glance
when a match was name-only versus tag-corroborated. The hidden-match count
(`602 weaker description-only matches hidden`) is honest about the tail rather than pretending
the top 5 is the whole story.

`selat skill list --available` returned the same 19 skills as session 1, with reliability dots and
`checked 9h ago`. Still the best-designed surface in the CLI.

Two gaps worth naming:

1. **Ranking is payability-blind.** Nothing in `selat search` scoring accounts for whether a
   service can currently be paid. Sorting is name-match × price. `--explain` is documented as
   showing "why each match is or isn't payable right now", but the ranking itself doesn't use it,
   so the top result is routinely one that cannot be reached or settled. Feeding the
   `skill compare` probe result — or the registry reliability dot that `skill list` already
   has — back into `search` ordering would be a large quality win.
2. **`1/5 catalogs` on nearly every result.** Almost nothing is corroborated across registries,
   so the count is near-constant and carries little signal. Where it *does* vary
   (`stableenrich.dev` at `[circle,agentic,mpp]`) that's genuinely useful — a service three
   registries independently list is a better bet. Worth surfacing more prominently than the top-5
   name match.

Measured payability, five intents, `--limit 5` (23 candidates probed):

| Outcome | n | |
|---|---|---|
| live 402, priced | **2** | Apify person-enrichment $1.05 · Otto AI crypto-news $0.0010 |
| egress-blocked, misreported as "no challenge" (Finding 13) | 19 | |
| reachable host, router 502 `expected upstream 402, got 400` | 1 | `brave.mpp.paywithlocus.com` |
| reachable host, body validated before 402 | 1 | `mpp.orthogonal.com` |

Both successes are on allowlisted hosts. Every failure is explained by Finding 12 or by a
merchant-side body-validation quirk that `probeUpstream()` already has retry logic for. **The
catalog is not the problem; the direct probe is.**

## Status

Steps 1–2 (install, discovery) are **complete**. Findings 11–14 are new this session; 7–10 are
confirmed. Steps 3–6 (wallet, funding, paid calls, verify) are unblocked network-wise —
`api.circle.com` answers and the reachable-merchant list above has enough live endpoints to clear
the ≥3-endpoint / ≥0.5 USDC bar — but they are waiting on the user for an email, an OTP, and USDC.

---

# Session 3 (cont.) — wallet created, and three more findings from the init path

Step 3 is **done**. `selat init` completed exit 0: logged in as Vickmancrypto@gmail.com, wallet
`0x01224a287d5cbf9bfbd9cec6f93007a661062aac`, config at `/root/.config/selat-pay/.env` (0600),
and a Circle spending policy of **$0.50/tx · $2/day · $2/wk · $2/mo**. Two separate email OTPs
were required — one to log in, one to write the policy.

The policy prompt is the best safety design in the product. It is offered unprompted, before
funding, it explains *why* it matters in one sentence ("the one hard ceiling the agent literally
cannot bypass"), and it defaults to yes. Every agent-payments tool should do this.

## Finding 15 — `selat init`'s non-interactive flags cannot complete an init

`selat init --help` advertises two flags explicitly for headless use:

```
  --email <address>   Circle agent account email (skips the login email prompt).
  --wallet <n|address|new>
                      … For agent shells / CI with no TTY.
```

Both are honoured. Init still dies, at step 4 of 8:

```
[3/8] Checking Circle CLI
      ✓ Circle CLI installed
[4/8] Circle Agent Wallet login
      ✗ not logged in, and this shell has no TTY for the email/OTP login.
      Log in from an interactive terminal first: circle wallet login <email> --type agent
      (the Circle CLI prompts for the 6-digit code), then re-run `selat init` here.
[[EXIT 1]]
```

The flags skip SELAT's *own* prompts, but step 4 shells out to the Circle CLI, which opens its own
TTY-only prompt. So the documented CI path stops one step past the flags it documents. `--email`
in particular reads as though it makes login headless; it only pre-fills the address.

Worse, the remediation pushes the operator to run `circle wallet login` **directly** — the raw
Circle CLI — which is exactly what SELAT's own docs tell agents not to touch. An operator
following this text ends up outside the abstraction the product is selling.

**Workaround** (what I did): give `selat init` a pty instead, so it drives the Circle login itself
and the operator never touches `circle`:

```bash
tail -f /tmp/otp_feed | script -qfe -c "selat init --email <addr>" /tmp/init.raw &
# then write each answer to /tmp/otp_feed as its prompt appears
```

**Fix:** either pass the OTP through (`--otp`, or read it from stdin when not a TTY), or make the
remediation say "re-run under `script -c`" rather than "go use the Circle CLI directly".

## Finding 16 — `selat fund --help` doesn't print help, on the one command that moves money

```
$ selat fund --help
✗ no TTY to prompt for the deposit amount — re-run with --amount <usdc> (and --yes to confirm the deposit)
$ selat fund -h
✗ no TTY to prompt for the deposit amount — re-run with --amount <usdc> (and --yes to confirm the deposit)
```

Under a pty it's worse — `--help` is ignored and the command sits on the amount prompt waiting for
a number. `fund` reaches for the amount before it parses `--help`, so the flag never gets handled.

Every other command I ran (`search`, `skill compare`, `init`, `doctor`) handles `--help` correctly.
`fund` is the one that moves USDC, and it is the one whose documentation you cannot read without
either guessing flags or reading the source. I only know `--amount`, `--yes`, `--wait`, `--chain`
and `--method eco` from the error strings and from other commands' output.

**Fix:** handle `--help`/`-h` before the TTY check. One line, in the command that most needs it.

## Finding 17 — correction to Finding 10: `doctor` *does* check the network, but only the router

Sessions 1–3 reported `selat doctor` as network-blind. That was measured pre-init, and it was
incomplete. **Post-init, `doctor` grows a network section:**

```
Router reachability:
  ✓ https://router.selat.ai/healthz returns 200
```

So the check exists — it is just gated behind `selat init` writing `SELAT_ROUTER_URL`, which means
it is absent for the entire window in which a new user is most likely to be debugging egress. The
substance of Finding 10 survives, narrowed:

1. The one host `doctor` probes is `router.selat.ai` — the host that, per Finding 12, is the *only*
   one that ought to need reaching. It does not probe any of the five catalog registries
   (`catalog.selat.ai`, `api.circle.com`, `*.apify.com`, `api.cdp.coinbase.com`, `mpp.dev`) whose
   failure is what actually kills discovery (Finding 7). A user whose `selat search` returns
   nothing gets a green `doctor`.
2. It does not check `SELAT_ROUTER_URL` is *set* before init — the exact gap that makes the free
   `skill compare` fail (Finding 11).
3. It prints **`All checks passed.`** while four ⚠ lines about a zero balance sit above it.
   Pre-init it printed `3 check(s) failed.` for the same wallet-absent condition. The summary
   line and the body disagree.

Adding the five registries to `doctor`'s probe list — or, better, fixing Finding 12 so only the
router matters — would have saved all three sessions of this run.

## Status

Steps 1–3 complete. Step 4 (funding) is the next action and needs the user to send USDC to
`0x01224a287d5cbf9bfbd9cec6f93007a661062aac`. Steps 5–6 follow immediately after.

## Finding 18 — the body-validation retry exists on the direct probe but not the routed one

`probeUpstream()` carries an explicit workaround for gateways that validate the request body
*before* issuing a 402, and names the offender in a source comment (~`selat-pay.mjs:1085`):

> *Some gateways validate the request body BEFORE issuing the 402 challenge (observed live:
> `mpp.orthogonal.com` and `api.aisa.one/v1` answer an empty/absent body with a 4xx naming the
> missing fields, 402 only once they're present). On such a 4xx we parse the named fields and
> retry ONCE with sampleValue placeholders.*

That retry lives on the **direct** probe path. The **routed** probe at line 1394 has no equivalent,
so the same merchants fail there with the router's passthrough of the merchant's 400:

```
$ selat-pay POST https://mpp.orthogonal.com/nyne/person/newsfeed --chain base --probe-only
[selat-pay] error: expected 402 from router, got 502: {"error":"expected upstream 402 challenge, got 400"}   mode=routed-mpp
```

I probed 11 catalog endpoints on `mpp.orthogonal.com` priced $0.10–$0.44 — the single densest
reachable host in the catalog after Apify, at 78 entries. **One** produced a live 402. Six failed
with the 502-wrapping-400 above; four with `no x402 or MPP challenge detected`. The retry that
would have fixed most of them is implemented, tested against this exact host, and simply not
reached on the path `--probe-only` takes.

Combined with Finding 12 this is the same shape twice: the routed path is missing logic the direct
path has, so whichever one you need is the one that's incomplete.

## Probe census — what is actually payable from an egress-restricted host

42 catalog endpoints free-probed across every reachable merchant family, all priced ≥ $0.01:

| Family | probed | live 402 |
|---|---|---|
| `*.mpp.tempo.xyz` | 8 | **6** |
| `parallelmpp.dev` | 2 | **2** |
| `api.messari.io` | 2 | **2** |
| `api.nansen.ai` | 2 | **2** |
| `*.x402.paysponge.com`, `x402.tavily.com`, `stablesocial.dev` | 4 | **4** |
| `*.mpp.paywithlocus.com` | 13 | 0 |
| `mpp.orthogonal.com` | 11 | 1 |

`*.mpp.tempo.xyz` and the direct x402 merchants are in good shape. **`*.mpp.paywithlocus.com` went
0 for 13** — CoinGecko, Brave Search, Wolfram|Alpha, Stability AI, ScreenshotOne, Deepgram, Hunter,
RentCast, Billboard all returned `no x402 or MPP challenge detected` on a reachable host that
answers HTTP. That is a whole rail's worth of catalog listings that a buyer cannot transact
against, and nothing in `selat search` output distinguishes them from the ones that work — which
is Finding 14's "ranking is payability-blind" restated with numbers.

The 10 endpoints I could verify as live and priced, totalling **$0.5475**, are the paid-call plan:

```
$0.3000  POST  parallelmpp.dev/api/task                      Parallel — research task
$0.0788  GET   googlemaps.mpp.tempo.xyz/solar/v1/dataLayers  Google Maps Solar
$0.0600  POST  stablesocial.dev/api/reddit/search            StableSocial — Reddit
$0.0420  POST  fal.mpp.tempo.xyz/xai/grok-imagine-image      fal.ai — image gen
$0.0158  GET   serpapi.mpp.tempo.xyz/search                  SerpApi
$0.0105  GET   spyfu.mpp.tempo.xyz/apis/serp_api/v2/seo/*    SpyFu
$0.0105  GET   goflightlabs.mpp.tempo.xyz/flight-prices      GoFlightLabs
$0.0100  POST  x402.tavily.com/search                        Tavily
$0.0100  POST  api.nansen.ai/api/v1/tgm/flows                Nansen
$0.0100  GET   tripadvisor.x402.paysponge.com/…/location/search  Tripadvisor
```

Note the live prices run ~5% above the catalog's listed figures on the Tempo rail
(SerpApi listed $0.0150 → live $0.015750; Google Maps $0.0750 → $0.078750; fal.ai $0.0400 →
$0.042000). Consistent 1.05×, so the catalog appears to list pre-markup prices. Small, but a
budgeting agent that trusts `minAmountUsd` will under-estimate every Tempo call.
