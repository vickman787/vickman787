# Handoff — resume the SELAT bounty run here

**Session 2 ended blocked again**, on two hosts nobody knew about until this run:
`api.cdp.coinbase.com` and `mpp.dev`. The user's Custom network policy *did* apply — five of the
seven SELAT hosts are reachable now. It wasn't enough, because `selat search` fans out to five
registries in a `Promise.all` and any one of them failing zeroes the whole catalog (Finding 7).

Session 2 produced Findings 7–10 and a large evidence appendix. That work is committed. If the
run stops here permanently, the write-up is still a complete submission for the feedback half of
the bounty.

## First thing to do: confirm the policy

```bash
for h in api.circle.com catalog.selat.ai api.apify.com agi.apify.com api.cdp.coinbase.com mpp.dev; do
  printf '%-24s %s\n' "$h" "$(curl -sS --noproxy '*' -o /dev/null -w '%{http_code}' --max-time 10 https://$h/)"
done
```

`403` means blocked — the body reads `Host not in allowlist: <host>`. Anything else (200/307/404)
is reachable. **All six must be reachable** or `selat search` will fail again. Don't trust
`selat doctor` or the CLI's own sandbox hint for this; per Findings 9 and 10 neither one probes
these hosts.

The full allowlist to request:

```
api.circle.com  *.selat.ai  api.cdp.coinbase.com  mpp.dev  *.apify.com
registry.npmjs.org  *.npmjs.org
```

Paid calls will additionally need merchant hosts, which are per-endpoint and not knowable until
discovery works. Ones seen in the package: `gateway-api.circle.com` (reachable), `api.exa.ai`
(reachable), `x402.alchemy.com` (reachable), `agentskills.io` (blocked), `api.coingecko.com`
(blocked). Expect one more allowlist round after picking endpoints, or ask for a broader policy.

## State as of this handoff

| Item | Status |
|---|---|
| `@selat-ai/selat-cli` | **reinstall** — `npm i -g @selat-ai/selat-cli`. Was 0.15.7 both sessions; no new release. |
| plugin `selat@selat-plugins` | **reinstall** — `claude plugin marketplace add SELAT-AI/selat-plugins` then `claude plugin install selat@selat-plugins` |
| `selat doctor` | runs; 3 wallet failures; no network section (Finding 10) |
| `selat skill list --available` | ✅ works — 19 skills. Only needs `raw.githubusercontent.com`. |
| `selat search` | ❌ `Fatal: 403 api.cdp.coinbase.com` |
| `selat skill compare` | ❌ same failure via `rank.mjs exited 1` |
| Circle CLI | not installed; `selat init` installs it |
| wallet / funding / paid calls | **not started** |
| `FINDINGS.md` | Findings 1–10. Sessions 1 and 2 complete. Needs discovery + payment sections if those ever run. |
| `EVIDENCE.md` | raw output from both sessions, with source line numbers |

Reference clones (re-clone if missing):
`git clone --depth 1 https://github.com/selat-ai/selat-plugins /workspace/selat-ai/selat-plugins`
`git clone --depth 1 https://github.com/selat-ai/selat-skills  /workspace/selat-ai/selat-skills`

## Remaining bounty steps

**2. Discovery** — `selat search "<capability>"`, `selat skill compare "<intent>" --limit 3`,
`selat skill list --available`. Free. Record whether ranking surfaced sensible endpoints and what
was missing; that's a graded part of the submission. Blocked on the two hosts above.

**3. Wallet** — user supplies their email, then `selat init` runs and prompts for a 6-digit OTP.
Relay the prompt; the user enters the code. Never run `circle` directly; never ask for a private
key. `api.circle.com` is reachable, so this is *not* network-blocked — it's waiting on the user.

**4. Funding** — read the address back to the user, they send 1 USDC (Base recommended), then
`selat fund --chain base --amount 1 --method eco`. **Confirm before depositing.** Credits take
5–10 min; `--wait` blocks until spendable.

**5. Paid calls** — bar is **≥3 distinct endpoints and ≥0.5 USDC total**. Use `--dry-run` first,
surface the price, get an explicit OK per call. `selat freeze` is the kill switch.

**6. Verify** — `selat history` / `selat spend`, screenshot for the submission.

**8. (optional, +5 USDC)** — user chose **scaffold only, do NOT open a PR**. Author a skill per
`/workspace/selat-ai/selat-skills/meta/skill-creator/SKILL.md`, run `selat skill validate` and the
free `selat skill verify`, then stop and hand it to them.

## Decisions the user made at the end of session 2

- **Widen the allowlist and restart.** They are adding the seven hosts above and starting a fresh
  session. Confirm reachability first thing — if `api.cdp.coinbase.com` or `mpp.dev` still 403,
  the policy didn't apply; say so and stop rather than burning a third session on it.
- **Proceed with the wallet and paid calls.** Steps 3–5 are approved to run, target the
  ≥3 endpoints / ≥0.5 USDC bar. Ask for their email at the start of the session so `selat init`
  isn't waiting on it later, and relay the OTP prompt verbatim — they type the code.
- They accept that the wallet and local `selat history` die with the container. Capture the
  `selat history` / `selat spend` output for the submission **before** the session goes idle,
  not at the end.

## Standing constraints from the user

- Confirm before anything that spends or moves funds. Every time, not once.
- This container is ephemeral — the wallet and local `selat history` die with it. The user was told
  and chose to proceed here anyway. Get the screenshots before the session goes idle.
- Don't route around the egress proxy. If a host is blocked, report it.

## Note for whoever picks this up

Two sessions have now been spent discovering hosts by grepping `node_modules`. Before asking the
user for a third policy change, get the complete list in one pass:

```bash
grep -rhoE 'https://[a-zA-Z0-9._-]+' \
  "$(npm root -g)/@selat-ai/selat-cli/node_modules/@selat-ai/selat-discovery" | sort -u
```

and check each one, rather than trusting the CLI's `SANDBOX_ALLOW` constant — it lists two of the
five registries (Finding 8).
