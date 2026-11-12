# Polymarket: "not enough balance / allowance ... spender: 0xd91E80…, allowance: 0" on a neg-risk order

If your bot has every trading approval its SDK reports as done and a neg-risk BUY still comes back with

```
400: not enough balance / allowance: the allowance is not enough -> spender:
0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296, allowance: 0, sum of matched orders: 0,
order amount (inc. fees): 5024990
```

then the order is fine and the approval set is wrong: the CLOB wants two approvals to the **Neg Risk Adapter**
(`0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296`) on neg-risk markets, and a fresh wallet set up the documented way
does not have them.

The trap is the name. `NegRiskCtfCollateralAdapter` (`0xadA2005600Dec949baf300f4C6120000bDB6eAab`) is a *different*
contract, and approving it does not grant the Neg Risk Adapter approvals. Polymarket's own contract page says both
sentences, checked 2026-11-12:

> The CLOB still checks pUSD and Conditional Tokens approvals for the Neg Risk Adapter on orders in CTF
> negative-risk markets. … This Neg Risk Adapter (0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296) is a different
> contract from NegRiskCtfCollateralAdapter (0xadA2005600Dec949baf300f4C6120000bDB6eAab); approving
> NegRiskCtfCollateralAdapter does not grant the Neg Risk Adapter approvals.

<https://docs.polymarket.com/resources/contracts>

## Why the SDK says you are fully approved

Both SDKs stopped adding this spender on purpose after the v1 adapter was retired (ts-sdk PR #217; py-sdk issue #178, fixed by PR #179),
so `setupTradingApprovals()` / `setup_trading_approvals()` leaves it out and the SDK's leftover-allowance recovery
cannot fix it: it resolves the spender to the neg-risk exchange, finds that one already at MAX, and re-raises the
original rejection. Upstream reports, both open when this page was written:

- ts-sdk: <https://github.com/Polymarket/ts-sdk/issues/480>
- py-sdk: <https://github.com/Polymarket/py-sdk/issues/305>

## Check it before you spend anything

Three checks on Polygon (four `eth_call`s), no signature and no gas:

1. `pUSD.allowance(yourWallet, 0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296)` on the pUSD collateral token
   `0xC011a7E12a19f7B1f670d46F03B03f3342E82DFB`.
2. `isApprovedForAll(yourWallet, 0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296)` on Conditional Tokens
   `0x4D97DCd97eC945f40cF65F87097ACe5EA0476045`.
3. The same two reads with the neg-risk exchange as the spender — `negRiskExchange` in both SDKs, the Neg Risk CTF
   Exchange `0xe2222d279d744050d28e00520010520000310F59` — to see which half is missing rather than both.

If one of them is zero while `get_balance_allowance(asset_type='COLLATERAL')` shows all your other spenders at MAX,
that is this bug and not a balance problem.

## What we did not test

We read the SDK sources, both upstream issues and the contract page above, and we watch the error on someone else's
reports. We have not sent a neg-risk order ourselves after granting these two approvals to `0xd91E80…`, so this page
says what is missing, not that granting it clears the 400 on the current CLOB build. If you have run it end to end,
an issue on this repo with the outcome would let us take that sentence out.

We also keep a read-only approvals check as a paid endpoint, `GET /v1/pm/approvals?wallet=0x…` on
<https://api.curade.xyz>: it reads both allowances for every trading spender of one wallet and names the missing one,
$0.02 a call over x402 on Base. It answers the question above; it is not needed to fix the order. We have not seen a completed paid call through it, so the JSON it returns is what its code does, not a response we have read; what we checked live is the 402 challenge: 20000 atomic USDC on Base, $0.02.
