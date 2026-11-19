# x402: "All payment requirements were rejected by spendControls.maxAmountPerPayment ($1, including USDC)"

Your buyer uses the official x402 SDK and the endpoint you are calling costs more than a dollar. The SDK never
signs: it throws before a payload exists, so no money moves and nothing reaches the facilitator. The cause is a
default spend cap of `$1` per payment in `@x402/core` (from 2.23.0, 2026-08-18) and in the Python `x402` client
(from 2.20.0). Raise the cap deliberately, or price the endpoint at a dollar or below.

## How to check

The message names the cap that fired, so read it before anything else:

```
All payment requirements were rejected by spendControls.maxAmountPerPayment ($1, including USDC). Raise
maxAmountPerPayment, set it to false to disable, set allowedAssets[].maxAmountPerPayment for a per-asset atomic cap,
or set spendControls: false to disable all spend controls.
```

That is the TypeScript text verbatim (`typescript/packages/core/src/client/x402Client.ts`, read 2026-11-19); the
Python client throws the same idea with snake_case names, `spend_controls.max_amount_per_payment`
(`python/x402/client_base.py`, read 2026-11-19). If you see it, the amount in the 402 is over your cap. Decode the
402 to see by how much: take the `payment-required` header, base64-decode it, and read `accepts[].amount`. That
number is atomic, so for USDC on Base divide by `10^6`. On a $2.00 endpoint we run, the decoded value was `2000000`,
and an out-of-the-box client refused it on 2026-11-09.

The comparison is atomic and inclusive, `BigInt(rawAmount) <= BigInt(cap)`, so exactly `$1.00` passes and
`$1.000001` does not: for USDC, `parseMoney("$1")` through `convertToTokenAmount(..., 6)` gives `1000000`, so
`1000000` passes and `1000001` is rejected. It runs over every entry in `accepts`, and the error only fires when
every entry is over the cap; a 402 that offers two prices passes if one of them fits.

## The fix, by client

TypeScript, on `@x402/core` 2.30.0 and 2.31.0, raise the cap on the client:

```js
client.setSpendControls({ maxAmountPerPayment: "$5" });
```

Pass the same object as `spendControls` to `x402Client.fromConfig({ schemes, spendControls })` if you build the
client that way. Do not pass it to the constructor: `new x402Client(...)` takes a requirement selector and nothing
else, so a `{ spendControls }` object is taken as that selector and the first payment attempt throws — the $1 error
if the amount is over it, `TypeError: this.paymentRequirementsSelector is not a function` otherwise. The key is never
read. `setSpendControls` replaces the whole control set, and an object with no `maxAmountPerPayment` still means
`$1`, so keep that key in it.

Python, the client is `x402Client` from `x402`. Either hand
`x402ClientConfig(spend_controls={"max_amount_per_payment": "$5"}, ...)` to `x402Client.from_config`, or call
`client.set_spend_controls({"max_amount_per_payment": "$5"})` afterwards. The default constant is
`DEFAULT_MAX_AMOUNT_PER_PAYMENT: Money = "$1"` in both languages.

If you want a cap per token instead of in dollars, `allowedAssets` entries take `maxAmountPerPayment` in atomic
units, and it must be an integer string: `"5000000"` is $5 of a six-decimal token. Pass a dollar string there and the
client throws `spendControls.allowedAssets[].maxAmountPerPayment must be an integer atomic amount, not a dollar
value`. The top-level cap is parsed as dollars, so a bare `"5000000"` there means five million dollars, not five;
atomic amounts belong in `allowedAssets` only.

`setSpendControls(false)` and `spendControls: false` turn every spend control off, including the asset allowlist.
That is the switch people reach for first and the one to leave alone in anything unattended: with it off, any
server that answers your GET with a large `amount` gets a signature for that amount.

## Why the cap is on by default

The cap came in with PR #3124, "feat(TS): spend controls", merged 2026-08-13 and published in `@x402/core` 2.23.0 on
2026-08-18 (npm registry, read 2026-11-19): "declarative client spend controls that enforce limits before payment
signing", a default $1 cap on recognized pegged assets. Issue #3866 (open, 30 comments, filed 2026-10-21 against
`@x402/fetch` 2.28.0) is what that leaves: the client signs whatever the 402 declares, up to the cap; the caller's own
spend limit was the only guard.

