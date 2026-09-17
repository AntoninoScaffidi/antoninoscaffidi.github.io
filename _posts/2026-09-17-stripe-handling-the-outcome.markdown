---
layout: post
title: "Stripe with Rails: Handling the Outcome"
series: "stripe-with-rails"
episode: 2
lang: en
ref: stripe-handling-the-outcome
permalink: /stripe-handling-the-outcome/
canonical_url: https://antoninoscaffidi.github.io/stripe-handling-the-outcome/
image: /assets/images/stripe-ep2-banner.png
date: 2026-09-17 22:44:00 +0200
---

[Episode 1]({% post_url 2026-09-16-stripe-setup-and-initiating-a-payment %}) got a customer from "buy now" to Stripe's hosted checkout page, with `StripeCallbacksController#webhook` left as a stub — an endpoint that existed only so Stripe had somewhere to send its confirmation. This episode fills it in for real: verifying that a webhook genuinely came from Stripe, marking orders paid, and surviving retried deliveries — plus two real bugs hit while building it, both worth knowing about beyond this one app.

Code is tagged [`episode-2`](https://github.com/AntoninoScaffidi/stripe-with-rails/tree/episode-2) in the [stripe-with-rails](https://github.com/AntoninoScaffidi/stripe-with-rails) repo.

## Two channels, same principle as Nexi, different shape

Once a customer finishes on Stripe's checkout page, Stripe talks back through two independent channels — exactly the same split as the [Nexi series]({% post_url 2026-09-13-nexi-xpay-handling-the-outcome %}), just with different names:

- **A browser redirect** (`success_url`/`cancel_url`) — UX only. If the customer closes the tab the instant payment finishes, this redirect simply never happens.
- **A webhook event** (`POST /stripe/webhook`) — Stripe's own servers call your backend directly, with no dependency on the customer's browser still being open. This is the only channel this app trusts to change an order's state.

Nexi routes both outcomes through one shared `#return` action that reads an `esito` field to decide what to show. Stripe gives each its own named route instead — but `StripeCallbacksController#success` and `#cancel` still do nothing but redirect; neither one is allowed anywhere near `Order#mark_paid!`. Only `#webhook` does that.

## `StripeEvent`: an audit log, not a domain model

```bash
bin/rails generate model StripeEvent stripe_event_id:string event_type:string raw_payload:jsonb
```

```ruby
# app/models/stripe_event.rb
class StripeEvent < ApplicationRecord
  validates :stripe_event_id, presence: true, uniqueness: true
  validates :event_type, presence: true
end
```

Same role as `OrderPaymentNotification` in the Nexi series: a record of *every* verified event Stripe sends, kept separate from the `Order` it might affect. The unique index on `stripe_event_id` isn't decoration — it's the actual idempotency mechanism, covered below.

## Verifying the signature: no MAC to write by hand this time

Episode 1 already noted that creating a Checkout Session needs no manual signature — the gem authenticates outgoing requests with a Bearer token. Verifying an *incoming* webhook is the one place Stripe does ask you to check a signature yourself, and the official gem does the cryptography:

```ruby
event = Stripe::Webhook.construct_event(payload, sig_header, webhook_secret)
```

One call replaces all of what `Nexi::Gateway#verify_mac` had to do by hand — concatenate fields, `SHA1.hexdigest`, `secure_compare` against the received value. Under the hood `construct_event` does the equivalent (HMAC-SHA256 over `"#{timestamp}.#{raw_body}"`, constant-time compare, plus a timestamp-tolerance check the hand-rolled Nexi version never had), but as the caller you never see any of it — just a `Stripe::SignatureVerificationError` if it fails.

**A bug hit writing the tests for this**: several Stripe guides (and older versions of the gem) reference `Stripe::Webhook.generate_test_header` as the way to sign a payload in a test. It doesn't exist in `stripe` 13.5.1 — a `NoMethodError` confirmed it outright. The real API in this version is one level down:

```ruby
timestamp = Time.current
signature = Stripe::Webhook::Signature.compute_signature(timestamp, payload, secret)
sig_header = Stripe::Webhook::Signature.generate_header(timestamp, signature)
```

Worth knowing before copying a webhook-testing snippet from an older tutorial into a current `Gemfile`.

## `StripeCallbacksController`, for real this time

```ruby
# app/controllers/stripe_callbacks_controller.rb
class StripeCallbacksController < ActionController::Base
  skip_before_action :verify_authenticity_token, only: [ :webhook ]

  def webhook
    payload = request.body.read
    sig_header = request.headers["Stripe-Signature"]
    webhook_secret = Rails.application.credentials.dig(:stripe, :webhook_secret)

    begin
      event = Stripe::Webhook.construct_event(payload, sig_header, webhook_secret)
    rescue JSON::ParserError, Stripe::SignatureVerificationError => e
      Rails.logger.warn("[Stripe] Webhook rejected: #{e.message}")
      head :bad_request and return
    end

    begin
      StripeEvent.create!(stripe_event_id: event.id, event_type: event.type, raw_payload: event.to_hash)
    rescue ActiveRecord::RecordInvalid, ActiveRecord::RecordNotUnique
      head :ok and return # Stripe retried a delivery we already processed
    end

    handle_event(event)
    head :ok
  end

  def success
    redirect_to root_path
  end

  def cancel
    redirect_to root_path
  end

  private

  def handle_event(event)
    case event.type
    when "checkout.session.completed"
      handle_checkout_completed(event)
    end
  end

  def handle_checkout_completed(event)
    session = event.data.object
    order = Order.find_by(order_number: session.client_reference_id)
    return if order.nil? || !order.pending_payment?

    order.mark_paid!(stripe_payment_intent_id: session.payment_intent, stripe_raw_event: event.to_hash)
  end
end
```

A few things that aren't obvious on first read:

**`payload = request.body.read` has to be the raw bytes, not `params`.** Stripe signs the exact bytes it sent; anything that round-trips through Rails' JSON parsing first (reformatted, re-serialized) would no longer match the signature. Reading the raw body works here specifically because Rails' own params-parsing middleware already ran earlier in the stack and rewinds the input stream when it's done — so `request.body.read` in the action still gets the full original payload, not an empty stream. Confirmed for real against a running server, not just in a test — see below.

**`StripeEvent.create!` failing is the idempotency mechanism, not an error path.** Stripe *will* retry a webhook delivery if your endpoint doesn't answer fast enough, so the same `event.id` can legitimately arrive more than once. The uniqueness validation on `stripe_event_id` makes the second attempt fail — and that failure is treated as "already handled," not a bug.

**The second real bug, hit writing the retry test**: that rescue clause originally only caught `ActiveRecord::RecordNotUnique` — the exception a raw *database* unique-index violation raises. It never fired. `create!` runs Ruby-level validations first, and a failed `uniqueness: true` check raises `ActiveRecord::RecordInvalid` before the INSERT is even attempted — Rails' own default exception mapping then turned that uncaught error into an HTTP `422`, not the `200` idempotent no-op the code was supposed to return. `RecordNotUnique` only exists as a fallback for the (here, purely hypothetical) case where that validation gets bypassed and the database's own unique index catches it instead. Both exceptions needed rescuing, not just one — an easy assumption to get backwards if you've mostly worked with the DB-level version of this pattern.

**Order lookup uses `client_reference_id`, exactly as planned in episode 1.** `session.client_reference_id` is our own `order_number`, unchanged since we set it building the Checkout Session — this is the whole reason it was worth carrying through in the first place.

## Verified for real — no mocking needed this time

Episode 1 could only verify the *shape* of the call to Stripe with mocks, because creating a real Checkout Session needs live API credentials this project doesn't have yet. Verifying an incoming webhook is different: it's pure local cryptography, with no dependency on Stripe's API being reachable at all. So instead of another mock, this got tested against a real, running `bin/rails server`:

```bash
$ curl -i -X POST http://localhost:3111/stripe/webhook \
    -H "Content-Type: application/json" \
    -H "Stripe-Signature: t=1789677669,v1=0f8c31e3...b5284d6b" \
    --data-binary @webhook_payload.json

HTTP/1.1 200 OK
```

```
Started POST "/stripe/webhook" for ::1
Processing by StripeCallbacksController#webhook
  StripeEvent Create (8.4ms)  INSERT INTO "stripe_events" ...
  Order Update (1.0ms)  UPDATE "orders" SET "status" = 1,
    "stripe_payment_intent_id" = 'pi_live_curl_test', "paid_at" = '2026-09-17 20:41:18' ...
Completed 200 OK in 107ms
```

A payload signed by hand with plain Ruby (`OpenSSL::HMAC.hexdigest`, matching Stripe's own `"#{timestamp}.#{payload}"` scheme) against a genuinely running Puma process, not a request spec double. Then the other two branches, same way: a header with a made-up signature got a real `400`, and replaying the identical signed request a second time got a `200` with no second row in `stripe_events` and no change to `paid_at`.

## Trying it

```bash
git checkout episode-2
bundle install
bin/rails db:migrate
bin/rails test
```

17 tests now (up from 11) — the new ones cover a validly signed `checkout.session.completed` marking the order paid, an unsigned/mis-signed request rejected with the order untouched, a retried delivery of the same event id being a genuine no-op, and an event referencing an unknown order being accepted (so Stripe doesn't keep retrying it) without changing anything.

## What's next

Episode 3 covers the parts still missing before this could handle real traffic: `checkout.session.expired` and `payment_intent.payment_failed`, not just the happy path, plus live credentials and what changes moving off test mode.
