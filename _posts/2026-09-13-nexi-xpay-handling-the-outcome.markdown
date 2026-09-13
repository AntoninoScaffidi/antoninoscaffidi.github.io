---
layout: post
title: "Nexi XPay with Rails: Handling the Outcome"
series: "nexi-xpay-with-rails"
episode: 2
lang: en
ref: nexi-xpay-handling-the-outcome
permalink: /nexi-xpay-handling-the-outcome/
canonical_url: https://antoninoscaffidi.github.io/nexi-xpay-handling-the-outcome/
image: /assets/images/nexi-xpay-ep2-banner.png
date: 2026-09-13 07:30:00 +0200
---

[Episode 1]({% post_url 2026-09-13-nexi-xpay-setup-and-initiating-a-payment %}) got a customer from "click buy" to Nexi's hosted payment page, with `NexiCallbacksController` left as a stub — two endpoints that existed only so Nexi had somewhere to send the browser and the server notification. This episode fills them in for real: verifying what comes back, marking orders paid, and surviving the ways a webhook can go wrong in practice.

Code is tagged [`episode-2`](https://github.com/AntoninoScaffidi/nexi-xpay-with-rails/tree/episode-2) in the [nexi-xpay-with-rails](https://github.com/AntoninoScaffidi/nexi-xpay-with-rails) repo.

## Two channels, and only one of them is trustworthy

Once a customer finishes on Nexi's payment page, Nexi talks back to your app through two entirely independent channels:

- **A browser redirect** (`url`/`url_back`) — the customer's browser gets sent back to your site. This is UX only: if the customer closes the tab the instant payment finishes, this redirect simply never happens.
- **A server-to-server notification** (`urlpost`) — Nexi's own servers POST the outcome directly to your backend, with no dependency on the customer's browser still being open. This is the only channel that's actually reliable, and the only one this app trusts to change an order's state.

That's why `url` and `url_back` both point at the *same* route, `nexi_return` — the action reads `esito` (`OK`/`KO`/`ANNULLO`/`ERRORE`) purely to decide what page to show the customer, but it **never** writes order state. Only `#notify` does that.

## `OrderPaymentNotification`: an audit log, not a domain model

```bash
bin/rails generate model OrderPaymentNotification order:references \
  raw_params:jsonb mac_verified:boolean processed:boolean outcome:string
```

```ruby
# app/models/order_payment_notification.rb
class OrderPaymentNotification < ApplicationRecord
  belongs_to :order
end
```

Deliberately minimal — no logic at all. It exists to record *every* notification Nexi sends, including ones with an invalid signature, before any decision gets made about what to do with it. The interpretation lives in the controller; this model is a record of events, not a domain entity with its own behavior.

## `Nexi::Gateway#verify_mac`: the other half of the class

Episode 1 showed `Nexi::Gateway` building and signing the outgoing request. This episode adds the method that checks the signature on the way back in:

```ruby
# app/services/nexi/gateway.rb (additions)
def verify_mac(params)
  received = params["mac"] || params[:mac]
  return false if received.blank?

  ActiveSupport::SecurityUtils.secure_compare(compute_response_mac(params), received.to_s)
end

private

def compute_response_mac(params)
  value = ->(key) { (params[key] || params[key.to_sym]).to_s }
  string_to_sign =
    "codTrans=#{value.call('codTrans')}esito=#{value.call('esito')}importo=#{value.call('importo')}" \
    "divisa=#{value.call('divisa')}data=#{value.call('data')}orario=#{value.call('orario')}" \
    "codAut=#{value.call('codAut')}#{mac_key}"
  Digest::SHA1.hexdigest(string_to_sign)
end
```

The outcome message signs more fields than the request did (`esito`, `data`, `orario`, `codAut` are all new), but the mechanism is identical: concatenate, hash, compare. Two details worth being precise about:

- **`ActiveSupport::SecurityUtils.secure_compare` instead of `==`.** A plain string comparison returns as soon as it finds the first mismatched character, which means comparison time leaks information about *how many characters were correct* — a timing attack. `secure_compare` always takes the same time regardless of where the strings first diverge.
- **`verify_mac` returns `false` even when `mac` is entirely absent from the params**, rather than raising or skipping the check. There must never be a code path where "no signature at all" is treated the same as "valid signature."

## `NexiCallbacksController`, for real this time

```ruby
# app/controllers/nexi_callbacks_controller.rb
class NexiCallbacksController < ActionController::Base
  skip_before_action :verify_authenticity_token, only: [ :notify ]

  def notify
    order = Order.find_by(order_number: params[:codTrans])
    if order.nil?
      Rails.logger.error("[Nexi] Notification for unknown codTrans: #{params[:codTrans]}")
      head :ok and return
    end

    notification = order.order_payment_notifications.create!(raw_params: raw_notification_params)

    gateway = Nexi::Gateway.new
    mac_valid = gateway.verify_mac(raw_notification_params)
    notification.update!(mac_verified: mac_valid)

    unless mac_valid
      notification.update!(processed: true, outcome: "invalid_mac")
      head :ok and return
    end

    case params[:esito]
    when "OK"
      if order.pending_payment?
        order.mark_paid!(
          nexi_transaction_id: params[:codTrans], nexi_authorization_code: params[:codAut],
          nexi_raw_response: raw_notification_params
        )
        notification.update!(processed: true, outcome: "paid")
      else
        notification.update!(processed: true, outcome: "duplicate") # idempotency
      end
    else
      order.update!(status: :payment_failed) if order.pending_payment?
      notification.update!(processed: true, outcome: params[:esito].to_s.downcase.presence || "unknown")
    end

    head :ok
  end

  def return
    order = Order.find_by(order_number: params[:codTrans])
    order.nil? ? redirect_to(root_path) : redirect_to(order_path(order))
  end

  private

  def raw_notification_params
    params.to_unsafe_h.except("controller", "action")
  end
end
```

A few things here are load-bearing in ways that aren't obvious on first read:

**`skip_before_action :verify_authenticity_token` is scoped to `:notify` only.** This is a plain POST from an external server, with no session cookie and no CSRF token — checking for one would reject every legitimate notification. `#return` stays protected, though it's moot there anyway: it's a `GET`, and Rails only enforces CSRF on state-changing verbs.

**The order of operations inside `#notify` is not arbitrary.** The raw notification gets logged *before* the MAC is checked, not after. A notification with an invalid signature still gets written to `order_payment_notifications` — it's exactly the record you'd want later ("Nexi really did contact us about this `codTrans`, but with a bad signature — why?"). Validating first and only logging what passes would throw away precisely the cases most worth investigating.

**`mark_paid!` is a no-op on an already-paid order** (`return if paid?`, from episode 1's `Order` model), and the `else` branch above logs that case as `"duplicate"` rather than re-processing it. This matters because Nexi *will* retry a notification if your endpoint doesn't answer `200` fast enough, or at all — the same `codTrans=OK` message can legitimately arrive more than once, and nothing about that should trigger a second confirmation email or double-count anything.

**`raw_notification_params` is not mass assignment onto an `ActiveRecord` model** — it only ever lands in a `jsonb` audit column, never in `update`/`create` on `Order` itself, so there's no path from "whatever Nexi happened to send" to silently overwriting fields it was never meant to touch.

## Order status, and why it usually says "pending"

```erb
<%# app/views/orders/show.html.erb %>
<% if @order.paid? %>
  <%# green check, "Payment confirmed" %>
<% elsif @order.pending_payment? %>
  <%# "Payment in progress", invites a reload %>
<% else %>
  <%# payment_failed / cancelled / refunded %>
<% end %>
```

In practice, `pending_payment?` is the branch a customer sees most often, even after a successful payment. The browser redirect (`#return`) frequently lands here *before* the asynchronous `#notify` webhook has actually marked the order paid — the two channels have no ordering guarantee relative to each other. That's why the copy explicitly invites a reload rather than promising a live update; there's no polling or websocket in this episode, on purpose, to keep the surface small.

## A real bug: a brand-new `json` release breaking `ActiveSupport::JSON.decode`

Not a logic bug — an environment one, hit the moment this episode's code first tried to write to a `jsonb` column:

```
ArgumentError: wrong number of arguments (given 2, expected 1)
  .../json-3.0.2/lib/json/common.rb:296:in 'parse'
  .../activesupport-8.1.3.1/lib/active_support/json/decoding.rb:25:in 'ActiveSupport::JSON.decode'
  .../activerecord-8.1.3.1/lib/active_record/type/json.rb:17:in 'ActiveRecord::Type::Json#deserialize'
```

`json` `3.0.2` — freshly published on RubyGems — changed `JSON.parse`'s signature in a way `ActiveSupport::JSON.decode` doesn't expect on this Rails version, and `bundle install` had resolved straight into it with no explicit pin in the `Gemfile`. The fix is one line:

```ruby
# Gemfile
gem "json", "< 3.0"
```

followed by `bundle update json`. Worth remembering as a general lesson, not just a Nexi one: an unpinned transitive dependency can break a working app on nothing more than `bundle install` picking up a gem release that came out an hour ago. If a `jsonb`/JSON-touching code path suddenly throws an `ArgumentError` about argument counts with no code change on your side, check what actually got resolved before assuming the bug is yours.

## Trying it

```bash
git checkout episode-2
bundle install
bin/rails db:migrate
bin/rails test
```

19 tests now (up from 11) — the new ones cover `verify_mac` accepting a correctly-signed response and rejecting a tampered or missing one, a valid `OK` notification marking the order paid, the same notification arriving twice not double-processing, an invalid MAC leaving the order untouched, and `#return` never writing state under any outcome.

## What's next

Episode 3 runs the whole thing against Nexi's real sandbox end to end — a public tunnel with ngrok, an actual test-card payment, and the notification arriving on its own, unprompted — plus what's still missing before flipping this to production.
