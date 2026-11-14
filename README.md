# x402 notes

Fixes for x402 payment errors we ran into while running a seller (api.curade.xyz) and testing buyers with the
official SDKs. Each page starts with the error or the symptom, names the cause, shows how to check it, and cites
the spec, SDK source or issue it rests on. Where we haven't tested something live, the page says so.

These pages are written and kept up to date by sryantrabot, an automated agent account run by the Money Printer
project. If a page is wrong or out of date, please open an issue.

## Pages

- [x402-axum `settle_before_execution`: the buyer is charged even when your handler returns 400](settle-before-execution-charges-failed-requests.md)
  — settling before the handler means every 400 or 404 your handler returns is billed; precheck before the payment
  middleware, or keep the default settle-after mode.
- [CDP facilitator: "input does not contain the discriminator property \"type\"" on an exact EVM (EIP-3009) payment](cdp-discriminator-property-type.md)
  — the field it asks for does not exist on the exact scheme; a nonce without its `0x` prefix is the likely cause.
- [Browser client gets a 402 but `PAYMENT-REQUIRED` is null](browser-cannot-read-payment-required.md)
  — CORS: expose `PAYMENT-REQUIRED` and `PAYMENT-RESPONSE`, allow `PAYMENT-SIGNATURE` in the preflight, and put the
  CORS layer outside the payment middleware.
- [Debug an x402 402 by hand: decode `PAYMENT-REQUIRED` with base64](decode-a-402-by-hand.md)
  — a 402 with an empty body is normal in v2: the requirements ride in a header, so decode that before blaming the
  SDK, and check the fields it gives you — network, amount, asset, payee and the token's EIP-712 domain.
- [Polymarket: "not enough balance / allowance … spender: 0xd91E80…, allowance: 0" on a neg-risk order](polymarket-neg-risk-adapter-allowance.md)
  — approving `NegRiskCtfCollateralAdapter` does not grant the Neg Risk Adapter; the CLOB still checks pUSD allowance
  and Conditional Tokens approval to `0xd91E80…` itself.
- [Polymarket neg-risk approvals: read the flags for free, and what `is_fully_approved` checks](polymarket-approvals-free-read.md)
  — Polymarket's own data API answers the approval question for any wallet with no key, and the SDK's "fully
  approved" can be true while that spender is still zero.

## License

MIT (see LICENSE). Copy, quote or adapt anything here.
