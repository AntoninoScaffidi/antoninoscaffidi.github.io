---
layout: post
title: "VicinoTe: Services and Categories"
series: "vicinote"
episode: 3
lang: en
ref: vicinote-services-and-categories
permalink: /vicinote-services-and-categories/
canonical_url: https://antoninoscaffidi.github.io/vicinote-services-and-categories/
date: 2026-08-20 07:00:00 +0200
image: /assets/images/vicinote-banner.png
---

[Episode 2]({% post_url 2026-08-13-vicinote-authentication-with-rails-8 %}) got accounts working end to end — sign up, sign in, sign out, password reset — and left it there deliberately: nothing touched `Service` or `Booking` yet. This episode is where the marketplace actually starts being a marketplace: a signed-in user can list something they offer, and anyone can browse what's listed.

It's also where the `has_many :services` association from [episode 1]({% post_url 2026-08-08-vicinote-project-setup-and-domain %})'s domain sketch finally gets written into the `User` model. It's been sitting in a blog post as a design decision for two episodes; today it becomes real code.

Code is tagged [`episode-3`](https://github.com/AntoninoScaffidi/vicinote-tutorial/tree/episode-3) in the [vicinote-tutorial](https://github.com/AntoninoScaffidi/vicinote-tutorial) repo.

## Generating the two models

```bash
bin/rails generate model Category "name:string:uniq"
bin/rails generate model Service title:string description:text price_cents:integer user:references category:references
```

`user:references` and `category:references` do two things at once: add a `belongs_to` to the generated model, and add the foreign-key column plus index to the migration. That's why `Service`'s generated model already has both associations written in — the generator inferred them from the reference fields, not from anything we typed by hand.

## A decision worth slowing down on: price_cents, not price

The generator wrote `price_cents:integer`, not `price:decimal`. That's deliberate, and it's the same category of decision episode 1 made about `Booking` storing its own price rather than reading `service.price` live: money handled as a float or a naive decimal eventually produces a rounding error that shows up as a few cents off on an invoice, at the worst possible moment. Storing the amount as an integer number of cents sidesteps the whole class of bug — there's no fractional part to round.

The cost is that nobody wants to type "4250" into a form and mean $42.50. So `Service` gets a small virtual accessor that speaks dollars on the way in and out, while `price_cents` stays what's actually validated and stored:

```ruby
# app/models/service.rb
class Service < ApplicationRecord
  belongs_to :user
  belongs_to :category

  validates :title, presence: true
  validates :description, presence: true
  validates :price_cents, presence: true, numericality: { greater_than: 0, only_integer: true }

  # price_cents is what's stored and compared (no float rounding surprises),
  # but nobody wants to type cents into a form. This speaks dollars on the
  # way in and out, so the form field can just be "price".
  def price
    price_cents && price_cents / 100.0
  end

  def price=(value)
    self.price_cents = value.present? ? (value.to_f * 100).round : nil
  end
end
```

`price=` is a normal Ruby setter, not an ActiveRecord attribute — Rails just calls it like any other method when a form submits `service[price]`, the same mechanism that lets `User.new(email_address: "...")` work at all. Because it runs before validation, `price_cents` is already populated by the time `numericality` checks it. The form field, later in this post, is just `form.text_field :price` — the view never knows cents exist.

## Categories are seeded, not created

A marketplace where anyone can invent a new category ends up with fifty categories that mean the same thing, spelled five different ways, and browsing-by-category stops being useful. VicinoTe curates a fixed list instead:

```ruby
# db/seeds.rb
[
  "Home Repair",
  "Tutoring",
  "Cleaning",
  "Gardening",
  "Pet Care",
  "Moving Help",
  "Tech Support",
  "Beauty & Wellness"
].each do |name|
  Category.find_or_create_by!(name: name)
end
```

```bash
bin/rails db:migrate
bin/rails db:seed
```

`find_or_create_by!` makes this safe to run more than once — re-running `db:seed` after adding a ninth category later won't duplicate the first eight. Nothing in this episode builds a UI for managing categories; for now they're something the app ships with, not something a `Service` form can invent.

## Writing the association episode 1 only sketched

Episode 1's domain design showed this code as an illustration of the "role emerges from the association" decision. It was never actually in the `User` model — episode 2 was about authentication and didn't touch it. It goes in now:

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password
  has_many :sessions, dependent: :destroy
  has_many :services, dependent: :destroy

  normalizes :email_address, with: ->(e) { e.strip.downcase }
end
```

`dependent: :destroy` matches what `sessions` already does — if a user's account is ever deleted, their listings shouldn't linger as orphaned rows pointing at a `user_id` that no longer exists.

## The controller: public index, protected everything else

```ruby
# app/controllers/services_controller.rb
class ServicesController < ApplicationController
  allow_unauthenticated_access only: :index

  def index
    @services = Service.includes(:category, :user).order(created_at: :desc)
  end

  def new
    @service = Current.user.services.new
  end

  def create
    @service = Current.user.services.new(service_params)

    if @service.save
      redirect_to services_path, notice: "Your service is live."
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  def service_params
    params.require(:service).permit(:title, :description, :price, :category_id)
  end
end
```

Episode 2's `Authentication` concern requires a session on every action by default, which is exactly right for `new` and `create` — nobody should be able to list a service anonymously — but wrong for `index`: browsing the marketplace has to work for someone who hasn't signed up yet, or there's no reason to sign up in the first place. `allow_unauthenticated_access only: :index` opts that one action back out, the same mechanism episode 2 used to make the landing page itself public.

`Current.user.services.new` is the payoff of wiring up `has_many :services` a few paragraphs ago: it builds a new `Service` already associated with whoever is signed in, so there's no `user_id` in `service_params` for a visitor to tamper with — the provider is whoever `Current.user` says it is, not whatever a hidden form field claims.

`service_params` permits `:price`, not `:price_cents` — the controller talks to the same dollars-facing interface the form does.

## Routes and the two placeholder buttons

```ruby
# config/routes.rb
resources :services, only: [:index, :new, :create]
```

Only three actions — no `show` yet, since there's no individual service page to link to until a later episode builds one.

Episode 1's landing page shipped with two buttons that did nothing on purpose:

```erb
<span class="... opacity-60 cursor-not-allowed">Browse services</span>
<span class="... opacity-60 cursor-not-allowed">Offer a service</span>
```

They finally go somewhere:

```erb
<%= link_to "Browse services", services_path, class: "..." %>
<%= link_to "Offer a service", authenticated? ? new_service_path : new_registration_path, class: "..." %>
```

"Browse services" is unconditional — the index is public, so the link always makes sense. "Offer a service" checks `authenticated?` (the helper episode 2's concern exposes) and sends a signed-out visitor to sign up first rather than to a form that would just bounce them to the sign-in page anyway. Same destination either way, eventually — just one fewer redirect for the common case of someone who isn't signed in yet.

## A views-only gotcha: the error said "price cents"

Submitting the form empty for the first time, before fixing this, produced:

```
Category must exist
Title can't be blank
Description can't be blank
Price cents can't be blank
Price cents is not a number
```

Everything else reads naturally — "Title can't be blank" — because Rails derives the human-readable label from the attribute name via `humanize`. But the attribute actually being validated is `price_cents`, not `price`, so that's the name that leaked into the message. A user filling out this form has never heard of `price_cents`; the form field right below the error just says "Price (USD)".

The fix is a one-line I18n override, not a code change to the model — the validation is correct, only its rendered name was wrong:

```yaml
# config/locales/en.yml
en:
  activerecord:
    attributes:
      service:
        price_cents: "Price"
```

One detail that cost a second pass: writing the override as `price` (lowercase) produced "price can't be blank" — lowercase, inconsistent with "Title" and "Description" next to it. Rails' default `humanize` capitalizes automatically; a custom I18n string is used exactly as written, with no capitalization applied on top. The fix was just capitalizing the override itself, `"Price"`.

## Trying it

```bash
bin/dev
```

Sign up (or sign in), click "Offer a service", fill in a title, pick a category, write a description, and a price like `42.50`. Submit, and it redirects to `/services` with "Your service is live." in a flash banner, the listing right below it — category, title, whoever posted it, description, and `$42.50`, not `4250`. Sign out and visit `/services` directly: still there, still public. Try `/services/new` while signed out and it redirects to sign-in, same as any other protected page.

## What's next

Episode 4 builds `Booking` — the record of an agreement between two users, and the flow that actually lets someone book a service that's been listed.
