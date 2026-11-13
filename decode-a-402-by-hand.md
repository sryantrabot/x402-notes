# Debug an x402 v2 402 by hand: decode `PAYMENT-REQUIRED` with curl and base64

Your paid API answers `402 Payment Required`, the body is empty, and nothing in the response says what you are
supposed to pay. In x402 v2 that is not a broken response: the payment requirements travel base64-encoded in a
`PAYMENT-REQUIRED` header, and the body is allowed to be empty.[^http] Decoding it by hand gives you the whole
ask in one object: version, network, amount, token, payee. Do that before you touch an SDK.

## The command

```sh
curl -si https://your-host/paid \
  | awk 'tolower($1)=="payment-required:"{print $2}' \
  | tr -d '\r' \
  | base64 -d
```

Add `| jq .` if you have jq. The `tr -d '\r'` matters when you paste a header out of a terminal: a stray carriage
return makes `base64` stop early or fail with `invalid input`. `curl -i` prints the status line and every header, so the
`awk` stage picks the one you want; header names are case-insensitive on the wire and servers spell this one both
`PAYMENT-REQUIRED` and `Payment-Required`, which is what `tolower($1)` handles.

I ran the header-selection, `tr` and `base64` stages of that pipeline on 2026-11-13 against our own seller,
`api.curade.xyz`. This machine has no curl, so I sent the request with `openssl s_client` and piped the raw
response into them; for your own server you will just use curl.

## What came back

Payment and CORS lines only, rest of the headers elided:

```
GET /v1/funding/current
HTTP/1.1 402 Payment Required
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: payment-required,payment-response,x-payment-response
Content-Length: 0
Payment-Required: eyJhY2NlcHRzIjpbeyJhbW91bnQiOiIxMDAwIiwiYXNzZXQiOiIweDgzMzU4...
```

and the decoded object, keys reordered so `x402Version` reads first, with the `extensions.bazaar` block elided:

```json
{
  "x402Version": 2,
  "error": "Payment-Signature header is required",
  "resource": {
    "url": "https://api.curade.xyz/v1/funding/current",
    "description": "Latest Hyperliquid funding-rate snapshot",
    "mimeType": "application/json",
    "serviceName": "Curade",
    "tags": ["polymarket", "prediction-markets", "x402", "data-api"],
    "iconUrl": "https://api.curade.xyz/icon.svg"
  },
  "accepts": [
    {
      "scheme": "exact",
      "network": "eip155:8453",
      "amount": "1000",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "payTo": "0x56641A5A6C20E0673f1F58844a19136CFA24AF8F",
      "maxTimeoutSeconds": 300,
      "extra": { "assetTransferMethod": "eip3009", "name": "USD Coin", "version": "2" }
    }
  ]
}
```

One server's shape, not the only shape: `error` and `extensions` are optional, and other servers fill `accepts`
differently.[^v2] Only `resource.url` is required inside `resource`.

## Field by field

**`x402Version` must be 2.** If you see `1` (or a `maxAmountRequired` field), you are on the v1 protocol: the
requirements sit in the JSON body rather than in a header, the network is a name like `base-sepolia` instead of
CAIP-2, and the paid retry goes in `X-PAYMENT`, not `PAYMENT-SIGNATURE`.[^v1]

**`network` is CAIP-2.** `eip155:8453` is Base mainnet and `eip155:84532` is Base Sepolia, a testnet with test
USDC; I checked both by asking a public RPC for `eth_chainId` on 2026-11-13, which answered `0x2105` and `0x14a34`.

**`amount` is atomic units of `asset`, not dollars.** `1000` with USDC's six decimals is $0.001: divide by
`10^decimals` for a human number, and compare against the asset contract, not a ticker.

**`asset` is the token contract.** `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` is USDC on Base. What has to
hold is that you hold that token on that chain.

**`payTo` is the recipient.** Check it against the address the provider publishes. A 402 you cannot trace to the party
you meant to pay is a reason not to sign.

**`extra.name` and `extra.version` are the EIP-712 domain of that token.** They have to match the contract's own
`name()` and `version()`, because they are hashed into the signature you will produce.[^evm] You can read both from
a public Base RPC with `eth_call`, selector `0x06fdde03` for `name()` and `0x54fd4d50` for `version()`:

```sh
curl -s https://mainnet.base.org -H 'content-type: application/json' -d '{
  "jsonrpc":"2.0","id":1,"method":"eth_call",
  "params":[{"to":"0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913","data":"0x54fd4d50"},"latest"]}'
```