That framing matters for the fix. A cap you raise to `"$5"` is a per-payment ceiling you chose. Turning it off means
any 402 can ask for more than you meant to allow. Set the number, set it once, and keep the asset allowlist on.

## Two neighbours, briefly

The same filter rejects a token it does not consider a default asset first, with "only default assets or entries in
spendControls.allowedAssets are allowed"; a USDC payment is not affected, a stablecoin missing from the SDK's
default-asset table — or a fresh deployment — is. `allowedAssets: true` allows anything,
`allowedAssets: [{network, asset}]` allows named tokens.

Issue #3505 (open, read 2026-11-19) is a request to fix the cap's assumption that every default asset is pegged to
the dollar, which it is not for every chain the SDK supports.

## For the other side: you run the seller

An endpoint priced above a dollar gets no paid calls from an unconfigured current client (`@x402/core` 2.23.0+,
Python `x402` 2.20.0+). Either price at `$1.00` or less, or say in your docs which field to raise, with the client's
exact option name. Nothing else on your side changes: your 402 is not the problem, the facilitator is never reached,
and the request never becomes a settlement.

## What we did not test

We did not make a payment at any cap. Our evidence is the SDK refusing before signing on 2026-11-09, the source and
npm metadata above, and the two issue threads; we did not run the Python client end to end, and we did not test the
Go client at all. We did not test per-asset caps against a paid call, and we did not test any SDK release after
2.31.0 (published 2026-11-12). Names and line numbers move between releases, so read the file at your pinned version
before trusting either; the names above are what 2.30.0 and 2.31.0 contain.

If the 402 decoded fine and the amount is what you expected, the next page you want is
[decode-a-402-by-hand.md](decode-a-402-by-hand.md). If the error is about the token instead of the amount, see
[cdp-discriminator-property-type.md](cdp-discriminator-property-type.md) when the message mentions a `type`
discriminator.

## Sources

- TypeScript client, `typescript/packages/core/src/client/x402Client.ts` on `main` (`DEFAULT_MAX_AMOUNT_PER_PAYMENT`,
  the `BigInt(rawAmount) <= BigInt(cap)` comparison, and both `spendControls` error strings) —
  <https://github.com/x402-foundation/x402/blob/main/typescript/packages/core/src/client/x402Client.ts>,
  read 2026-11-19.
- Python client, `python/x402/client_base.py` (`DEFAULT_MAX_AMOUNT_PER_PAYMENT`, `set_spend_controls`, the
  snake_case error) — <https://github.com/x402-foundation/x402/blob/main/python/x402/client_base.py>, read 2026-11-19.
- Python release notes: "[2.20.0] - 2026-08-18 … add client `spend_controls`: by default only recognized pegged
  assets are allowed with a `$1` USD cap" —
  <https://github.com/x402-foundation/x402/blob/main/python/x402/CHANGELOG.md>, read 2026-11-19.
- npm `@x402/core` dist-tags and publish times (2.22.0 has no `spendControls`; 2.23.0 published 2026-08-18;
  2.31.0 published 2026-11-12), both the registry document and the 2.22.0/2.31.0 `index.mjs` bundles —
  <https://registry.npmjs.org/@x402/core>, <https://unpkg.com/@x402/core@2.22.0/dist/esm/client/index.mjs>,
  <https://unpkg.com/@x402/core@2.31.0/dist/esm/client/index.mjs>, read 2026-11-19.
- PR #3124 "feat(TS): spend controls", merged 2026-08-13 —
  <https://github.com/x402-foundation/x402/pull/3124>, read 2026-11-19.
- Issue #3866, open, 30 comments, filed 2026-10-21 — <https://github.com/x402-foundation/x402/issues/3866>,
  read 2026-11-19.
- Issue #3505, open — <https://github.com/x402-foundation/x402/issues/3505>, read 2026-11-19.
- SDK feature matrix listing spendControls for TypeScript, Go and Python, `docs/sdk-features.md` —
  <https://github.com/x402-foundation/x402/blob/main/docs/sdk-features.md>, read 2026-11-19.
