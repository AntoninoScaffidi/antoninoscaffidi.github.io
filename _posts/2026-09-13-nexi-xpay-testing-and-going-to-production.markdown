---
layout: post
title: "Nexi XPay with Rails: Testing, Local Verification, and Going to Production"
series: "nexi-xpay-with-rails"
episode: 3
lang: en
ref: nexi-xpay-testing-and-going-to-production
permalink: /nexi-xpay-testing-and-going-to-production/
canonical_url: https://antoninoscaffidi.github.io/nexi-xpay-testing-and-going-to-production/
image: /assets/images/nexi-xpay-ep3-banner.png
date: 2026-09-13 08:00:00 +0200
---

[Episode 1]({% post_url 2026-09-13-nexi-xpay-setup-and-initiating-a-payment %}) built the request side, [episode 2]({% post_url 2026-09-13-nexi-xpay-handling-the-outcome %}) built the response side. 19 automated tests pass, and every one of them runs without touching the network — which is exactly the point of a test suite, but it also means none of them actually prove Nexi's real servers accept what this app sends them. This episode closes that gap: a real payment, against the real sandbox, with the webhook arriving on its own.

Code is tagged [`episode-3`](https://github.com/AntoninoScaffidi/nexi-xpay-with-rails/tree/episode-3) in the [nexi-xpay-with-rails](https://github.com/AntoninoScaffidi/nexi-xpay-with-rails) repo — this one's lighter on new application code and heavier on process, since testing and verifying are the actual subject.

## Why the test suite alone isn't enough

Every test written across the last two episodes constructs its own MAC using the app's own `mac_key` and checks the app can verify its own signature — a real, meaningful check, but a closed loop. It proves the *algorithm* is implemented correctly; it says nothing about whether Nexi's servers happen to compute that same algorithm the same way for *your* specific terminal configuration, whether your sandbox alias actually works, or whether a notification can really reach your server from the outside. Those things only get confirmed by an end-to-end run against the real thing.

## Nexi can't reach `localhost`

This is the practical blocker: `/nexi/notify` needs to be reachable from Nexi's own servers, and `localhost:3000` isn't reachable from anywhere but your machine. The fix is a public tunnel:

```bash
brew install --cask ngrok
ngrok config add-authtoken <token>   # free account, from ngrok.com
bin/dev                               # or: bin/rails server -p 3000
ngrok http 3000
```

`ngrok` prints an `https://something.ngrok-free.app` URL. Two things have to be true for the rest of this to work:

**You have to actually browse from that URL, not `localhost`.** `OrdersController#create` builds `notify_url` from whatever host the current request came in on — visit from `localhost` and Nexi gets handed a `notify_url` it can never reach, no matter how correctly everything else is built.

**Rails has to be told to trust the ngrok host**, or `ActionDispatch::HostAuthorization` rejects the request before your controller ever sees it:

```ruby
# config/environments/development.rb
config.hosts << /.*\.ngrok-free\.(app|dev)/
```

Without this line, visiting the ngrok URL shows Rails' "Blocked hosts" error page — a real gotcha, not a hypothetical one, and an easy one to lose ten minutes to if you don't already know `config.hosts` is the mechanism.

## A real test card, and a real payment

Nexi's sandbox backoffice (the same **Area test** page episode 1 pointed at for credentials) hands out test card numbers:

- `4539 9700 0000 0006` — Visa, approved
- `4539 9700 0000 0014` — Visa, declined (useful for exercising the `KO` branch of `#notify` on purpose)

Any future expiry date, any 3-digit CVV. 3D Secure authentication in sandbox always accepts the OTP `123456`.

Running the full flow — checkout, redirect to Nexi, card entry, 3D Secure — and then checking what actually arrived confirms the whole thing works, not just each piece in isolation. Here's an actual notification received on `/nexi/notify` during this exact test, redacted only where it genuinely doesn't matter:

```
Started POST "/nexi/notify" for 185.198.117.20
Processing by NexiCallbacksController#notify as HTML
Parameters: {
  "codTrans"=>"19JWR3YNVOKC",
  "esito"=>"OK",
  "importo"=>"1200",               # €12.00
  "divisa"=>"EUR",
  "data"=>"20260912", "orario"=>"225428",
  "codAut"=>"MP7EL4",              # real authorization code
  "pan"=>"453997******0006",       # masked, never in clear
  "brand"=>"VISA",
  "tipoTransazione"=>"VBV_FULL",   # full 3D Secure
  "mac"=>"9fee38d052f87f5ac6548ce0a9ba20c2d9933e8e",
}
```

No manual step involved anywhere in that log — Nexi's own servers found the ngrok URL, POSTed to it, and the app marked the order paid entirely on its own. That's the actual bar for "this works," and it's a meaningfully higher one than "the tests are green."

One thing worth knowing if you go looking at your own log: **Nexi may send more fields than any table in its documentation lists** — a merchant-configured field like `num_contratto`, or terminal-specific ones like `check`/`aliasEffettivo`, depending on what's been set up in your specific backoffice profile. That's exactly why `raw_notification_params` captures the entire payload indiscriminately rather than picking out a fixed list of expected keys — there's no way to know in advance the complete set a given terminal will actually send.

## Seed data, for anyone trying this themselves

```ruby
# db/seeds.rb
Product.find_or_create_by!(name: "Sample experience") do |product|
  product.price = 12.00
  product.purchasable = true
end
```

```bash
bin/rails db:seed
```

Low price on purpose — the sandbox environment has limits, and there's no reason to test near them.

## What's genuinely still missing before production

This series builds a complete, *tested*, *verified* payment flow — deliberately not a production-ready one. Specifically still open:

- **`nexi.production` credentials** — a real terminal alias and MAC key, added to `credentials.yml.enc` alongside `nexi.sandbox` once you're ready to accept real money. The production endpoint drops the `int-` prefix: `https://ecommerce.nexi.it/...`.
- **A confirmation email**, sent from `#notify` after `mark_paid!` — never from `#return`, since that action is UX-only and can be skipped by the customer entirely.
- **An admin view onto orders and notifications** — read-mostly, with an explicit `mark_refunded` action if refunds are needed, never a generic `update` that could touch financial fields.
- **Explicitly re-validating `importo`/`divisa` from the notification against `order.total_amount`/`order.currency`** before calling `mark_paid!`. The MAC already guarantees Nexi didn't send something a tampered notification would fail to sign correctly — this would be defense-in-depth on top of that, not a fix for a known hole.
- **Turning `purchasable` on**, one real product at a time, once everything above is in place.

None of these change how the core flow works — they're what separates "this correctly processes a payment" from "this is what a real business runs unattended."

## Wrapping up

Three episodes, one small but complete Rails app: a checkout that never trusts a price from the client, a payment gateway that signs what it sends and verifies what it receives, an idempotent webhook that never double-processes the same notification, and a test suite plus a real sandbox run standing behind all of it. If you're integrating XPay yourself, the [full repo](https://github.com/AntoninoScaffidi/nexi-xpay-with-rails) is there to clone, read, and adapt.
