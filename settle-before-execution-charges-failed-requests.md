# x402-axum `settle_before_execution`: the buyer is charged even when your handler returns 400

If you turn on `settle_before_execution()` in x402-axum 2.0.2, your paid handler runs **after** the facilitator's
`/settle` call, so any error your handler returns — a 400 for a bad query parameter, a 404 for a row that moved, a
503 from a database that blinked — is answered to a buyer who has already been charged. The fix is to make every
check your handler would make *before* the payment middleware, or to leave settlement in its default after-the-handler
mode and give up the receipt inside the handler.

I am writing this because we hit it on our own server: one route settles first so its handler can bind a license to
the settlement transaction, and until 2026-11-09 a paid retry with a missing or unknown `?market=`
went to the facilitator's `/settle` before the handler answered 400 or 404 (we watched that on a guard
facilitator, and no settlement has ever run through our server — see "What we did not test").

## What the crate actually does, in order

Read from the source of x402-axum 2.0.2, which is still the newest version on crates.io (published 2026-07-13, checked 2026-11-12; <https://crates.io/crates/x402-axum>). In `paygate.rs`, when `settle_before_execution` is true the middleware:

1. decodes the payment header,
2. calls `settle_payment()` — straight to the facilitator, no `/verify` call first,
3. validates the settlement and puts `Some(settlement)` into the request extensions,
4. calls your handler,
5. inserts the `Payment-Response` header on whatever comes back.

There is no status check between steps 4 and 5. A handler error is returned exactly as the handler wrote it, with the
money already moved. In the default mode the order is different: `verify_payment()`, handler with `None` in the
extensions, and if the response is 4xx or 5xx it is returned *without* settling. Only a successful response reaches
`settle_payment()`. Source: <https://github.com/x402-rs/x402-rs/blob/v2.0.2/crates/x402-axum/src/paygate.rs> (read
2026-11-12).

The crate's own docs give the reason people turn it on: settling first "prevents issues where failed settlements need
retry or authorization expires" (the `Settlement Timing` section of `lib.rs` at the same tag, read 2026-11-12). That
reason is real. The cost is the one above, and the docs do not warn about it.

## How to check whether you are affected

You need one paid request your handler is guaranteed to refuse. A missing or bogus required query parameter is the
easy one.

1. Watch your facilitator's request log, or count `POST /settle` calls, while you send that request with a valid
   `PAYMENT-SIGNATURE` header.
2. If you see a `/settle` for a request your handler answered 400 or 404, you are charging for errors.
3. In settle-after mode the same test shows a `/verify` and no `/settle`.

We found ours this way: a guard facilitator that logged `POST /settle` with no `/verify` before it, paired with the
handler's own 400 in the same log. That pair of lines is
the whole proof, and it is worth adding to your own test suite rather than trusting a code read.

If you would rather not send a payment at all, read your own middleware order instead: any `settle_before_execution`
on a route whose handler can return an error is the same bug waiting for its first buyer.

## The fix we shipped

We put a small layer **outside** the payment middleware, in front of it in the router. If the request carries a
payment header (`PAYMENT-SIGNATURE` for v2, `X-PAYMENT` for v1) and the handler would refuse it — missing or unknown
query parameter, no data snapshot, wrong HTTP method — the layer answers with the same error the route would have
returned, and the payment is never submitted. The signed authorization simply expires. Unpaid requests pass
through untouched, so discovery traffic still gets its 402 challenge.

The layer also hands its validated rows to the handler through the request extensions, so the handler cannot disagree
with it and turn a paid call into a 404 after settlement anyway. Ours is roughly this shape:

```rust
pub async fn paid_precheck(mut req: Request, next: Next) -> Response {
    let paid = req.headers().contains_key("payment-signature")
        || req.headers().contains_key("x-payment");
    if paid {
        match check(req.method(), req.uri()) {   // the handler's own checks
            Ok(checked) => { req.extensions_mut().insert(checked); }
            Err(refusal) => return refusal,
        }
    }
    next.run(req).await
}
```

and it is attached *outside* the paygate layer, not inside it. We have a regression test that sends a paid retry with
a bad query parameter; its facilitator is a dead port, so if the paygate were reached the answer would be a 402
"Settlement failed", and the test asserts the handler's own 400/404/503 instead. That test is what keeps the bug
from coming back.

If your handler does not need the settlement receipt, the simpler fix is to delete `settle_before_execution()` and
keep the default. Then a 4xx or 5xx costs the buyer nothing and the receipt is only missing where there is nothing
to bind it to.

## The same class of bug in the official SDKs

This is not an x402-axum quirk. x402-foundation/x402 PR #3813, "fix: a HEAD request cannot settle a payment" (open,
opened 2026-10-14, seven comments when I read it on 2026-11-12), refuses to settle a payment carried on a HEAD
request that reached a paid route through the GET fallback, because a HEAD response carries no body to deliver. It
would make the same fix in the official Go, Python and TypeScript server middleware (today's releases still settle a
paid HEAD), and it keeps serving the challenge
over HEAD so the price stays discoverable:
<https://github.com/x402-foundation/x402/pull/3813>. The lesson carries: whichever stack settles, the place to refuse
is before the money moves.

## What we did not test

- No mainnet settlement has ever gone through our server, in either mode. We have zero settlements, so the buyer who
  would have been charged in our case was a signed authorization that expired unused, not a real payment.
- The settle-after behaviour ("4xx/5xx is returned unsettled") is read from the source above, not exercised with a
  real payment.
- We only ran this on x402-axum 2.0.2 under axum. The Go, Python and TypeScript behaviour is what PR #3813 says,
  read, not run.
