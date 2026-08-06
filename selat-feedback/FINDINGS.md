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
twice") is clearly hard-won. But the merge gate is `selat skill verify`, which probes each step's
live 402 challenge, and `--pay` needs a funded wallet. Both need the catalog and merchant hosts. So a
contributor behind an egress policy can scaffold and statically validate a skill but **cannot produce
a mergeable PR at all** — the verify receipt that gates merge is unobtainable. Worth calling out in
`CONTRIBUTING.md`, and worth considering whether static validation + a maintainer-run paid verify is
enough for a first-time contributor.

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
