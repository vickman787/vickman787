# Screen recording script (Pond AI form, question 3)

Target: "discovery through paid calls, under normal usage conditions." About 4 to 6 minutes.
Everything below is real, no simulation. The wallet is on the Circle account, so the paid call
at step 6 genuinely settles.

Balance available: **0.417250 USDC**. The whole script spends about **$0.026**.

## Before you hit record

```bash
npm i -g @selat-ai/selat-cli @selat-ai/selat-pay
export SELAT_ROUTER_URL=https://router.selat.ai
```

Do `selat init` off camera if you would rather not film the OTP, it needs a 6 digit code emailed
to Vickmancrypto@gmail.com. Filming it is fine too, just blur or skip the code. On a new machine
the local `selat history` starts empty, which is expected, the ledger is per machine while the
balance is not.

## On camera

**1. Environment, 10 seconds**

```bash
selat --version && node -v
```

**2. Health check**

```bash
selat doctor
```

Shows the wallet, the Gateway balance, the spending policy, and the router reachability line.

**3. Discovery, free**

```bash
selat search "web search"
```

The line to pause on is `Catalog: 2596/2596 merged services`, that is all five registries merging.

**4. Available skills**

```bash
selat skill list --available
```

Reliability dots and the "checked Nh ago" timestamp.

**5. Free probe before spending**

```bash
selat-pay POST https://x402.tavily.com/search --chain base --max-amount 0.02 --probe-only
```

Prints the live 402 quote without settling. Worth showing, it is the responsible step before a
paid call.

**6. Paid call, this one settles**

```bash
selat-pay POST https://x402.tavily.com/search --chain base --max-amount 0.02 \
  --body '{"query":"x402 agent payments protocol","max_results":3}'
```

Costs $0.0105. Ends `status=200` with real search results.

**Do not omit `--body`.** Without it the merchant returns `400 Validation failed` and you are
still charged, which is Finding 21. If you want to demonstrate that bug on camera it is a
deliberate $0.0105, otherwise keep the body.

**7. A second endpoint, different rail**

```bash
selat-pay GET "https://serpapi.mpp.tempo.xyz/search?q=x402+protocol&engine=google" \
  --chain base --max-amount 0.03
```

Costs $0.01575, settles `routed-mpp` rather than `routed-x402`, so the recording covers both rails.

**8. Verify the spend**

```bash
selat history
selat spend
```

This is also the artifact for the "Gateway transactions screenshot" question. Screenshot this
frame, or export it separately.

## If you want the run to also show the headline finding

Optional, free, about 20 seconds. Shows that the router reaches a host direct egress cannot,
which is Finding 12:

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://api.agentstools.dev/search
curl -sD - "https://router.selat.ai/proxy?target=https%3A%2F%2Fapi.agentstools.dev%2Fsearch" \
  | head -3
```

Only meaningful from a network that blocks the first host. On an unrestricted home connection
both succeed and the point does not land, so skip it unless you are recording from a restricted
environment.

## Things worth narrating

- No API keys anywhere, no accounts with any of the merchants.
- Each call settles in seconds against one USDC balance.
- The spending policy is a Circle side ceiling the agent cannot exceed.
