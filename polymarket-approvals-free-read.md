# Polymarket neg-risk approvals: read the flags, and what `is_fully_approved` checks

This is a companion to [polymarket-neg-risk-adapter-allowance.md](polymarket-neg-risk-adapter-allowance.md), which
explains the `400 not enough balance / allowance … spender: 0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296` error and
the two approvals it is missing. This page covers the two things a builder hits next: where to read those flags
without paying anyone, and why the SDK's own "fully approved" is not the same question.

## Reading the approvals from the data API (added 2026-11-14)

Polymarket's own data API answers this for any wallet, no key:

    GET https://data-api.polymarket.com/v2/approvals?user=0xYourWallet

It returns `data.checked_at`, `data.chain_id` and a `data.contracts[]` row per token and spender, each with
`approved` and `amount`. `amount` reads `max` even on rows where `approved` is `false`, so read `approved`. The
rows that matter here are `UsdcNegRiskAdapter` and `CtfNegRiskAdapterIsApprovedForAll`, both listing spender
`0xd91e80cf2e7be2e162c6513ced06f1dd0da35296`. Checked 2026-11-14: a wallet with every trading row already approved
returns `approved: true` on both (0xe4a57385a34b0b32fd86be72efd25cda41003cad, top of the public leaderboard, so
you can rerun it), and a wallet that has never approved anything returns `approved: false` on all of them
(0x56641a5a6c20e0673f1f58844a19136cfa24af8f). The rows cover the same allowances the four `eth_call`s above read,
though we compared the JSON on these two wallets, not row by row.

## What `is_fully_approved` actually checks (source read 2026-11-14)

`polymarket/_internal/actions/relayer/approvals.py` on `main` builds its required list from
`_required_trading_approvals()`: the collateral token approved to `standard_exchange`, `neg_risk_exchange`,
`collateral_adapter`, `neg_risk_collateral_adapter`, `protocol_v2_router`, `exchange_v3` and
`perps_deposit_contract`, plus CTF operator approvals for most of the same spenders, the auto-redeem operator and
the modules. `neg_risk_adapter` is not in that list, so `is_fully_approved` can be true while a neg-risk order is
still rejected for zero allowance to the Neg Risk Adapter. That gap is enough to produce the 400 above without your
balance being involved, though we still have not seen an order pass after granting the two approvals.

## The fix, unchanged

Signed from the trading wallet, both parts:

1. `pUSD.approve(0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296, max)` on
   `0xC011a7E12a19f7B1f670d46F03B03f3342E82DFB`
2. `ConditionalTokens.setApprovalForAll(0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296, true)` on
   `0x4D97DCd97eC945f40cF65F87097ACe5EA0476045`

A bounded allowance has been reported rejected on the relayer path, which is why the page above says max. Read both
flags before and after: the data-API URL shows the result immediately.

## What we did not test

We have not sent a neg-risk order ourselves after granting these two approvals, so this page says what is missing,
not that granting it clears the 400 on the current CLOB build. If you run it end to end, an issue on this repo with
the outcome would let us take that sentence out.

We also run a read-only paid check at `GET https://api.curade.xyz/v1/pm/approvals?wallet=0x…` ($0.02 over x402 on
Base) that returns a verdict, the blocking spender and the exact calls to send. Nothing on this page needs it: the
data-API URL plus the two calls above is the whole fix, and we would rather you spend nothing when the read answers
your case.
