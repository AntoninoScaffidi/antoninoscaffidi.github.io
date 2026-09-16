---
layout: post
title: "Stripe with Rails: Setting Up and Initiating a Payment"
series: "stripe-with-rails"
episode: 1
lang: en
ref: stripe-setup-and-initiating-a-payment
permalink: /stripe-setup-and-initiating-a-payment/
canonical_url: https://antoninoscaffidi.github.io/stripe-setup-and-initiating-a-payment/
image: /assets/images/stripe-ep1-banner.png
date: 2026-09-16 21:00:00 +0200
---

This is the first episode of a new series: integrating [Stripe](https://stripe.com) — probably the most developer-friendly payment gateway there is — into a Ruby on Rails app, from scratch. If you've just come from [Nexi XPay with Rails]({% post_url 2026-09-13-nexi-xpay-setup-and-initiating-a-payment %}), this series deliberately reuses the same `Product`/`Order` domain, so what's actually different between an in-house European gateway and a modern, developer-first American one stands out clearly, line by line — not buried under two different apps you have to mentally diff yourself.

Code is in the [stripe-with-rails](https://github.com/AntoninoScaffidi/stripe-with-rails) repo, tagged [`episode-1`](https://github.com/AntoninoScaffidi/stripe-with-rails/tree/episode-1).

## Stripe Checkout: the hosted-page equivalent

Like Nexi's "Pagamento Semplice," Stripe has a hosted-page mode called **Checkout**: you never collect card details yourself, you redirect the customer to a page Stripe hosts, and Stripe redirects them back once they're done. Same PCI-DSS-avoidance logic as before — card data never touches your server — but the *mechanism* for getting there is different in a way worth being precise about.

Nexi's "Pagamento Semplice" is a **form POST with hidden fields**, signed with a hand-rolled MAC, submitted by the *browser* directly to Nexi's `DispatcherServlet`. Your Rails app never talks to Nexi's servers directly to start a payment — it just builds a signed form and hands it to the customer's browser.

Stripe Checkout works the other way around: your Rails app makes a **server-to-server API call** to Stripe (`Stripe::Checkout::Session.create`), Stripe creates a session and hands back a URL, and *then* you redirect the customer's browser to that URL. Your server talks to Stripe before the customer ever sees a payment page — which is also why there's no MAC to hand-compute: the official `stripe` gem authenticates that API call for you, using your secret key, the same way any authenticated HTTP API call works.

## API keys: test mode needs zero setup

Get yours from the [Stripe Dashboard](https://dashboard.stripe.com/test/apikeys) — test mode is available the instant you create a free account, no business verification, no waiting, unlike Nexi's sandbox which needs credentials issued through a separate backoffice flow. Two keys matter here:

- **Secret key** (`sk_test_...`) — used server-side only, to authenticate API calls like creating a Checkout Session. Never exposed to the browser.
- **Publishable key** (`pk_test_...`) — safe to expose client-side, used if you ever embed Stripe.js directly (not needed for this episode, since Checkout is a pure redirect).

Both go in Rails' encrypted credentials, never `.env`:

```bash
bin/rails credentials:edit
```

```yaml
stripe:
  secret_key: sk_test_...
  publishable_key: pk_test_...
  webhook_secret: whsec_...
```

(`webhook_secret` isn't used until episode 2 — added now so there's one place to look.)

```ruby
# config/initializers/stripe.rb
Stripe.api_key = Rails.application.credentials.dig(:stripe, :secret_key)
```

One line. `Stripe.api_key=` is a global, process-wide setter the `stripe` gem reads before every API call — there's no `Stripe::Client.new(env:)` juggling sandbox-vs-production namespaces the way `Nexi::Gateway` needed; Stripe test/live mode is determined entirely by *which* secret key you set (`sk_test_...` vs `sk_live_...`), not by an explicit environment flag in your code.

## The domain: same shape, different fields

```bash
bin/rails generate model Product name:string price:decimal purchasable:boolean
bin/rails generate model Order order_number:string product:references status:integer \
  unit_price:decimal total_amount:decimal quantity:integer currency:string \
  guest_first_name:string guest_last_name:string guest_email:string guest_phone:string \
  stripe_checkout_session_id:string stripe_payment_intent_id:string stripe_raw_event:jsonb paid_at:datetime
```

Same shape as the Nexi series' `Order` — `build_from_product!` reading the price from the database, `generate_order_number` for a public, non-sequential identifier, the same guest-checkout validations. One real difference worth flagging: `currency` defaults to `"eur"` **lowercase**. Nexi wanted `"EUR"` uppercase; Stripe's API rejects an uppercase currency code outright. Small, easy to get wrong once and never think about again — which is exactly the kind of thing worth writing down.

```ruby
# app/models/order.rb (the Stripe-specific addition)
# Stripe expects amounts as an integer number of the currency's smallest unit
# (cents, for EUR/USD) — never a decimal.
def unit_price_in_cents
  (unit_price * 100).round
end
```

## `StripeCheckoutService`: building the session

```ruby
# app/services/stripe_checkout_service.rb
class StripeCheckoutService
  def self.create_session(order:, success_url:, cancel_url:)
    Stripe::Checkout::Session.create(
      mode: "payment",
      line_items: [
        {
          price_data: {
            currency: order.currency,
            product_data: { name: order.product.name },
            unit_amount: order.unit_price_in_cents
          },
          quantity: order.quantity
        }
      ],
      customer_email: order.guest_email,
      client_reference_id: order.order_number,
      success_url: success_url,
      cancel_url: cancel_url
    )
  end
end
```

A few things worth being precise about:

**`line_items` wants a per-unit price plus a quantity, not a pre-multiplied total.** `unit_amount` is `unit_price_in_cents` — not `total_amount` — because Stripe itself multiplies price × quantity to compute the charge. Passing the total here would silently double-charge on any order with `quantity > 1`.

**`client_reference_id` is Stripe's answer to Nexi's `codTrans`** — your own reference, attached to the session, that comes back untouched in every webhook event about it. This is how episode 2 will find the right `Order` row when a webhook arrives: not by any Stripe-generated ID minted *before* your order existed, but by the identifier you handed Stripe yourself.

**There's no explicit signature to compute here**, unlike `Nexi::Gateway#build_payment_params`. The entire request is authenticated by the `Authorization: Bearer sk_test_...` header the `stripe` gem attaches automatically from `Stripe.api_key`. That's the trade-off of a hosted SDK versus hand-rolling a protocol: less code, less to get subtly wrong, but also less visibility into what's actually happening on the wire unless you go looking for it.

## What a webhook actually is, and why Checkout needs one

This matters enough to slow down for, especially if "webhook" is still a word you nod along to rather than one you could explain from first principles.

An ordinary API call is something **you** initiate: your Rails app decides it wants something from Stripe, sends a request, and gets a response back — like `Stripe::Checkout::Session.create` above. It works because *you* know when you need information, and you can just ask for it.

A **webhook is the opposite direction.** After the customer finishes paying on Stripe's hosted page, *Stripe* is the one that finds out first — your Rails server has no way to know, from where it's sitting, that a specific customer on a specific browser just successfully entered a card number three seconds ago. There's nothing to poll: you could ask Stripe "did this session get paid yet?" once a second forever, but that's wasteful, slow to notice, and doesn't scale. So instead, you give Stripe a URL — `POST /stripe/webhook` in this app — and Stripe **calls you**, the moment something happens, with a JSON payload describing the event. Your server doesn't ask; Stripe tells.

That inversion creates a problem an ordinary API call never has: **anyone on the internet can send a POST request to a public URL.** When your Rails app calls Stripe, the response is trustworthy because you initiated the connection to a host you already know is really Stripe's. When Stripe calls *you*, the roles reverse — your `/stripe/webhook` endpoint has to independently verify that a given incoming request genuinely came from Stripe and not from someone who found the URL and is pretending. That's exactly the role Nexi's hand-rolled MAC played on the way back in, and it's exactly what Stripe's own signature verification (`Stripe::Webhook.construct_event`, covered in full next episode) does here — same underlying problem, two very differently-shaped solutions worth comparing directly once both are on the page.

For now, this episode's `StripeCallbacksController#webhook` is a stub — the endpoint exists, so a route is there for Stripe to call, but it doesn't yet verify anything or act on what arrives:

```ruby
# app/controllers/stripe_callbacks_controller.rb
class StripeCallbacksController < ActionController::Base
  skip_before_action :verify_authenticity_token, only: [ :webhook ]

  def webhook
    head :ok
  end

  def success
    redirect_to root_path
  end

  def cancel
    redirect_to root_path
  end
end
```

`success` and `cancel` are the *other* channel — the browser redirect, UX-only, exactly like Nexi's `#return`. Stripe sends the customer's browser back to `success_url`/`cancel_url` regardless of whether the webhook has already arrived, already been processed, or hasn't shown up yet at all. Same two-channel shape as Nexi, same reason: the redirect is convenient but unreliable (a closed tab means it never fires), the webhook is the only channel actually trustworthy enough to mark an order paid.

## Routes and the controller

```ruby
# config/routes.rb
resources :products, only: [ :index, :show ]
resources :orders, only: [ :new, :create ], param: :order_number

get "stripe/success", to: "stripe_callbacks#success", as: :stripe_success
get "stripe/cancel", to: "stripe_callbacks#cancel", as: :stripe_cancel
post "stripe/webhook", to: "stripe_callbacks#webhook", as: :stripe_webhook
```

```ruby
# app/controllers/orders_controller.rb
def create
  redirect_to(product_path(@product)) and return unless @product.purchasable?

  @order = Order.build_from_product!(
    product: @product, guest_attributes: order_params.to_h,
    quantity: order_params[:quantity].presence || 1
  )

  session = StripeCheckoutService.create_session(
    order: @order,
    success_url: stripe_success_url(order_number: @order.order_number),
    cancel_url: stripe_cancel_url(order_number: @order.order_number)
  )
  @order.update!(stripe_checkout_session_id: session.id)

  redirect_to session.url, allow_other_host: true
rescue ActiveRecord::RecordInvalid => e
  @order = e.record
  render :new, status: :unprocessable_entity
end
```

`allow_other_host: true` is required, not optional — Rails 7+ raises `ActionController::Redirecting::UnsafeRedirectError` on a `redirect_to` pointed at a host your app doesn't recognize as its own, as a defense against open-redirect vulnerabilities. `session.url` legitimately points at `checkout.stripe.com`, so the safety check has to be explicitly overridden for this one, deliberate case — not disabled globally.

## A bug I hit tonight, twice: a brand-new gem breaking on first boot

Before any of the above could even render, the very first page load threw:

```
ArgumentError (wrong number of arguments (given 2, expected 1))
app/views/layouts/application.html.erb:10 — <%= csp_meta_tag %>
```

Same root cause as [a bug from the Nexi series]({% post_url 2026-09-13-nexi-xpay-handling-the-outcome %}): `json` `3.0.2`, freshly published, breaks `ActiveSupport::JSON.decode`'s call into `JSON.parse` on this Rails version. Last time it only surfaced once `jsonb` columns actually got written to, in episode 2 — this time it hit immediately, from `csp_meta_tag` itself, before a single line of this episode's own code even ran. Same fix:

```ruby
# Gemfile
gem "json", "< 3.0"
```

And a second, unrelated one, found while writing this episode's tests: `Stripe::Checkout::Session.create` needs to be stubbed to test `StripeCheckoutService` without a live API key, using Minitest's built-in `Object#stub` — except it wasn't built in. **Minitest `6.0`, also just published, removed `Minitest::Mock` from the core gem entirely** ("moving minitest/mock.rb to its own gem," per its own changelog). Same shape of problem as the `json` one — a brand-new major release resolved by `bundle install` with no pin in place — same fix:

```ruby
# Gemfile
gem "minitest", "~> 5.25"
```

Two unrelated gems, two unrelated breaking changes, both freshly released, both hit on the very first `bundle install` of a brand-new app, same night. Worth internalizing as a real pattern, not a coincidence to shrug off: an unpinned `Gemfile` is a bet that whatever version `bundle install` happens to resolve *today* will keep working — and on any given day, that bet can just lose.

## A subtler bug: Turbo Drive and a cross-origin redirect

This one took real digging to track down, and it's more interesting than "add `data: turbo: false` again" — the *reason* it's needed here is genuinely different from the Nexi series' Turbo gotcha.

Submitting the checkout form did nothing. Not an error — nothing. No navigation, no console error, the button just... didn't. But the server logs told a different story:

```
Started POST "/orders" for ::1
Processing by OrdersController#create as TURBO_STREAM
...
Redirected to https://checkout.stripe.com/...
Completed 302 Found in 277ms
```

The server did its job. It built the order, created the Stripe session, and issued a real `302` to Stripe's real hosted checkout page. The browser just never went there. To confirm this wasn't specific to Stripe's URL somehow, I temporarily swapped the redirect target for a plain `https://example.com` — same result, dead stop, page never moved.

Here's the actual mechanism: Turbo Drive intercepts ordinary form submissions and replays them as a `fetch()` request instead of a full browser navigation — that's how it avoids a full page reload for same-site form posts. When that `fetch()` response is a redirect, the browser's networking layer follows it transparently *at the HTTP level* — but the response `fetch()` ultimately resolves with is the body from the *final* destination (`checkout.stripe.com`, or `example.com` in the isolated test), addressed to a request Turbo Drive itself initiated, not a real top-level page navigation. Turbo then tries to process that response as if it were more content from *this* app — and since it's neither valid Turbo Stream content nor anything Turbo recognizes, it just quietly gives up. The redirect happened, technically, entirely inside a `fetch()` call the address bar never found out about.

The fix is identical in shape to the Nexi series' Turbo gotcha, arrived at for a different reason:

```erb
<%= form_with url: orders_path, method: :post, data: { turbo: false } do |f| %>
```

`data: { turbo: false }` makes this one form submit the old-fashioned way — a real, full browser navigation, not a `fetch()` Turbo Drive intercepts. A real navigation follows a cross-origin redirect exactly the way you'd expect: the address bar updates, and the customer actually lands on Stripe's page. Confirmed by temporarily redirecting to `example.com` and watching the browser actually arrive there once `turbo: false` was in place — before that, same form, same server-side redirect, the tab simply never moved.

The general lesson, worth remembering independent of Stripe or Nexi specifically: **any form whose action ends in a redirect to a different origin needs `data: { turbo: false }`.** Same-origin redirects work fine under Turbo Drive without it — it's specifically the cross-origin case where a `fetch()`-based redirect and a real browser navigation stop being equivalent.

## Trying it

```bash
git clone https://github.com/AntoninoScaffidi/stripe-with-rails.git
cd stripe-with-rails
git checkout episode-1
bundle install
bin/rails db:prepare
bin/rails db:seed
bin/rails test
```

11 tests, and none of them make a real network call — `StripeCheckoutServiceTest` and `OrdersControllerTest` both stub `Stripe::Checkout::Session.create` with `Minitest::Mock`, checking that the params sent to Stripe are correct (price in cents, not decimal; `client_reference_id` set to our own `order_number`) without needing a live API key just to run the suite. Add your own test keys and the checkout form genuinely redirects to a real, working Stripe-hosted payment page — verified tonight, including tracking down the Turbo Drive issue above by watching it fail against a real redirect target before fixing it.

## What's next

Episode 2 covers the other side: Stripe's event-based webhooks (not a single `esito` field the way Nexi has one, but named events like `checkout.session.completed`), signature verification via the official SDK's `Stripe::Webhook.construct_event` — and a direct, line-by-line comparison against the hand-rolled MAC verification the Nexi series built from scratch.
