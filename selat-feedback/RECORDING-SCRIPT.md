# Screen recording script (Pond AI form, question 3)

Target: "using the SELAT plugin under normal usage conditions, ideally covering discovery through
paid calls." About 5 to 8 minutes.

The question says **plugin**, and the bounty's own task prompts are natural language, so record the
agent driving SELAT rather than hand typed CLI calls. Q2 was answered "Claude Code", so the
recording should match that.

Everything below is a real run. The wallet lives on the Circle account, not in any container, so
the paid calls genuinely settle. Balance available: **0.417250 USDC**. The script spends about
**$0.03**.

## Setup, off camera

VS Code with the Claude Code extension, or Claude Code in the VS Code integrated terminal. Either
reads as normal usage.

```bash
npm i -g @selat-ai/selat-cli
claude plugin marketplace add SELAT-AI/selat-plugins
claude plugin install selat@selat-plugins
export SELAT_ROUTER_URL=https://router.selat.ai
```

Then `selat init` if this machine has never run it. It emails a 6 digit code to
Vickmancrypto@gmail.com. Do this off camera, or film it and skip past the code.

The local `selat history` starts empty on a new machine. That is expected, the ledger is per
machine while the balance is not.

## On camera, prompts not commands

Type these to the agent and let it work. Pause after each so the output is readable.

**1. Confirm the setup**

> Run selat doctor and tell me if my SELAT setup is healthy.

Shows wallet, Gateway balance, spending policy, router reachability.

**2. Discovery, free**

> Find me a web search endpoint I can pay for through SELAT. Don't spend anything yet.

The agent should run `selat search`. The line worth pausing on is
`Catalog: 2596/2596 merged services`, that is all five registries merging.

**3. What skills exist**

> What SELAT skills are available to install?

**4. Price it before spending**

> Probe the Tavily x402 search endpoint so I can see the live price without paying.

Expect `--probe-only` and a quoted price around $0.0105.

**5. The paid call**

> Make one paid call to https://x402.tavily.com/search searching for "x402 agent payments
> protocol". Send a proper JSON body with a query field, cap it at $0.02, and tell me the price
> before you settle.

Asking for the body explicitly matters. Without it the merchant returns `400 Validation failed`
and you are charged anyway, which is Finding 21. Costs about $0.0105 and ends `status=200` with
real results.

**6. A second endpoint on the other rail**

> Now make a paid call to the SerpApi endpoint at serpapi.mpp.tempo.xyz searching for
> "x402 protocol" on Google, capped at $0.03.

About $0.01575 and settles `routed-mpp` rather than `routed-x402`, so the recording covers both
rails.

**7. Verify**

> Show me my SELAT payment history and total spend.

`selat history` and `selat spend`. Screenshot this frame, it doubles as the Gateway transactions
artifact the later question asks for.

## If the agent goes off script

Two failure modes are worth knowing rather than being surprised by on camera.

- **It picks a different endpoint than you asked for.** Most catalogue merchants are on hosts that
  may not resolve or may not serve a challenge. Steer it back to the two named above, both are
  confirmed working.
- **It omits the request body.** That is a charged 400. If it happens, leave it in the recording
  and say so, it is a genuine demonstration of the most serious finding in the submission.

The Circle policy caps every transaction at $0.50 and the day at $2, so the agent cannot run away
with the balance regardless.

## Fallback, if the plugin misbehaves

Direct CLI, same coverage, less representative of "plugin usage":

```bash
selat doctor
selat search "web search"
selat skill list --available
selat-pay POST https://x402.tavily.com/search --chain base --max-amount 0.02 --probe-only
selat-pay POST https://x402.tavily.com/search --chain base --max-amount 0.02 \
  --body '{"query":"x402 agent payments protocol","max_results":3}'
selat-pay GET "https://serpapi.mpp.tempo.xyz/search?q=x402+protocol&engine=google" \
  --chain base --max-amount 0.03
selat history
selat spend
```

## Worth narrating

- No API keys anywhere, no accounts with any of the merchants.
- Each call settles in seconds against one USDC balance.
- The spending policy is a Circle side ceiling the agent cannot exceed.
