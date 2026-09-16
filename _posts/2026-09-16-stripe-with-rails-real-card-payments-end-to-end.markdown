---
layout: post
title: "Stripe with Rails: Real Card Payments, End to End"
lang: en
ref: stripe-with-rails-real-card-payments-end-to-end
permalink: /stripe-with-rails/
canonical_url: https://antoninoscaffidi.github.io/stripe-with-rails/
image: /assets/images/stripe-banner.png
date: 2026-09-16 20:38:00 +0200
---

A new series just started: **Stripe with Rails** — integrating [Stripe Checkout](https://stripe.com) into a Ruby on Rails app, from a real `rails new` to a working, tested payment flow. It deliberately reuses the same `Product`/`Order` domain as [Nexi XPay with Rails]({% post_url 2026-09-13-nexi-xpay-setup-and-initiating-a-payment %}), so what's genuinely different between a European "in-house" gateway and a modern, developer-first American one shows up in the code itself, not just in prose.

![Stripe with Rails: real card payments, end to end — a code card showing Stripe::Checkout::Session.create, with the flow checkout → hosted page → webhook verified.](/assets/images/stripe-banner.png)

## The one-line mental model

Nexi's "Pagamento Semplice" is a **browser-submitted, signed form POST**: your Rails app builds a form with hidden fields and a hand-rolled MAC, and the *browser* sends it straight to Nexi's servers. Your backend never talks to Nexi directly to start a payment.

Stripe Checkout inverts that. Your server calls Stripe first:

```ruby
session = Stripe::Checkout::Session.create(
  mode: "payment",
  line_items: [{
    price_data: {
      currency: "eur",
      product_data: { name: "Sicilian cooking class" },
      unit_amount: 3500 # cents, per unit — never the total
    },
    quantity: 1
  }],
  client_reference_id: order.order_number,
  success_url: stripe_success_url,
  cancel_url: stripe_cancel_url
)

redirect_to session.url, allow_other_host: true
```

Stripe hands back a URL, and *then* you send the browser there. No MAC to compute by hand — the request itself is authenticated by a `Bearer sk_test_...` header the official gem attaches for you.

## Same two-channel confirmation, different signature

Both gateways solve the same problem the same way: a redirect back to your site is UX-only and can simply never happen (closed tab, crashed browser), so it can never be the thing that marks an order paid. The only source of truth is a message the gateway sends **your server directly**:

| | Nexi | Stripe |
|---|---|---|
| Async channel | S2S notification, MAC-signed | Webhook event, HMAC-signed |
| Sync channel (UX only) | Browser redirect + `esito` field | Browser redirect to `success_url`/`cancel_url` |
| Your merchant reference | `codTrans` | `client_reference_id` |

That's the shape of episode 2: verifying a Stripe webhook signature with `Stripe::Webhook.construct_event`, side by side with the hand-written SHA1 MAC check from the Nexi series.

## What episode 1 actually built

Not a sketch — a real Rails 8.1 app, Postgres, Tailwind, git-tagged per episode, with its own Minitest suite:

```ruby
test "create_session sends the price in cents, not decimal euros" do
  order = orders(:pending_order)
  captured_params = nil
  fake_session = OpenStruct.new(id: "cs_test_123", url: "https://checkout.stripe.com/pay/cs_test_123")

  Stripe::Checkout::Session.stub(:create, ->(params) { captured_params = params; fake_session }) do
    StripeCheckoutService.create_session(
      order: order, success_url: "https://example.com/success", cancel_url: "https://example.com/cancel"
    )
  end

  line_item = captured_params[:line_items].first
  assert_equal 3500, line_item[:price_data][:unit_amount]
  assert_equal "eur", line_item[:price_data][:currency]
end
```

11 tests, no live Stripe calls, and along the way, two real bugs from gems published earlier that same day (`json` 3.0.2 breaking `ActiveSupport::JSON.decode`, `minitest` 6.0.6 dropping `Minitest::Mock` from the core gem) — plus a genuinely subtle Turbo Drive gotcha: a cross-origin redirect silently swallowed by a `fetch()`-based form submission, diagnosed live in the browser before the fix.

## Read episode 1

[Setting Up and Initiating a Payment]({% post_url 2026-09-16-stripe-setup-and-initiating-a-payment %}) covers all of it in full — the Stripe Checkout mechanism, the domain model, the `StripeCheckoutService`, a from-first-principles explanation of what a webhook actually is and why Checkout needs one, and both bugs in detail.

## What's next

- **Episode 2** — handling the outcome: verifying the webhook signature, mapping `checkout.session.completed` to `Order#mark_paid!`, testing locally with the Stripe CLI instead of ngrok.
- **Episode 3** — going to production: a real audit trail for incoming events, handling `checkout.session.expired` and `payment_intent.payment_failed`, not just the happy path.

Companion code: [github.com/AntoninoScaffidi/stripe-with-rails](https://github.com/AntoninoScaffidi/stripe-with-rails), tagged `episode-1`.
