# Browser x402 client gets a 402 but `PAYMENT-REQUIRED` is null: the CORS headers your server needs

Your x402 endpoint answers `curl` with a 402 and a `PAYMENT-REQUIRED` header, but the same call from a page in a
browser gives you `r.status === 402` and `r.headers.get("PAYMENT-REQUIRED") === null`, or the paid retry dies as
`TypeError: Failed to fetch` with no status at all. The cause is CORS: none of the three x402 headers is
CORS-safelisted, so the browser hides `PAYMENT-REQUIRED` from JavaScript unless the response names it in
`Access-Control-Expose-Headers`, and it refuses to send `PAYMENT-SIGNATURE` unless the preflight allows that header.
The fix is `Access-Control-Allow-Origin` and `Access-Control-Expose-Headers` on the 402, an allow list naming the
payment header on the preflight, and the CORS layer outside the payment middleware rather than inside it.

## How to check

Two requests against your own URL. The first is the unpaid call a browser buyer makes; the pass condition is an
`Access-Control-Allow-Origin` line and an expose list that includes `payment-required`.

```sh
curl -si -H 'Origin: https://example.com' https://your-host/paid | grep -i '^access-control'
```

```sh
curl -si -X OPTIONS https://your-host/paid \
  -H 'Origin: https://example.com' \
  -H 'Access-Control-Request-Method: GET' \
  -H 'Access-Control-Request-Headers: payment-signature,access-control-expose-headers'
```

Failures, in the order they usually appear:

- no `Access-Control-Allow-Origin` on the first call: the browser discards the whole response, and the client
  reports a fetch failure with no status;
- `Access-Control-Allow-Origin` present but `Access-Control-Expose-Headers` missing or not naming
  `payment-required`: the 402 arrives, the price is invisible, and the client fails to parse an apparently empty
  challenge;
- the second call answers 402 instead of 2xx, so the browser never sends the paid request at all.

## The fix

Whatever framework you use, the shape is the same: CORS runs outside the payment middleware, it allows every origin
(the payment is the authentication; there are no cookies), it exposes the payment response headers, the v1
`X-PAYMENT-RESPONSE` included, and it answers `OPTIONS` itself instead of letting the paywall price it.

Here is the tower-http 0.6 layer we run in our own axum server, applied as the outermost layer so it wraps the
payment gate:

```rust
use axum::http::{HeaderName, Method};
use tower_http::cors::{AllowHeaders, Any, CorsLayer};

CorsLayer::new()
    .allow_origin(Any)
    .allow_methods([Method::GET, Method::HEAD, Method::POST, Method::OPTIONS])
    .allow_headers(AllowHeaders::mirror_request())
    .expose_headers([
        HeaderName::from_static("payment-required"),
        HeaderName::from_static("payment-response"),
        HeaderName::from_static("x-payment-response"),
    ])
    .max_age(std::time::Duration::from_secs(86_400))
```

