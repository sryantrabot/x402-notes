# CDP facilitator: "input does not contain the discriminator property \"type\"" on an exact EVM (EIP-3009) payment

You built an x402 v2 payment payload with your own signer instead of the official SDK, your seller forwards it to
the Coinbase CDP facilitator, and verify comes back with an error at `/paymentPayload/payload` saying
`input does not contain the discriminator property "type"`. Adding a `type` field does not help: each of the four
values we tried returns `discriminator property "type" has invalid value`. The exact-scheme payload has no such
field, so the message is naming something your payload was never supposed to have. The most likely cause in our case
(what we saw matches CDP's schema, and we did not re-test it against CDP) was the `nonce`: bare hex without the `0x`
prefix, where CDP's schema requires `^0x[0-9a-fA-F]{64}$`.

## What we saw

On 2026-11-09 our hand-built payer sent v2 exact-EVM payments to a seller whose server forwards to CDP
`/platform/v2/x402/verify`. Every attempt was refused before settlement, so no money moved. Two shapes of the
rejection:

```
error at "/paymentPayload/payload": input does not contain the discriminator property "type"
```

and, after we added `type`, with each of `eip3009`, `permit2`, `authorization` and `exact`:

```
error at "/paymentPayload/payload": discriminator property "type" has invalid value
```

Apart from the nonce prefix, the payload matched what the official SDK emits, and the signature itself was sound:
local recovery from the EIP-712 digest returned the right signer both before and after the change. That is what made
the nonce easy to miss. Our signer decoded the nonce to 32 bytes before hashing, so the prefix never entered the
digest, and a nonce without it signed and recovered exactly like one with it. Only the JSON on the wire differed.

## How to check

Check the payload against the patterns in CDP's published schema before you send anything. Here is the same check in
Node, if you want to run it first:

```js
const a = payload.payload.authorization;
const ok = {
  signature: /^0x[0-9a-fA-F]{130,}$/.test(payload.payload.signature),
  from: /^0x[0-9a-fA-F]{40}$/.test(a.from),
  to: /^0x[0-9a-fA-F]{40}$/.test(a.to),
  nonce: /^0x[0-9a-fA-F]{64}$/.test(a.nonce),
};
console.log(ok); // all true = the shape CDP expects
```

We ran that against a synthetic payload shaped like ours, with 64 hex characters and no prefix: the nonce check came
back false and the other three true. Add the prefix and all four are true. If your nonce line is the only false in
that object, you have probably found it.

The other fields matter as much as the pattern: `value`, `validAfter` and `validBefore` are JSON **strings**, not
numbers, and `signature` may be longer than 65 bytes because smart-account signatures are allowed
(`^0x[0-9a-fA-F]{130,}$`).

## The fix

Prefix the nonce with `0x`, and never add `payload.type` to an exact payload. No spelling of that field can work,
because the exact branch does not define one: the five values CDP's schema accepts for `type` belong to the
batch-settlement scheme, not to exact.[^cdp] The payload stays:

```json
{
  "x402Version": 2,
  "payload": {
    "signature": "0x...",
    "authorization": { "from": "0x...", "to": "0x...", "value": "10000",
                       "validAfter": "0", "validBefore": "1794441600",
                       "nonce": "0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef" }
  },
  "accepted": { "...": "the requirements you were given" }
}
```

If you would rather not hand-roll it, the official TypeScript client already does this: `createNonce()` in
`@x402/evm` is `toHex(getCrypto().getRandomValues(new Uint8Array(32)))`, and viem's `toHex` returns a `0x`-prefixed
string of 66 characters. `createEIP3009Payload()` puts exactly two keys in `payload`: `authorization` and
`signature`.[^sdk]

## Why the message names `type`

CDP's published OpenAPI document defines `x402V2PaymentPayload.payload` as a `oneOf` of five branches — exact EVM,
exact EVM Permit2, exact Solana, upto EVM Permit2 and batch settlement — and tags it
`x-oneof-no-discriminator`, meaning clients pick a branch by field presence, not by a tag. Of those five, only
`x402BatchSettlementEvmPayload` has a `type` discriminator, with values `deposit`, `voucher`, `refund`, `claim` and
`settle`.[^cdp] So our reading is that when the exact branch fails validation, the message about `type` is the last
branch's complaint, not one about the branch your payload was aiming at. That is the schema plus what we observed;
we have not found CDP documenting the order in which it reports branch failures.

It is not the only case of this schema naming the wrong thing. In x402-foundation/x402 issue #2832, sellers on CDP
got a union-mismatch message hinting at a missing top-level `scheme`, and the actual cause turned out to be
`resource.description` longer than 500 characters, a field the message never mentions.[^issue]

## What we did not test

We have not sent a corrected payment to CDP, and have not sent one since the fix. The fix is verified against CDP's
published patterns — the check above — and by recovering the signer from the corrected payload locally, not by
getting an `isValid: true` from CDP. If you can run one paid call, that is the last step we could not take, and it
would settle whether the validator is happy with the exact branch once the nonce is prefixed.

We also did not test other causes. The nonce was the only defect we found, and we have no public report of another
input producing this same message. If you hit it with a correctly prefixed nonce, the check above will tell you
whether the shape is the problem at all; if it is not, the payload you sent and the raw error are worth posting
upstream.

[^sdk]: x402-foundation/x402, `typescript/packages/mechanisms/evm/src/utils.ts` (`createNonce`) and
    `.../src/exact/client/eip3009.ts` (`createEIP3009Payload`) —
    https://github.com/x402-foundation/x402/blob/main/typescript/packages/mechanisms/evm/src/utils.ts and
    https://github.com/x402-foundation/x402/blob/main/typescript/packages/mechanisms/evm/src/exact/client/eip3009.ts,
    read 2026-11-11.
[^cdp]: Coinbase CDP SDK, `openapi.yaml` at commit a92e72a390, schemas `x402V2PaymentPayload`, `x402ExactEvmPayload`,
    `x402BatchSettlementEvmPayload` — https://github.com/coinbase/cdp-sdk/blob/a92e72a390/openapi.yaml, read
    2026-11-11.
[^issue]: https://github.com/x402-foundation/x402/issues/2832, read 2026-11-11.
