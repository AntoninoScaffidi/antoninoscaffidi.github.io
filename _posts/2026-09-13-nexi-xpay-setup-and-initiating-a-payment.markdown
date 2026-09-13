---
layout: post
title: "Nexi XPay with Rails: Setting Up and Initiating a Payment"
series: "nexi-xpay-with-rails"
episode: 1
lang: en
ref: nexi-xpay-setup-and-initiating-a-payment
permalink: /nexi-xpay-setup-and-initiating-a-payment/
canonical_url: https://antoninoscaffidi.github.io/nexi-xpay-setup-and-initiating-a-payment/
image: /assets/images/nexi-xpay-ep1-banner.png
date: 2026-09-13 07:00:00 +0200
---

This is the first episode of a new series: integrating [Nexi XPay](https://ecommerce.nexi.it/specifiche-tecniche/) — one of the most common card payment gateways for Italian merchants — into a Ruby on Rails app, from scratch, end to end. Everything in this series is verified against a real Nexi sandbox environment, including an actual test payment that went all the way through.

Code is in the [nexi-xpay-with-rails](https://github.com/AntoninoScaffidi/nexi-xpay-with-rails) repo, tagged [`episode-1`](https://github.com/AntoninoScaffidi/nexi-xpay-with-rails/tree/episode-1). This episode gets a customer from "click buy" to Nexi's own payment page, with the price read from the database, never from anything the browser sends.

## Four ways to integrate, and why "Pagamento Semplice"

Nexi's documentation offers several integration modes, and the right one depends entirely on how much you want to own:

| Mode | Card data | Complexity |
|---|---|---|
| **Pagamento Semplice** | Never touches your server (hosted page) | Low — a POST + a signature |
| Pagamento OneClick | Never touches your server, but needs a token from a prior "Semplice" payment | Medium |
| Server-to-Server | **Passes through your server** | High |
| Lightbox / XPay Build | Never touches your server, via iframe/JS SDK | Medium-high |

The deciding factor is **Server-to-Server**: it requires the card number to transit — even briefly — through your own backend, which pulls your app into PCI-DSS compliance scope. That's a real cost (audits, infrastructure requirements) that isn't justified unless you're processing serious volume. **Pagamento Semplice** — redirecting the customer's browser to a page Nexi hosts — means card data never reaches your server at all. Your app only ever sees the final outcome, and even then only a masked version of the card number (`453997******0006`), never the number in clear text. That's the mode this series builds.

## The domain: `Product` and `Order`

Two models, deliberately small:

```bash
bin/rails generate model Product name:string price:decimal purchasable:boolean
bin/rails generate model Order order_number:string product:references status:integer \
  unit_price:decimal total_amount:decimal quantity:integer currency:string \
  guest_first_name:string guest_last_name:string guest_email:string guest_phone:string \
  nexi_transaction_id:string nexi_authorization_code:string nexi_raw_response:jsonb paid_at:datetime
```

`Product#purchasable` is a plain boolean, default `false` — it's the switch that decides whether a product shows a "Buy now" button at all. `Order` carries both a `unit_price` and a `total_amount` rather than just reading `product.price` live: once an order exists, it's a record of what was actually agreed, and a later price change on the product must never rewrite it.

```ruby
# app/models/order.rb
class Order < ApplicationRecord
  belongs_to :product

  enum :status, { pending_payment: 0, paid: 1, payment_failed: 2, cancelled: 3, refunded: 4 }

  before_validation :generate_order_number, on: :create

  validates :order_number, presence: true, uniqueness: true
  validates :unit_price, :total_amount, numericality: { greater_than: 0 }
  validates :guest_first_name, :guest_last_name, presence: true
  validates :guest_email, presence: true, format: { with: VALID_EMAIL_REGEX }

  def to_param
    order_number
  end

  # The only place that reads the price from the database and computes the total.
  def self.build_from_product!(product:, guest_attributes:, quantity: 1)
    raise ArgumentError, "Product is not purchasable" unless product.purchasable?

    unit_price = product.price
    create!(
      product: product, quantity: quantity.to_i, unit_price: unit_price,
      total_amount: unit_price * quantity.to_i,
      **guest_attributes
    )
  end

  private

  def generate_order_number
    self.order_number ||= loop do
      candidate = SecureRandom.alphanumeric(12).upcase
      break candidate unless Order.exists?(order_number: candidate)
    end
  end
end
```

`build_from_product!` is the single chokepoint every order passes through, and it's the reason a manipulated price in a form POST can never actually change what gets charged — `unit_price` comes from `product.price`, full stop, regardless of what a malicious client sends alongside it. `order_number` — 12 random uppercase alphanumeric characters, collision-checked against the database rather than just trusted to be unique by chance — is what appears in public URLs (`to_param` overrides the default), never the sequential database `id`.

## Credentials, and the sandbox-vs-production gotcha

Nexi credentials — an `alias`, a `mac_key`, and the endpoint URL — live in Rails' encrypted credentials, never in `.env` or source:

```bash
bin/rails credentials:edit
```

```yaml
nexi:
  sandbox:
    alias: YOUR_SANDBOX_ALIAS
    mac_key: your_sandbox_mac_key
    pay_url: https://int-ecommerce.nexi.it/ecomm/ecomm/DispatcherServlet
```

**A real gotcha worth knowing before you hit it yourself**: the alias for the sandbox (`int-ecommerce.nexi.it`) is *not* the same alias you'll eventually get for production. It's a separate test terminal, found in the Nexi backoffice under **Area test → Pagamento Semplice/OneClick**, and it's easy to grab the wrong one — the production terminal's alias, with your real business's name and address — and have every sandbox request rejected with `"Alias non valido per l'operazione richiesta"`. The same backoffice page hands you the test card numbers too (episode 3 covers using them for real).

## `Nexi::Gateway`: the one place that knows the protocol

Everything Nexi-specific lives in a single service object — building the request, and later (episode 2) verifying what comes back:

```ruby
# app/services/nexi/gateway.rb
module Nexi
  class Gateway
    class MissingCredentials < StandardError; end

    def initialize(env: Rails.env.production? ? :production : :sandbox)
      @credentials = Rails.application.credentials.dig(:nexi, env)
      raise MissingCredentials, "Missing Nexi credentials for environment #{env}" if @credentials.blank?
    end

    def payment_form_url
      @credentials[:pay_url]
    end

    def build_payment_params(order, return_url:, cancel_url:, notify_url:)
      cod_trans = order.order_number
      divisa = order.currency
      importo = amount_in_cents(order.total_amount)

      {
        "alias" => alias_code, "importo" => importo, "divisa" => divisa, "codTrans" => cod_trans,
        "url" => return_url, "url_back" => cancel_url, "urlpost" => notify_url,
        "mail" => order.guest_email, "nome" => order.guest_first_name, "cognome" => order.guest_last_name,
        "mac" => compute_request_mac(cod_trans: cod_trans, divisa: divisa, importo: importo)
      }.compact
    end

    private

    def alias_code = @credentials[:alias]
    def mac_key = @credentials[:mac_key]
    def amount_in_cents(decimal_amount) = (decimal_amount * 100).round.to_s

    def compute_request_mac(cod_trans:, divisa:, importo:)
      Digest::SHA1.hexdigest("codTrans=#{cod_trans}divisa=#{divisa}importo=#{importo}#{mac_key}")
    end
  end
end
```

Two details worth being deliberate about:

- **The `raise` on missing credentials isn't defensive for its own sake.** Without it, a deploy that forgot to set `nexi.production` would silently build a request with `nil` values and send a customer to a broken payment page. With it, the app fails loudly on the very first checkout attempt instead.
- **`amount_in_cents` matters because Nexi's `importo` field is an integer with no decimal separator** — `€42.50` has to become the string `"4250"`, not `"42.50"`. Getting this wrong doesn't raise an error; it just charges the wrong amount.

`compute_request_mac` is the **MAC** (Message Authentication Code) — a SHA1 hash of a handful of fields concatenated with no separators, plus the secret key, computed exactly the way Nexi's own servers will compute it independently to confirm the request wasn't tampered with in transit. This half of the gateway builds and signs the *outgoing* request; episode 2 adds the other half, which verifies the *incoming* one.

## Routes and `OrdersController`

```ruby
# config/routes.rb
resources :products, only: [ :index, :show ]
resources :orders, only: [ :new, :create ], param: :order_number

get "nexi/return", to: "nexi_callbacks#return", as: :nexi_return
post "nexi/notify", to: "nexi_callbacks#notify", as: :nexi_notify
```

`nexi_return` and `nexi_notify` need to exist as routes from day one — `OrdersController#create` generates URLs for both — even though the controller behind them, `NexiCallbacksController`, is still a stub in this episode (`#notify` just returns `head :ok`, `#return` just redirects home). Nexi needs somewhere real to send the browser and the server notification; what those endpoints actually *do* with what arrives is next episode's problem.

```ruby
# app/controllers/orders_controller.rb
class OrdersController < ApplicationController
  before_action :set_product, only: [ :new, :create ]

  def new
    redirect_to(product_path(@product)) and return unless @product.purchasable?
    @order = Order.new
  end

  def create
    redirect_to(product_path(@product)) and return unless @product.purchasable?

    @order = Order.build_from_product!(
      product: @product, guest_attributes: order_params.to_h,
      quantity: order_params[:quantity].presence || 1
    )

    gateway = Nexi::Gateway.new
    @payment_form_url = gateway.payment_form_url
    @payment_params = gateway.build_payment_params(
      @order, return_url: nexi_return_url, cancel_url: nexi_return_url, notify_url: nexi_notify_url
    )
  rescue ActiveRecord::RecordInvalid => e
    @order = e.record
    render :new, status: :unprocessable_entity
  end

  private

  def set_product
    @product = Product.find(params[:product_id])
  end

  def order_params
    params.permit(:quantity, :guest_first_name, :guest_last_name, :guest_email, :guest_phone)
  end
end
```

`purchasable?` gets checked in both `#new` and `#create`, not just once — between a customer opening the form and submitting it, that flag could get flipped off. The second check closes that window. And notice what `order_params` deliberately leaves out: no `:total_amount`, no `:unit_price`. It's not filtered out as a special security rule — it's simply never in the permitted list, so `params.permit` silently drops it before it ever reaches `build_from_product!`. A malicious `total_amount: "1"` in the POST body has nowhere to go.

## Two view gotchas that cost real debugging time

The checkout form:

```erb
<%# app/views/orders/new.html.erb %>
<%= form_with url: orders_path, method: :post, data: { turbo: false } do |f| %>
  <%= f.hidden_field :product_id, value: @product.id %>
  <%= f.text_field :guest_first_name, required: true %>
  <%# ... %>
  <%= f.submit "Go to payment" %>
<% end %>
```

**Gotcha #1 — Turbo Drive silently swallows the submit.** Without `data: { turbo: false }`, Rails logs the request as `Processing ... as TURBO_STREAM`, and nothing happens: no error, no redirect, just a page that looks like the button did nothing. The fix is one line, but finding it means actually reading `log/development.log` while clicking submit — the `as TURBO_STREAM` line is the tell.

The auto-submitting handoff to Nexi:

```erb
<%# app/views/orders/create.html.erb %>
<%# form_with/form_tag always force accept-charset="UTF-8": Nexi requires
    ISO-8859-1, so the <form> tag has to be written by hand here. %>
<form id="nexi-payment-form" method="post" action="<%= @payment_form_url %>" accept-charset="ISO-8859-1">
  <% @payment_params.each do |name, value| %>
    <%= hidden_field_tag name, value, id: nil %>
  <% end %>
</form>
<script>document.getElementById("nexi-payment-form").submit();</script>
```

**Gotcha #2 — Rails hardcodes `accept-charset="UTF-8"` on every form.** Nexi needs `ISO-8859-1` so accented names come through correctly, but `form_with`/`form_tag` set `accept-charset` *unconditionally* inside `action_view/helpers/form_tag_helper.rb` — any value you pass in `html: {...}` gets overwritten right after. The only way around it is what's above: skip the helper for the `<form>` tag itself, and use `hidden_field_tag` (which doesn't go through the same wrapper) for the fields.

## Trying it

```bash
git clone https://github.com/AntoninoScaffidi/nexi-xpay-with-rails.git
cd nexi-xpay-with-rails
git checkout episode-1
bundle install
bin/rails db:prepare
bin/rails db:seed
bin/rails test
```

The test suite (11 tests at this point) covers the two things most worth trusting before a single euro is involved: that `build_from_product!` always uses the database price regardless of what the client sends, and that `Gateway#build_payment_params` converts the amount to cents correctly. Run the server, visit the seeded product, and the checkout form hands off to Nexi's real sandbox payment page — the customer just can't finish a payment yet, because nothing on this end is listening for the answer.

## What's next

[Episode 2]({% post_url 2026-09-13-nexi-xpay-handling-the-outcome %}) covers the two channels Nexi uses to tell you what happened — a browser redirect (UX only) and a server-to-server webhook (the actual source of truth) — MAC verification on the way back in, and the real bugs hit building it.
