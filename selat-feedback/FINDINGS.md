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
