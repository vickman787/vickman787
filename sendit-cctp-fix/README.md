# SendIt — CCTP bridge stuck on `Challenges not complete: …=PENDING`

Diagnosis and fix for [0xvictoren/SendIt](https://github.com/0xvictoren/SendIt):
Arc Testnet → Base Sepolia bridge burns USDC, then stalls before attestation
because at least one UCW challenge never leaves `PENDING`.

`0001-fix-ucw-cctp-pending-challenges.patch` applies to the SendIt repo:

```bash
git clone https://github.com/0xvictoren/SendIt && cd SendIt
git checkout -b fix/ucw-cctp-pending-challenges
git am < /path/to/0001-fix-ucw-cctp-pending-challenges.patch
cd server && npm install && npx tsc --noEmit && npx tsx --test src/services/circle-ucw.test.ts
```

---

## 1. Why a challenge stays PENDING after a successful PIN and a confirmed burn

This is expected Circle behaviour, not a bug in your PIN flow. A UCW challenge
is only flipped to `COMPLETE` once Circle has finished processing it end to
end. For a `CONTRACT_EXECUTION` challenge (`approve`, `depositForBurn`) that
happens **after** the user's PIN, and routinely **after** the transaction is
already mined on Arc.

So this state is entirely normal for a few minutes:

| | value |
|---|---|
| `GET /user/challenges/{id}` → `status` | `PENDING` |
| correlated transaction `txHash` | `0x…` (mined on Arc) |

`ChallengeStatusEnum` is `PENDING | IN_PROGRESS | COMPLETE | FAILED | EXPIRED`
(`@circle-fin/user-controlled-wallets`). Only `FAILED` and `EXPIRED` mean the
step did not happen. `PENDING` means "not finished processing" — it does **not**
mean "the user did not sign".

Your code treated anything that was not `COMPLETE` as a failure:

```ts
// server/src/services/circle-ucw.ts — before
if (st !== "COMPLETE") allDone = false;
// …
error: `Challenges not complete: ${pending.map(p => `${p.challengeId.slice(0,8)}…=${p.status}`)}`
```

That string is exactly the error in your screenshot. It fires on a **healthy**
burn whose challenge record has not caught up yet.

Two things then made it terminal rather than transient:

**The verify window was far too short.** `executeChallengeAndVerify` defaulted
to `timeoutMs: 60000`, and `/verify-challenges` capped it at `120_000`. Circle's
own UCW signing strategy waits `600_000` ms (10 minutes) by default with a 2s
poll — see `CircleUserWalletSigningBaseOptions.timeoutMs` in
`@circle-fin/adapter-circle-wallets`. A 60s ceiling turns a slow-but-fine burn
into a reported failure.

**The WebView was destroyed mid-flush.** `challenge.html` resolved `execute()`
on *any* non-failed status, including `IN_PROGRESS` and `PENDING`:

```js
// mobile/assets/challenge.html — before
if (status === 'FAILED' || status === 'EXPIRED' || …) { reject(…); return; }
resolve(result || { status: status || 'COMPLETE' });   // resolves on IN_PROGRESS
```

There was also a dead branch above it — an `if` whose body was only comments —
which is a good sign the status handling was never finished. The Flutter screen
then popped the route immediately on `success`, which unmounts the WebView and
kills the Circle iframe while it is still posting the signed payload back. That
turns a recoverable "still settling" into a genuinely stuck `PENDING`.

## 2. Why the mint never happened even though the burn did

This is the part that actually cost you the transfer, and it is a plain control-flow
bug rather than anything to do with Circle:

```dart
// mobile/lib/core/wallet/circle_wallet_service.dart — before
final burnResult = await runStep({'step': 'depositForBurn', …}, resolveTx: true);
if (burnResult['ok'] != true) return burnResult;   // ← returns before /cctp/finish
```

`runStep` reports `ok: false` when verification times out — including when the
burn is already on-chain. The early `return` skips the `/v1/circle/cctp/finish`
call that resolves the burn hash, polls Iris, and mints on Base Sepolia. So
attestation never starts, and there is no retry path because the same `runStep`
also called `_confirmActivity(ok: false)`, discarding the receipt for USDC that
had genuinely been burnt.

`/cctp/finish` already accepts `burnChallengeId` **or** `burnTxHash` and does its
own longer hash resolution — the flow just never reached it.

## 3. The recommended way to run multiple UCW challenges

Three rules, all now enforced in the patch:

**Never judge a challenge by its status alone.** Read the challenge, take
`correlationIds[0]` as the transaction id, read that transaction, and treat the
step as settled if the challenge is `COMPLETE` **or** the transaction has a hash
or reached `SENT`/`CONFIRMED`/`COMPLETE`. Only `FAILED`/`EXPIRED` challenges and
`FAILED`/`DENIED`/`CANCELLED` transactions mean nothing moved.

**Create the next challenge only after the previous one has settled.** Your
sequential CCTP path (`/cctp/burn` → approve → `/cctp/burn/continue` → burn)
already does this correctly, and it is the right shape. The `step: "all"` mode
in `createCctpBurnChallenges` creates approve and burn up front; the burn will
revert if it is executed before the approve lands. Keep it on `sequential`.

**Never re-prompt a settled challenge.** The old retry loop in `_runChallenges`
re-opened the WebView for anything not `COMPLETE`, which could ask the user to
re-sign a burn that had already happened. It now skips `settled` ids.

Given rule 1, one `W3SSdk` instance per challenge in a single WebView session is
fine. What is not fine is resolving before the SDK reports a terminal status.

## 4. What to call when a challenge looks stuck

There is no API that "forces" a challenge to complete — the challenge is just a
record of an authorization, and the pipeline is driven by the transaction, not by
the challenge. The recovery path is to **stop waiting on the challenge and act on
the transaction**:

1. `GET /v1/w3s/user/challenges/{id}` → take `correlationIds[0]`.
2. `GET /v1/w3s/transactions/{id}` → if there is a `txHash`, the burn happened.
3. `POST /v1/circle/cctp/finish` with `burnTxHash` (or `burnChallengeId`) — this
   polls Iris for the attestation and mints on the destination.

Your app already exposes step 3 as `finishBridge()` and the **Retry mint** button
in `bridge_screen.dart`. Before this patch, it was unreachable for the failing
case, because the burn's `activityId` was discarded and `_burnTxHash` was never
set. It now works.

## 5. App Kit on Arc Testnet → Base Sepolia

I found no Arc-specific or WebView-specific defect. The App Kit job relay had one
real gap, which is worth fixing regardless:

`ucwAdapter` passed `onChallenge` but not `resolveTypedDataSignature`. Per
`@circle-fin/adapter-circle-wallets`:

> Required to sign typed data: without it the strategy does not declare the
> `evm-typed-data` payload family, so `supportsSignTypedData()` reports `false`
> […] a signature is **only** delivered to the client that executed the
> challenge, in the W3S browser SDK's challenge result. A completed challenge
> read from the server carries `id`, `status`, and `type` and nothing else.

So any step needing an EIP-712 signature either silently took a slower on-chain
route or stalled — and `challenge.html` was discarding `result.data.signature`
anyway. The patch adds the relay: the WebView captures the signature, the client
POSTs it to `/v1/app-kit/jobs/:id/signature`, and the adapter's
`resolveTypedDataSignature` resolves.

Note `onChallenge` itself is correctly fire-and-forget — the strategy resumes by
polling and does not wait for it. That part of your code was fine.

## What changed

**Server**

- `circle-ucw.ts` — `classifyChallenge()` (pure, unit-tested) and
  `readChallengeSettlement()` decide from challenge status *and* transaction
  state. `waitForChallengesComplete()` returns `settled`/`dead` per challenge
  plus the first observed `txHash`, and stops re-reading settled challenges.
- `circle-wallets.ts` — `/verify-challenges` cap `120s → 600s`, default `300s`;
  returns `anySettled`/`allSettled`/`dead`/`txHash` even when not all complete.
  `/cctp/finish` burn-hash resolve window `90s → 180s`.
- `appKitMoney.ts` — typed-data signature relay + `onProgress` logging so a job
  shows movement between PIN and the landed transaction.
- `app-kit.ts` — `POST /v1/app-kit/jobs/:id/signature`; jobs expose
  `awaitingSignature` and per-challenge `type`/`needsSignature`.

**Client**

- `circle_wallet_service.dart` — bridge falls through to `/cctp/finish` unless
  the user cancelled or the challenge is dead; activity is no longer discarded
  on an inconclusive burn; settled challenges are never re-prompted; signature
  relay; verify timeouts `60s → 180s` (burn `240s`).
- `challenge.html` — waits for a terminal SDK callback (6s grace on
  `IN_PROGRESS`, 180s hard timeout) instead of resolving immediately; captures
  `result.data.signature`; dead branch removed.
- `circle_challenge_screen.dart` — 1.2s settle delay before popping so the SDK
  can flush; collects and returns signatures.

## Verification

- `npx tsc --noEmit` — clean.
- `npx tsx --test src/services/circle-ucw.test.ts` — 10/10 pass, covering the
  PENDING-challenge-with-a-tx-hash case that caused this.
- `npx tsx --test src/utils/handles.test.ts` — 4/4, unchanged.

Not run: `flutter analyze` / `dart analyze` — no Dart or Flutter SDK in this
environment. The Dart edits were reviewed by hand and brace-balance checked, but
they have not been compiler-verified. Run `flutter analyze` before you rely on
them.
