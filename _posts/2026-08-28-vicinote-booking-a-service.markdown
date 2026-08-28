---
layout: post
title: "VicinoTe: Booking a Service"
series: "vicinote"
episode: 4
lang: en
ref: vicinote-booking-a-service
permalink: /vicinote-booking-a-service/
canonical_url: https://antoninoscaffidi.github.io/vicinote-booking-a-service/
date: 2026-08-28 05:00:00 +0200
image: /assets/images/vicinote-ep4-banner.png
---

[Episode 3]({% post_url 2026-08-20-vicinote-services-and-categories %}) got a signed-in user to the point of listing something they offer. Nobody could actually book it, though — there was no individual page for a service, and no record of an agreement between two people. This episode closes both gaps: `Service` finally gets a `show` page, and a new `Booking` model is the record of one user agreeing to pay another for something, on a specific day, at a specific price.

Code is tagged [`episode-4`](https://github.com/AntoninoScaffidi/vicinote-tutorial/tree/episode-4) in the [vicinote-tutorial](https://github.com/AntoninoScaffidi/vicinote-tutorial) repo.

## What this episode touches, at a glance

```
db/migrate/..._create_bookings.rb     new  — the bookings table
app/models/booking.rb                 new  — Booking: belongs_to :service/:customer, validations
app/models/concerns/priceable.rb      new  — the euros<->cents accessor, extracted so Booking can share it
app/models/service.rb                 edit — has_many :bookings, uses Priceable
app/models/user.rb                    edit — adds has_many :bookings and :received_bookings
app/controllers/services_controller.rb edit — adds #show
app/controllers/bookings_controller.rb new — #create and #index
app/views/services/show.html.erb      new — service details plus the booking form
app/views/services/index.html.erb     edit — service titles now link to their show page
app/views/bookings/index.html.erb     new — "My bookings", split into booked vs. received
app/views/layouts/application.html.erb edit — a "My bookings" nav link
config/routes.rb                      edit — service show, nested bookings#create, bookings#index
```

## Generating the model

```bash
bin/rails generate model Booking service:references customer:references price_cents:integer status:string scheduled_on:date
```

`service:references` behaves exactly as it did for `Service` back in episode 3 — a foreign key column, an index, and a `belongs_to :service` written straight into the generated model. `customer:references` does the same thing, but wrongly: there's no `Customer` model in this app. "Customer" is a role a `User` plays in a given booking, not a class of its own — the same "role emerges from the association" decision episode 1 made about `Service` providers, applied here to the other side of a booking.

The generator has no way to know that, so it guesses based on the name alone. What it actually produced:

```ruby
# generated, before editing
t.references :customer, null: false, foreign_key: true
```

```ruby
# generated, before editing
belongs_to :customer
```

Left alone, the migration would try to add a foreign key to a `customers` table that will never exist, and the model would look up a `Customer` class that will never exist either — both would fail the moment either line actually ran. Both needed one explicit correction.

## The migration, corrected

```ruby
# db/migrate/..._create_bookings.rb
class CreateBookings < ActiveRecord::Migration[8.1]
  def change
    create_table :bookings do |t|
      t.references :service, null: false, foreign_key: true
      t.references :customer, null: false, foreign_key: { to_table: :users }
      t.integer :price_cents, null: false
      t.string :status, null: false, default: "pending"
      t.date :scheduled_on, null: false

      t.timestamps
    end
  end
end
```

- `t.references :customer, null: false, foreign_key: { to_table: :users }` — the fix for the wrong guess. `foreign_key: true` alone infers the target table from the reference's own name (`customer` → `customers`); passing a hash with `to_table:` overrides that inference and points the constraint at the real table, `users`, while the column itself stays named `customer_id` — the name that actually matters for the code that reads it.
- `t.integer :price_cents, null: false` — no default here, deliberately. A booking always has an agreed price; there's no reasonable value to fall back to if one isn't given.
- `t.string :status, null: false, default: "pending"` — every booking starts in the same state. This episode never changes it — nothing here builds a "confirm" or "cancel" action — but the column, and the set of values it's allowed to hold (enforced in the model below), are already in place for whenever that's built.
- `t.date :scheduled_on, null: false` — the one piece of information the customer actually supplies: which day this is for.

## Sharing the price accessor: the Priceable concern

Episode 3 gave `Service` a small `price`/`price=` pair to speak euros while `price_cents` stays what's stored. `Booking` needs the exact same behavior — it has its own `price_cents` column now too. Copying those four lines a second time would work, but two independent copies of the same logic tend to drift apart the moment one of them needs a fix and the other is forgotten. This is what `ActiveSupport::Concern` exists for:

```ruby
# app/models/concerns/priceable.rb
module Priceable
  extend ActiveSupport::Concern

  def price
    price_cents && price_cents / 100.0
  end

  def price=(value)
    self.price_cents = value.present? ? (value.to_f * 100).round : nil
  end
end
```

Nothing about this code changed from episode 3 — it's the same two methods, lifted verbatim out of `Service` into their own file under `app/models/concerns/`, a directory Rails autoloads by convention without any require needed. `extend ActiveSupport::Concern` is what makes `include Priceable` behave predictably even if the concern later grows a `included do ... end` block or its own class methods — for a module this simple it's not strictly required, but it's the standard shape a Rails concern takes, and reaching for it consistently means never having to remember which modules need the special treatment and which don't.

```ruby
# app/models/service.rb
class Service < ApplicationRecord
  include Priceable

  belongs_to :user
  belongs_to :category
  has_many :bookings

  validates :title, presence: true
  validates :description, presence: true
  validates :price_cents, presence: true, numericality: { greater_than: 0, only_integer: true }
end
```

`Service` lost its two price methods and gained `include Priceable` and `has_many :bookings` instead — the other half of `Booking belongs_to :service`, needed the moment anything wants to ask a service for the bookings made against it.

## The Booking model

```ruby
# app/models/booking.rb
class Booking < ApplicationRecord
  include Priceable

  belongs_to :service
  belongs_to :customer, class_name: "User"

  validates :scheduled_on, presence: true
  validates :status, presence: true, inclusion: { in: %w[pending confirmed completed cancelled] }
  validates :price_cents, presence: true, numericality: { greater_than: 0, only_integer: true }

  validate :customer_is_not_the_provider

  private

  def customer_is_not_the_provider
    return unless service && customer_id

    errors.add(:customer, "can't book their own service") if service.user_id == customer_id
  end
end
```

- `belongs_to :customer, class_name: "User"` — the model-side half of the same correction the migration needed. `class_name: "User"` tells Rails "the association is called `customer`, but the actual class on the other end is `User`" — without it, Rails would try to constantize `"Customer"` and fail. The foreign key (`customer_id`) is still inferred correctly from the association name; only the class needed spelling out.
- `validates :status, ..., inclusion: { in: %w[pending confirmed completed cancelled] }` — the database column is a plain string with no constraint of its own beyond `null: false`; this is what actually stops a `Booking` from ever holding a typo or an invented status Rails doesn't recognize. Only `"pending"` is reachable through anything built in this episode, but the full set is declared now rather than grown one string at a time later.
- `validate :customer_is_not_the_provider` — a validation that isn't checking one column against a fixed rule, but one association against another. `return unless service && customer_id` guards against running the comparison at all when either side is still missing (a brand-new, blank `Booking` shouldn't produce this specific error just because nothing's been filled in yet — the presence validations above already cover that case). The actual check, `service.user_id == customer_id`, is what turns "you can't book your own service" from a sentence in episode 1's design document into something the database can never actually contain.

Notice what's *not* here: no `before_validation` that copies `service.price_cents` into the booking automatically. That's deliberate, and the reasoning is in the controller next — the price snapshot happens where it's visible, not hidden inside a callback.

## Wiring the associations episode 1 only sketched

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password
  has_many :sessions, dependent: :destroy
  has_many :services, dependent: :destroy
  has_many :bookings, foreign_key: :customer_id, dependent: :destroy, inverse_of: :customer
  has_many :received_bookings, through: :services, source: :bookings

  normalizes :email_address, with: ->(e) { e.strip.downcase }
end
```

Episode 1's domain design showed both of these lines as illustrations of "one `User`, two roles emerging from associations, not a role column." Episode 3 wired up `has_many :services`; these two are the ones that were still just a sketch until now.

- `has_many :bookings, foreign_key: :customer_id, dependent: :destroy, inverse_of: :customer` — without `foreign_key:`, Rails would look for a `user_id` column on `bookings`, which doesn't exist; the actual column is `customer_id`, so it has to be named explicitly. `inverse_of: :customer` tells Rails that this association and `Booking belongs_to :customer` are two sides of the same relationship, which lets Rails skip a redundant database query when it already has one side loaded in memory and needs the other. `dependent: :destroy` matches the same policy already applied to `sessions` and `services`: deleting a `User` shouldn't leave `Booking` rows pointing at a `customer_id` that no longer exists.
- `has_many :received_bookings, through: :services, source: :bookings` — a `has_many :through`, not a direct foreign key on `users` at all. Read right to left: for each of this user's `services`, follow that service's own `bookings` association, and treat the combined result as this user's `received_bookings`. `source: :bookings` is only needed because the association name on this side (`received_bookings`) doesn't match the association name being followed on `Service` (`bookings`) — without it, Rails would look for a `received_bookings` method on `Service`, which doesn't exist, instead of the real `bookings` one.

The distinction between these two matters: `user.bookings` is "what I've booked, as a customer" — a direct relationship. `user.received_bookings` is "what's been booked from me, across every service I offer" — a relationship that only exists by way of a second table in between. Same underlying `bookings` table, two genuinely different questions.

## Finally: a page for one service

```ruby
# app/controllers/services_controller.rb
class ServicesController < ApplicationController
  allow_unauthenticated_access only: %i[index show]

  def index
    @services = Service.includes(:category, :user).order(created_at: :desc)
  end

  def show
    @service = Service.includes(:category, :user).find(params[:id])
    @booking = Booking.new
  end

  # new/create unchanged from episode 3
end
```

`allow_unauthenticated_access only: %i[index show]` — episode 3 exempted only `index` from the sign-in requirement; `show` joins it here, for the same reason: browsing a single listing has to work for a visitor who hasn't signed up, or the marketplace is only visible to people who already committed to it. `@booking = Booking.new` builds an empty, unsaved `Booking` purely so the view's `form_with model: @booking` has something to build a form around — nothing here saves it or even fills in its attributes yet.

```ruby
# config/routes.rb
resources :services, only: [:index, :new, :create, :show] do
  resources :bookings, only: [:create]
end
resources :bookings, only: [:index]
```

```
          Prefix Verb URI Pattern                              Controller#Action
service_bookings POST /services/:service_id/bookings(.:format) bookings#create
        bookings GET  /bookings(.:format)                      bookings#index
```

Nesting `resources :bookings, only: [:create]` inside `resources :services` is what produces `POST /services/:service_id/bookings` and the `service_bookings_path(service)` helper used in the view below — the URL itself says which service a new booking is for, so the controller never has to guess it from anywhere else. The second, unnested `resources :bookings, only: [:index]` is deliberately separate: "list of my bookings" isn't scoped to any one service, so it doesn't belong under `/services/:service_id/` at all.

## Booking, and where the price actually gets copied

```ruby
# app/controllers/bookings_controller.rb
class BookingsController < ApplicationController
  def index
    @my_bookings = Current.user.bookings.includes(:service).order(scheduled_on: :asc)
    @received_bookings = Current.user.received_bookings.includes(:service, :customer).order(scheduled_on: :asc)
  end

  def create
    service = Service.find(params[:service_id])
    booking = Current.user.bookings.new(booking_params.merge(service: service, price_cents: service.price_cents))

    if booking.save
      redirect_to bookings_path, notice: "Booked for #{booking.scheduled_on}."
    else
      redirect_to service_path(service), alert: booking.errors.full_messages.to_sentence
    end
  end

  private

  def booking_params
    params.require(:booking).permit(:scheduled_on)
  end
end
```

No `allow_unauthenticated_access` anywhere in this controller — both actions stay behind the default sign-in requirement, which is correct: nobody should list or make a booking anonymously.

- `def index` — two separate queries assigned to two separate instance variables, one for each side of a booking a signed-in user could be on. `Current.user.bookings` uses the direct association; `Current.user.received_bookings` uses the `has_many :through` one. Both get `includes(...)` for the same reason episode 3's service index did: rendering a list of bookings and reading `booking.service.title` (or `booking.customer.email_address`) for each one would otherwise fire a fresh query per row.
- `service = Service.find(params[:service_id])` — `:service_id`, not `:id`, because of the nested route: a request to `POST /services/7/bookings` carries the service's id under that key, not the booking's (which doesn't exist yet).
- `Current.user.bookings.new(booking_params.merge(service: service, price_cents: service.price_cents))` — this single line is the entire price-snapshot mechanism from episode 1's design, made concrete. `booking_params` only ever contains `scheduled_on` — the one field the form actually submits — so `price_cents: service.price_cents` has to be added explicitly, here, by the controller, reading the service's *current* price at the exact moment someone commits to booking it. Nothing links the resulting `Booking` back to `Service#price` after this line runs; if the provider raises their price tomorrow, every booking made today keeps the number that was true when it was made. `Current.user.bookings.new(...)`, building through the association rather than `Booking.new(customer: Current.user, ...)`, is also what fixes the customer side to whoever is actually signed in — there's no `customer_id` anywhere in `booking_params` for a visitor to tamper with.
- `if booking.save` / the two branches — on success, redirect to the bookings list with a flash notice, the same Post/Redirect/Get shape every write action in this series has used since episode 2. On failure — someone tried to book their own service, or the date was left blank — there's no dedicated `bookings/new` view to re-render with inline errors, because the form lives on someone else's page (the service's `show`), not its own. Redirecting back to `service_path(service)` with `alert: booking.errors.full_messages.to_sentence` is the simpler fit for that shape: the errors show up as one flash message rather than field-by-field, which is a real trade-off (less precise feedback) made deliberately rather than forcing a `render` across controllers to avoid it.

## The service page and its booking form

```erb
<%# app/views/services/show.html.erb %>
<div class="max-w-2xl mx-auto w-full">
  <% if alert = flash[:alert] %>
    <p class="py-2 px-3 bg-red-50 mb-5 text-red-700 font-medium rounded-lg inline-block" id="alert"><%= alert %></p>
  <% end %>

  <p class="text-xs font-semibold uppercase tracking-wide text-indigo-600"><%= @service.category.name %></p>
  <div class="mt-1 flex items-start justify-between gap-4">
    <h1 class="text-2xl font-bold text-gray-900"><%= @service.title %></h1>
    <p class="whitespace-nowrap text-2xl font-semibold text-gray-900"><%= number_to_currency(@service.price) %></p>
  </div>
  <p class="mt-1 text-sm text-gray-500">by <%= @service.user.email_address %></p>
  <p class="mt-4 text-gray-600"><%= @service.description %></p>

  <hr class="my-8 border-gray-200">

  <% if !authenticated? %>
    <p class="text-gray-600">
      <%= link_to "Sign in", new_session_path, class: "text-indigo-600 hover:underline" %> to book this service.
    </p>
  <% elsif Current.user == @service.user %>
    <p class="text-gray-500">This is your own service.</p>
  <% else %>
    <h2 class="text-lg font-semibold text-gray-900 mb-3">Book this service</h2>
    <%= form_with model: @booking, url: service_bookings_path(@service), class: "flex items-end gap-3" do |form| %>
      <div>
        <%= form.label :scheduled_on, "Date", class: "block text-sm font-medium text-gray-700" %>
        <%= form.date_field :scheduled_on, class: "mt-1 block rounded-md border border-gray-300 px-3 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500" %>
      </div>
      <%= form.submit "Book for #{number_to_currency(@service.price)}", class: "rounded-md bg-indigo-600 px-4 py-2 text-white font-medium hover:bg-indigo-700 cursor-pointer" %>
    <% end %>
  <% end %>
</div>
```

- `<% if !authenticated? %> ... <% elsif Current.user == @service.user %> ... <% else %> ... <% end %>` — a three-way branch on who's looking at this page: an anonymous visitor (prompted to sign in, not shown a form that would just fail server-side anyway), the provider themselves (told plainly this is their own listing, since booking your own service makes no sense and the controller would reject it regardless), or anyone else (the actual booking form). The model's `customer_is_not_the_provider` validation is the real enforcement; hiding the form here is the user-facing half of the same rule, not a replacement for it — the classic client-side-convenience-plus-server-side-truth pairing.
- `form_with model: @booking, url: service_bookings_path(@service), ...` — `service_bookings_path(@service)` is the nested route's helper, producing `/services/7/bookings` for service `7`; passing `@service` (an ActiveRecord object) rather than its `id` lets Rails call `.to_param` on it, which for a plain, unmodified `id` primary key just returns the id itself.
- `form.date_field :scheduled_on` — a native HTML5 date input, which gets a calendar-style picker in every modern browser for free, no JavaScript written for it.
- `form.submit "Book for #{number_to_currency(@service.price)}", ...` — the button's own label states the exact price being agreed to, computed the same way the page's price display above it was, so nobody submits without having just seen the number they're committing to.

## The bookings index

```erb
<%# app/views/bookings/index.html.erb %>
<div class="max-w-2xl mx-auto w-full">
  <% if notice = flash[:notice] %>
    <p class="py-2 px-3 bg-green-50 mb-5 text-green-700 font-medium rounded-lg inline-block" id="notice"><%= notice %></p>
  <% end %>

  <h1 class="text-2xl font-bold text-gray-900 mb-6">My bookings</h1>

  <h2 class="text-sm font-semibold uppercase tracking-wide text-gray-500 mb-3">Services you've booked</h2>
  <% if @my_bookings.none? %>
    <p class="text-gray-500 mb-8">You haven't booked anything yet.</p>
  <% else %>
    <div class="space-y-3 mb-8">
      <% @my_bookings.each do |booking| %>
        <div class="rounded-lg border border-gray-200 p-4 flex items-center justify-between">
          <div>
            <p class="font-medium text-gray-900"><%= booking.service.title %></p>
            <p class="text-sm text-gray-500"><%= booking.scheduled_on.strftime("%-d %B %Y") %> &middot; <%= booking.status %></p>
          </div>
          <p class="font-semibold text-gray-900"><%= number_to_currency(booking.price) %></p>
        </div>
      <% end %>
    </div>
  <% end %>

  <h2 class="text-sm font-semibold uppercase tracking-wide text-gray-500 mb-3">Bookings received</h2>
  <% if @received_bookings.none? %>
    <p class="text-gray-500">Nobody has booked one of your services yet.</p>
  <% else %>
    <div class="space-y-3">
      <% @received_bookings.each do |booking| %>
        <div class="rounded-lg border border-gray-200 p-4 flex items-center justify-between">
          <div>
            <p class="font-medium text-gray-900"><%= booking.service.title %></p>
            <p class="text-sm text-gray-500">
              <%= booking.scheduled_on.strftime("%-d %B %Y") %> &middot; <%= booking.status %> &middot; booked by <%= booking.customer.email_address %>
            </p>
          </div>
          <p class="font-semibold text-gray-900"><%= number_to_currency(booking.price) %></p>
        </div>
      <% end %>
    </div>
  <% end %>
</div>
```

Two independent lists on one page, each with its own empty state, mirroring the controller's two separate instance variables exactly — `@my_bookings` renders as "services you've booked," `@received_bookings` as "bookings received." `booking.price`, not `booking.service.price` — every number on this page is the frozen price from the moment of booking, deliberately never the service's current one. `booking.customer.email_address` only appears in the received list — you always know who booked from you; you don't need reminding who *you* are in your own booked list.

## Trying it, without a browser

The usual `bin/dev` walkthrough applies here too — sign in as one user, visit a service someone else listed, pick a date, book it, then check `/bookings` both as the customer and, signed in as the provider, on the receiving end. This episode's real verification happened a different way, straight in `bin/rails runner`, specifically to prove the one behavior that matters most and is hardest to eyeball from a screenshot — that a booking's price genuinely never moves after it's made:

```ruby
maria = User.find_by(email_address: "maria@example.com")
service = maria.services.first
customer = User.find_or_create_by!(email_address: "luca@example.com") { |u| u.password = "supersecret123" }

booking = customer.bookings.new(service: service, scheduled_on: Date.tomorrow, price_cents: service.price_cents)
booking.save!
puts "Booked at: #{booking.price}"

service.update!(price: 99.99)
puts "Service price now: #{service.reload.price}, booking price still: #{booking.reload.price}"
```

```
Booked at: 42.5
Service price now: 99.99, booking price still: 42.5
```

Along with that, trying `maria.bookings.new(service: service, ...)` — Maria attempting to book her own guitar lessons — came back `valid? false`, with exactly the message the model was written to produce: `"Customer can't book their own service"`.

## What's next

Episode 5 covers `Review` — left after a booking is completed, belonging to the booking rather than the service, exactly as episode 1's design decided: only someone who actually went through with a booking can leave one.
