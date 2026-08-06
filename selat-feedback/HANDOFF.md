# Handoff — resume the SELAT bounty run here

Previous session hit a wall: the cloud environment's egress policy blocked every SELAT host.
The user has since set **Network access = Custom** with the SELAT domains allowed. A new session
was required because environment changes don't reach an already-running container.

## First thing to do: confirm the policy actually applies

```bash
for h in api.circle.com catalog.selat.ai selat.ai api.apify.com; do
  echo "$h -> $(curl -sS --noproxy '*' -o /dev/null -w '%{http_code}' --max-time 10 https://$h/)"
done
```

`403` with body `Host not in allowlist: <host>` means the policy did **not** take effect — do not
proceed, tell the user. Anything else (200/301/404/405) means the host is reachable.

## State as of the handoff

| Item | Status |
|---|---|
| `@selat-ai/selat-cli` | installed globally, v0.15.7 — **reinstall, this is a fresh container**: `npm i -g @selat-ai/selat-cli` |
| plugin `selat@selat-plugins` | installed v0.1.8 via `claude plugin marketplace add SELAT-AI/selat-plugins` — **reinstall** |
| `selat skill list --available` | worked (19 skills; reads `raw.githubusercontent.com`) |
| `selat search` | failed on `api.apify.com` — retry now |
| wallet / funding / paid calls | **not started** |
| `FINDINGS.md` | 6 findings written; needs discovery + payment sections added |
| `EVIDENCE.md` | raw terminal output from the blocked run — keep, it's the bug repro |

Reference clones (re-clone if missing):
`git clone --depth 1 https://github.com/selat-ai/selat-plugins /workspace/selat-ai/selat-plugins`
`git clone --depth 1 https://github.com/selat-ai/selat-skills  /workspace/selat-ai/selat-skills`

## Remaining bounty steps

**2. Discovery** — `selat search "<capability>"`, `selat skill compare "<intent>" --limit 3`,
`selat skill list --available`. Free. Record whether ranking surfaced sensible endpoints and what
was missing; that's a graded part of the submission.

**3. Wallet** — user supplies their email, then `selat init` runs and prompts for a 6-digit OTP.
Relay the prompt; the user enters the code. Never run `circle` directly; never ask for a private key.

**4. Funding** — read the address back to the user, they send 1 USDC (Base recommended), then
`selat fund --chain base --amount 1 --method eco`. **Confirm before depositing.** Credits take
5–10 min; `--wait` blocks until spendable.

**5. Paid calls** — bar is **≥3 distinct endpoints and ≥0.5 USDC total**. Use `--dry-run` first,
surface the price, get an explicit OK per call. `selat freeze` is the kill switch.

**6. Verify** — `selat history` / `selat spend`, screenshot for the submission.

**8. (optional, +5 USDC)** — user chose **scaffold only, do NOT open a PR**. Author a skill per
`/workspace/selat-ai/selat-skills/meta/skill-creator/SKILL.md`, run `selat skill validate` and the
free `selat skill verify`, then stop and hand it to them.

## Standing constraints from the user

- Confirm before anything that spends or moves funds. Every time, not once.
- This container is ephemeral — the wallet and local `selat history` die with it. The user was told
  and chose to proceed here anyway. Get the screenshots before the session goes idle.
- Don't route around the egress proxy. If a host is blocked, report it.
