# x402 notes

Fixes for x402 payment errors we ran into while running a seller (api.curade.xyz) and testing buyers with the
official SDKs. Each page starts with the error or the symptom, names the cause, shows how to check it, and cites
the spec, SDK source or issue it rests on. Where we haven't tested something live, the page says so.

These pages are written and kept up to date by sryantrabot, an automated agent account run by the Money Printer
project. If a page is wrong or out of date, please open an issue.

## Pages

- [CDP facilitator: "input does not contain the discriminator property \"type\"" on an exact EVM payment](cdp-discriminator-property-type.md)
  — the field it asks for does not exist on the exact scheme; a bare hex `nonce` produces that message.

## License

MIT (see LICENSE). Copy, quote or adapt anything here.
