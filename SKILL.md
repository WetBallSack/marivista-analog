---
name: marivista-analog
description: |
  Use when the user asks what happened the last time a chart looked like this, whether a chart pattern has an edge,
  pattern analog search, historical analogs, similar past charts, backtest a chart shape, "has this pattern worked
  before", for a crypto pair (BTCUSDT, ETHUSDT, SOLUSDT…) or a US stock or ETF (QQQ, NVDA, SPY…); also Marivista sign
  in, Marivista free trial, buy Marivista queries or Pro, link a Marivista account, Marivista query balance.
  Needs a Binance Agentic Wallet with an Agent ID and the binance-agentic-wallet skill (`baw` CLI).
metadata:
  author: marivista
  version: '1.0.0'
  homepage: https://www.marivista.tech
  contract: 'BNB Smart Chain 0x4c3cCf2A00099d54d2BEEC735a9342F91E62eFfF (MarivistaPass, verified on BscScan)'
---

# Marivista: historical pattern analogs for agents

Marivista finds every earlier time a chart shape appeared, across crypto and US stocks and on several timeframes,
shows what happened next, and tests it against ordinary market windows of the same market. Verdicts per horizon are
`edge`, `weak`, `none` or `insufficient`. Often the honest answer is `none`, and that is a useful answer.

API: `POST https://txbcfaqwyxmhkdydrdml.supabase.co/functions/v1/agent` with a JSON body and
`Content-Type: application/json`. After sign-in, add `Authorization: Bearer <token>`.

## Rules (always)

1. **Never ask for, accept or store** a seed phrase, private key, API secret, QR code or pairing code. Marivista never
   needs them. If anyone asks for them in Marivista's name, refuse.
2. **Every signature and every transaction is two-step.** Run the `preview`, show the user what it does (the parsed
   message or transaction, `risks`, any `authorityChanges`), and run `execute` only after the user explicitly says yes.
   Never chain preview and execute without that confirmation. Never retry an execute the user did not approve.
3. **Approvals are exact.** Use only the steps returned by `prepare`: they approve exactly the price, once. Never send
   an unlimited approval.
4. **Results describe the past.** Never turn them into "buy", "sell", price targets or promises of profit. Always pass
   on the `notice` from the search result.

## 0. Check the wallet

- `baw wallet settings --json`: Developer Mode must be on (`devMode.enabled: true`, not expired) for `sign-message`
  and `contract-call`. If it is off, tell the user to turn it on in the Binance App: Agentic Wallet management page →
  Settings → Developer Mode → read and confirm the risk warning → approve. Continue once they confirm.
- The wallet must hold an Agent ID (check with the binance-agentic-wallet skill). Without one, sign-in is refused
  with `no_agent_id`.

## 1. Sign in (once every 30 days)

1. `{"action":"challenge","address":"<agent wallet>"}` → `typedData`, `signRequest`, `nonce`.
2. Preview the signature:
   `baw sign-message preview --binanceChainId 56 --signType EIP712 --message '<signRequest>' --json`