The answer on 2026-11-13 was an ABI-encoded string: offset `0x20`, length `0x1`, then `0x32` and 31 zero bytes.
`0x32` is the ASCII character `2`. Running the same call with `0x06fdde03` gave offset `0x20`, length `0x8` and
`55534420436f696e`, the ASCII for `USD Coin`. Our header says `"USD Coin"` and `"2"`, and the contract agrees. If
yours disagree, that mismatch is a real cause of signature rejections; we have not reproduced the specific error a
facilitator returns for it, so check the domain by hand rather than searching for a message.

**`maxTimeoutSeconds` is how long you have.** The spec calls it the maximum time allowed for payment completion.
The authorization you sign carries `validAfter` and `validBefore`, and `300` here is the window the server is
prepared to accept, so a clock that is off or an approval prompt left open for longer than that can produce a
payment the facilitator rejects as expired.

**`error` is prose, not a code.** Our server writes `Payment-Signature header is required`, and the v2 spec's own
example writes `PAYMENT-SIGNATURE header is required`; the v1 core spec example says `X-PAYMENT header is
required` while the v1 transport example says `Payment required to access this resource`. Branch on
`errorReason` codes in a settlement response, never on this text.[^v2]

## When it does not decode

No `PAYMENT-REQUIRED` header at all, and a JSON body holding `accepts` with `"x402Version": 1`: that is the v1
shape, where the requirements live in the body. Read the body and skip the base64.

Header present in curl, invisible to a page in a browser: CORS. A cross-origin `fetch()` cannot read
`PAYMENT-REQUIRED` unless the response names it in `Access-Control-Expose-Headers`, so the response arrives and
JavaScript sees `null`. Our seller sends that expose list, shown above, which is why it is readable there; a server
that omits it looks identical over curl. That failure has its own page:
[browser-cannot-read-payment-required.md](browser-cannot-read-payment-required.md).

Decoded text ends mid-object: a truncated copy-paste, not a truncated server response. `curl -i` writes one long
header line, and a copy from a terminal that wrapped it on screen loses the tail; pipe it as above instead.

Decodes fine, but the paid retry is rejected with a message about `type` in the payload: that is the payload you
built, not the requirements. See [cdp-discriminator-property-type.md](cdp-discriminator-property-type.md).

## The receipt

After a successful paid call the same encoding comes back in `PAYMENT-RESPONSE`:
`{"success":true,"transaction":"0x…","network":"eip155:8453","payer":"0x…"}`, and on failure
`{"success":false,"errorReason":"insufficient_funds",…}`. Decode it with the same three stages. We have not seen a
paid call settle on our own server yet, so the object above is the spec's example, not a receipt of ours.[^http]

## What we did not test

We decoded one seller's 402 — ours — and read the token contract and both chain ids. We did not build or sign a
payment, nothing here was tested live against a paid call, and we did not try the command against v1 servers or
Solana, or servers whose `accepts` offers more than one option. If your header decodes
but the numbers still look wrong to you, the object is the ground truth: take it to the provider rather than to a
client library.

[^v1]: x402-foundation/x402, `specs/x402-specification-v1.md` §5.1 (`maxAmountRequired`, string `network`) —
    https://github.com/x402-foundation/x402/blob/main/specs/x402-specification-v1.md, and
    `specs/transports-v1/http.md` ("Mechanism: HTTP 402 status code with JSON response body", `X-PAYMENT`) —
    https://github.com/x402-foundation/x402/blob/main/specs/transports-v1/http.md, both read 2026-11-13.
[^v2]: x402-foundation/x402, `specs/x402-specification-v2.md` §5.1.2 field tables (`amount`, CAIP-2 `network`,
    optional `error`, `ResourceInfo`) and §9 (`errorReason` codes such as `insufficient_funds`) —
    https://github.com/x402-foundation/x402/blob/main/specs/x402-specification-v2.md, read 2026-11-13.
[^http]: x402-foundation/x402, `specs/transports-v2/http.md`, "Response Body" and the `SettlementResponse`
    examples — https://github.com/x402-foundation/x402/blob/main/specs/transports-v2/http.md, read 2026-11-13.
[^evm]: x402-foundation/x402, `specs/schemes/exact/scheme_exact_evm.md`, `eip3009` extra fields: "The EIP-712
    domain name of the token contract. Used for `transferWithAuthorization` signature construction" —
    https://github.com/x402-foundation/x402/blob/main/specs/schemes/exact/scheme_exact_evm.md, read 2026-11-13.
