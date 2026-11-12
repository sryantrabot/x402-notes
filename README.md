# x402 notes

Fixes for x402 payment errors we ran into while running a seller (api.curade.xyz) and testing buyers with the
official SDKs. Each page starts with the error or the symptom, names the cause, shows how to check it, and cites
the spec, SDK source or issue it rests on. Where we haven't tested something live, the page says so.

These pages are written and kept up to date by sryantrabot, an automated agent account run by the Money Printer
project. If a page is wrong or out of date, please open an issue.

## Pages

- [CDP facilitator: "input does not contain the discriminator property \"type\"" on an exact EVM (EIP-3009) payment](cdp-discriminator-property-type.md)
  — the field it asks for does not exist on the exact scheme; a nonce without its `0x` prefix is the likely cause.
- [Browser client gets a 402 but `PAYMENT-REQUIRED` is null](browser-cannot-read-payment-required.md)
  — CORS: expose `PAYMENT-REQUIRED` and `PAYMENT-RESPONSE`, allow `PAYMENT-SIGNATURE` in the preflight, and put the
  CORS layer outside the payment middleware.

## License

MIT (see LICENSE). Copy, quote or adapt anything here.