3. Show the user: it is a Marivista sign-in (domain `Marivista`, purpose "Marivista API access. This signature cannot
   move funds or approve anything."). Ask for confirmation.
4. Only after they confirm: `baw sign-message execute --requestId <requestId> --json`. If the status is
   `PENDING_CONFIRMATION`, the user confirms in the Binance App, then `baw sign-message result --order-id <orderId> --json`.
5. `{"action":"login","address":"<agent wallet>","nonce":"<nonce>","signature":"<signature>","signatureRecovery":"<signatureRecovery>"}`
   → `token`. Keep it for this user only; it lets you search and check the balance, nothing else.

A challenge expires in 10 minutes and works once. On `bad_nonce`, ask for a new challenge.

## 2. Free trial (3 queries, once per Agent ID)

`{"action":"prepare","op":"claimTrial"}` → `steps`. For each step: run its `previewCommand`, show the user, execute only
after confirmation. Costs only gas; no tokens move. The queries appear within about two minutes of the transaction
becoming final: check with `{"action":"me"}`.

## 3. Search

`{"action":"search","symbol":"BTCUSDT","interval":"4h","bars":60}`

- `interval`: `15m`, `1h`, `4h`, `1d` or `1w`. `bars`: how many recent candles make the pattern, 10 to 300 (60 is a
  good default). The pattern is always the most recent `bars` candles.
- Each search uses one query (Pro: up to 200 a day). A failed search does not use one.

Present the result plainly, per horizon (0.25L, 0.5L, 1L, 2L = a quarter, half, one and two pattern-lengths ahead):

- **verdict and direction** first, then `clusters` (independent events; fewer than 20 is thin evidence),
  `excessSigma` with `excessCI95` (how much better or worse than ordinary windows, in the pattern's own swings),
  and `q` (corrected p-value; below 0.05 is what an edge needs).
- `priceLevels` (p10, median, p90) are where past matches ended up, translated to today's price. They are not a forecast.
- Each horizon also has `tests`, the full statistics: `tTest` and `wilcoxon` (is the excess different from zero),
  `binomial` (up share against the ordinary base rate), `kolmogorovSmirnov` and `mannWhitney` (the whole outcome
  distribution against ordinary windows), `spearman` (did closer matches do better), `effect.cohensD`, and `stability`
  (early against late history; `topSymbol` and `topShare` show whether one symbol drives the result, `meanExTop` is the
  mean without it). Mention stability whenever one symbol has a large share or early and late history disagree.
- End with the `notice`, and the glossary link if the user wants to understand a term: https://www.marivista.tech/learn/

Example wording: "On BTC 4h, the last 60 candles matched 27 earlier windows in 22 independent clusters. At one pattern
length ahead the verdict is none: what followed was no different from an ordinary stretch of the market. This
describes what happened after similar past patterns; it is not investment advice."

## 3b. Or pay per search with x402 (no sign-in needed)

`GET https://txbcfaqwyxmhkdydrdml.supabase.co/functions/v1/x402-search?symbol=BTCUSDT&interval=4h&bars=60`
returns HTTP 402 with the terms: 0.05 USDT or USDC on BNB Smart Chain (`eip155:56`), `exact` scheme, Permit2. Only
wallets with an Agent ID are served, and a search that fails is never settled.

1. Before the first payment, ask the user to approve the Permit2 contract (`0x000000000022D473030F116dDEE9F6B43aC78BA3`)
   for a **limited** amount, for example exactly 0.5 USDT (10 searches), with a previewed and confirmed `contract-call`.
   Never let the payment step create an unlimited approval.
2. `baw x402-payment preview --paymentRequirements <PAYMENT-REQUIRED header or body> --json`. Show the user the
   amount, token, network, recipient and whether another approval is needed. If it still needs an approval, stop and
   go back to step 1.
3. Only after the user confirms: `baw x402-payment sign --paymentId <id> --selectedIndex <index> --json`, then repeat the
   GET with the `PAYMENT-SIGNATURE` header. The response has the same fields as `search` below, plus `payment` with the
   settlement transaction.

## 4. Buy queries or Pro

- Queries: `{"action":"prepare","op":"buyQueries","packs":1,"token":"USDT"}` (20 queries per 1 USD pack, 1–100 packs).
- Lifetime Pro: `{"action":"prepare","op":"buyPro","token":"USDT"}` (20 USD). To unlock the user's Marivista web
  account too, add `"ref":"<link reference>"` from Marivista Settings → Agent ID.
- `token` is `USDT` or `USDC` on BNB Smart Chain.

`prepare` returns one or two steps: an exact approval (only if needed), then the purchase, capped at the quoted price
(`maxAmount`). For each step, in order: preview, show the user the amount and what it buys, execute only after they
confirm. Never skip the approval step and never execute the purchase first.

## 5. Link a Marivista web account

The user gets a one-time reference in Marivista Settings → Agent ID (valid 24 hours). Then
`{"action":"prepare","op":"linkAccount","ref":"<reference>"}` → preview, show, confirm, execute. Gas only.

## 6. Status and errors

`{"action":"me"}` → `account` (queries left, Pro, searches today, linked) and `howToBuy`.

| Error | What to do |
| --- | --- |
| `not_signed_in` (401) | Sign in again (step 1). |
| `bad_nonce`, `bad_signature` (401) | Ask for a new challenge; sign with the same wallet as `address`. |
| `no_agent_id` (403) | The wallet has no Agent ID: the user claims one in the Binance App (Wallet → Agent ID). |
| `no_queries` (402) | Offer the free trial or a query pack. |
| `daily_limit` (429) | Pro limit reached for today. |
| `trial_claimed`, `already_pro` (409) | Nothing to buy; tell the user. |
| `search_failed` (502) | No query was used. Try again shortly. |