We use `AllowHeaders::mirror_request()` rather than a fixed list on purpose (see the third point under "Why it
happens"). If your framework's CORS layer cannot mirror the request, list `PAYMENT-SIGNATURE`,
`Access-Control-Expose-Headers` and `Content-Type` explicitly. For Express
([cors middleware](https://expressjs.com/en/resources/middleware/cors.html)), Hono
([cors](https://hono.dev/docs/middleware/builtin/cors)), FastAPI
([CORSMiddleware](https://fastapi.tiangolo.com/tutorial/cors/)), Flask
([Flask-CORS](https://github.com/corydolphin/flask-cors)) and Next.js
([headers](https://nextjs.org/docs/app/api-reference/config/next-config-js/headers)) we point at their own CORS
documentation rather than guess: the ordering rule is what matters, and where a framework's middleware writes the
402 before its CORS hooks run, the header never lands. The upstream issue below covers Flask specifically, where
flask-cors does not touch the 402.

## What it looks like when it works

From `api.curade.xyz`, our own seller, on 2026-11-12 — CORS and payment lines only, with `Alt-Svc`, `Date`,
`Server` and `Vary` elided:

```
GET /v1/funding/current  Origin: https://example.com
HTTP/1.1 402 Payment Required
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: payment-required,payment-response,x-payment-response
Content-Length: 0
Payment-Required: eyJhY2NlcHRzIjpbeyJhbW91bnQiOiIxMDAwIi...
```

```
OPTIONS /v1/funding/current
Access-Control-Request-Method: GET
Access-Control-Request-Headers: payment-signature,access-control-expose-headers
HTTP/1.1 200 OK
Access-Control-Allow-Headers: payment-signature,access-control-expose-headers
Access-Control-Allow-Methods: GET,HEAD,POST,OPTIONS
Access-Control-Allow-Origin: *
Access-Control-Max-Age: 86400
```

Before we added the layer, the preflight on that host was answered by the payment gate with a 402, and the unpaid
402 carried neither `Access-Control-Allow-Origin` nor `Access-Control-Expose-Headers`. One layer, placed
outermost, fixed both.

## Why it happens

**The three headers are ordinary custom headers.** A cross-origin `fetch()` only exposes a small safelisted set of
response headers to JavaScript; `PAYMENT-REQUIRED` and `PAYMENT-RESPONSE` are not in it. The x402 HTTP transport
spec has a "Browser clients" section covering all of this, including the v1 spellings and the
`Vary: Origin` advice for servers that reflect an origin instead of sending `*`.[^spec] What this page adds is the working layer above and the two commands, not the diagnosis.

**The preflight can hit the paywall.** `PAYMENT-SIGNATURE` is not a safelisted request header, so the paid retry is
preflighted. An `OPTIONS` request carries no payment, and a payment middleware that matches by path will price it and
return 402. Registered before the handler but outside the payment gate, the CORS layer answers the preflight itself
and the paid request is sent.

**The TypeScript client adds a header of its own.** In `@x402/fetch` 2.30.0 — the version we ran — the paid retry
also sets the *request* header `Access-Control-Expose-Headers: PAYMENT-RESPONSE,X-PAYMENT-RESPONSE`, which is a
response-only directive in the wrong place. The browser therefore lists it in `Access-Control-Request-Headers`
alongside `payment-signature`, and a server whose allow list names exactly the protocol headers refuses that
preflight while every other check passes. Pull request #3478 is open to remove it from the clients.[^3478] Mirroring the
requested headers, as we do, survives it either way; if you keep a fixed allow list, add
`access-control-expose-headers` to it.

**Do not answer with `*` in `Access-Control-Expose-Headers`.** It works for `Allow-Origin` on an uncredentialed
request — which is why we use it there — but on a credentialed request `*` is read as a literal header name, so a
buyer that sends credentials sees the 402 and none of the payment headers.

## What we did not test

We have not watched this fail in a real browser: our checks are the same two requests as the curl commands above,
sent by hand over TLS against our own server, plus Rust integration tests that send a simulated preflight with
`Access-Control-Request-Headers: payment-signature,access-control-expose-headers` and assert a 2xx plus the expose
list on the 402. The headless-Chromium results in the upstream issue are not ours. We also have not run the Express,
Hono, FastAPI, Flask or Next.js middlewares, so we give no snippets for them.

We have not tested whether the `@x402/fetch` request header is still present in versions after 2.30.0; we read 2.30.0
only. And we make no claim about whether your endpoint should accept browser origins at all: refusing them is a
decision about who may read your resource, not a payment control.

[^spec]: x402-foundation/x402, `specs/transports-v2/http.md`, section "Browser clients" —
    https://github.com/x402-foundation/x402/blob/main/specs/transports-v2/http.md, read 2026-11-12.
[^3478]: x402-foundation/x402 pull request #3478, "fix(http): avoid response-only CORS headers in client retries" — https://github.com/x402-foundation/x402/pull/3478, read 2026-11-12. The server-side report with the
    Chromium matrix is issue #3722, https://github.com/x402-foundation/x402/issues/3722, read 2026-11-12.
