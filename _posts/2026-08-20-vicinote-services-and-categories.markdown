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
image: /assets/images/vicinote-ep3-banner.png
---

[Episode 2]({% post_url 2026-08-13-vicinote-authentication-with-rails-8 %}) got accounts working end to end — sign up, sign in, sign out, password reset — and left it there deliberately: nothing touched `Service` or `Booking` yet. This episode is where the marketplace actually starts being a marketplace: a signed-in user can list something they offer, and anyone can browse what's listed.

It's also where the `has_many :services` association from [episode 1]({% post_url 2026-08-08-vicinote-project-setup-and-domain %})'s domain sketch finally gets written into the `User` model. It's been sitting in a blog post as a design decision for two episodes; today it becomes real code.

This is a long one on purpose — the goal is that nothing in the diff is left unexplained: every generated file, every line we added by hand, every option passed to every method.

Code is tagged [`episode-3`](https://github.com/AntoninoScaffidi/vicinote-tutorial/tree/episode-3) in the [vicinote-tutorial](https://github.com/AntoninoScaffidi/vicinote-tutorial) repo.

## What this episode touches, at a glance

Before going file by file, here's the map. Two new models, one changed model, one new controller, two new views, a routing change, a seed file, and a one-line translation fix:

```
db/migrate/..._create_categories.rb   new  — the categories table
db/migrate/..._create_services.rb     new  — the services table
app/models/category.rb                new  — Category: has_many :services, validates :name
app/models/service.rb                 new  — Service: belongs_to :user/:category, validations, price accessor
app/models/user.rb                    edit — adds has_many :services
app/controllers/services_controller.rb new — index (public), new, create (both signed-in only)
app/views/services/index.html.erb     new  — the listing page
app/views/services/new.html.erb       new  — the "offer a service" form
config/routes.rb                      edit — resources :services, only: [:index, :new, :create]
config/locales/en.yml                 edit — fixes a leaked internal attribute name in error messages, sets the default currency to euros
db/seeds.rb                           edit — the fixed list of categories
app/views/pages/home.html.erb         edit — the two placeholder buttons now link somewhere
```

## Generating the two models

```bash
bin/rails generate model Category "name:string:uniq"
bin/rails generate model Service title:string description:text price_cents:integer user:references category:references
```

Every `field:type` pair after the model name becomes a migration column. The syntax has a few extra tricks worth spelling out, since both commands use them:

- `name:string:uniq` — the third segment, `:uniq`, isn't a type. It tells the generator to also add a unique index on that column, so it writes an `add_index` call for us instead of us remembering to add one later.
- `user:references` and `category:references` — `references` isn't a column type either, it's a generator shorthand meaning "this model belongs to that one." It expands into a `t.references` call in the migration (a foreign-key integer column plus an index), *and* it makes the generator write a `belongs_to :user` / `belongs_to :category` line directly into the generated model file. That's why `Service`'s model already has both associations in it the moment the generator finishes — we didn't type `belongs_to` ourselves.

Each command prints what it created:

```
      create    db/migrate/20260820055929_create_categories.rb
      create    app/models/category.rb
      invoke    test_unit
      create      test/models/category_test.rb
      create      test/fixtures/categories.yml
```

```
      create    db/migrate/20260820055946_create_services.rb
      create    app/models/service.rb
      invoke    test_unit
      create      test/models/service_test.rb
      create      test/fixtures/services.yml
```

The `test_unit` files are Rails' default test scaffolding (an empty test class and an empty fixture file) — this series isn't using them, so they're left as generated and not discussed further.

## The categories migration, line by line

Generated, then hand-edited to add one word:

```ruby
# db/migrate/..._create_categories.rb
class CreateCategories < ActiveRecord::Migration[8.1]
  def change
    create_table :categories do |t|
      t.string :name, null: false

      t.timestamps
    end
    add_index :categories, :name, unique: true
  end
end
```

- `class CreateCategories < ActiveRecord::Migration[8.1]` — every migration is a subclass of `ActiveRecord::Migration`, versioned to the Rails release that generated it (`[8.1]`). That version pin is what lets Rails change migration DSL behavior across major versions without silently breaking migrations written for an older one.
- `def change` — the one method Rails needs. For operations it can reverse automatically (creating a table, adding a column, adding an index), `change` is enough; Rails infers how to undo it if you ever roll the migration back. Irreversible operations would need separate `up`/`down` methods instead — nothing here needs that.
- `create_table :categories do |t|` — opens the table definition block; `t` is the object every column gets defined on.
- `t.string :name, null: false` — a `VARCHAR` column. The generator wrote `t.string :name`; the `null: false` is the one word we added by hand, matching the same rigor episode 2 put into the `users` table. Without it, Postgres would happily store a category with no name at all, and that's not a state the app has any use for.
- `t.timestamps` — shorthand for two columns, `created_at` and `updated_at`, both `datetime`, both filled in automatically by ActiveRecord on create/update. Almost every table in this series has this line; `Category` is no exception.
- `add_index :categories, :name, unique: true` — this is what `:uniq` in the generator command produced. It's a *database-level* uniqueness guarantee, enforced by Postgres itself, not just by a Rails validation that could in theory be bypassed by two simultaneous requests racing each other. `Category` also validates uniqueness in the model (below) — belt and suspenders: the model validation gives a friendly error in the normal case, the index guarantees correctness even under a race.

## The services migration, line by line

```ruby
# db/migrate/..._create_services.rb
class CreateServices < ActiveRecord::Migration[8.1]
  def change
    create_table :services do |t|
      t.string :title, null: false
      t.text :description, null: false
      t.integer :price_cents, null: false
      t.references :user, null: false, foreign_key: true
      t.references :category, null: false, foreign_key: true

      t.timestamps
    end
  end
end
```

- `t.string :title, null: false` — same shape as `Category#name`: a required short text field.
- `t.text :description, null: false` — `text` instead of `string`. In Postgres this is really a formality (both map to the same unbounded `text` type under the hood; Rails' `string` vs `text` distinction is mostly a Rails-level hint, not a Postgres storage difference), but it signals intent: this column holds a paragraph, not a label, and some form helpers (`form.text_area` instead of `form.text_field`) key off exactly this type later.
- `t.integer :price_cents, null: false` — a plain integer. There's a whole section below on why this is an integer and not a `decimal`.
- `t.references :user, null: false, foreign_key: true` — this line is what `user:references` on the command line generated, and it's doing three things at once:
  1. Adding an integer column named `user_id` (the `_id` suffix and the pluralization-to-singular are both Rails convention, not something we typed).
  2. `foreign_key: true` — adds an actual Postgres foreign key constraint from `services.user_id` to `users.id`. The database itself will now refuse to insert a `Service` row whose `user_id` doesn't match a real user, and refuses to delete a `User` row that still has services pointing at it (unless the association says otherwise — more on that below).
  3. An index on `user_id`, added automatically, because a foreign-key column that isn't indexed makes every query that joins through it slow as the table grows. This one wasn't optional or something we asked for — `t.references` always indexes.
  `null: false` here means every service *must* belong to somebody; there's no such thing as a service with no provider.
- `t.references :category, null: false, foreign_key: true` — identical shape, for the other side of the relationship.

Notice neither migration says anything about `belongs_to` or `has_many` — migrations only describe the *database schema* (tables, columns, constraints, indexes). Associations are a separate, Ruby-level concept declared in the models, which is the next section.

## Running the migrations

```bash
bin/rails db:migrate
```

This runs both pending migrations in timestamp order (categories, then services — services' foreign key to categories needs the categories table to already exist) and rewrites `db/schema.rb` to reflect the new state. `schema.rb` isn't something to edit by hand; it's Rails' cached snapshot of "what the database currently looks like," regenerated every time you migrate, and it's what a teammate's `bin/rails db:setup` reads to build a fresh database that matches yours without replaying every migration ever written.

## The generated models, before any edits

Right after the generator ran, before we touched anything:

```ruby
# app/models/category.rb
class Category < ApplicationRecord
end
```

```ruby
# app/models/service.rb
class Service < ApplicationRecord
  belongs_to :user
  belongs_to :category
end
```

`Category` is empty because it has no `references` columns — there was nothing for the generator to infer an association from. `Service` already has both `belongs_to` lines because of the `:references` fields we passed on the command line, as explained above. Both inherit from `ApplicationRecord`, the abstract base class every model in a Rails app shares (itself a thin subclass of `ActiveRecord::Base`), which is what gives them `.find`, `.create`, `.where`, validations, and everything else ActiveRecord provides — none of that is written in either file, it's inherited.

One thing worth naming explicitly: as of Rails 5, `belongs_to` is **required by default**. Writing `belongs_to :user` doesn't just declare the association, it also implicitly adds a presence validation — a `Service` without a `user` fails validation, on top of the database already refusing it via the `null: false, foreign_key: true` from the migration. Two independent layers enforcing the same rule, at two different levels (Ruby validation vs. SQL constraint), which is exactly the "Category must exist" error message you'll see later in this post — that sentence is Rails' default wording for a failed `belongs_to` presence check on `category`.

## Category, filled in

```ruby
# app/models/category.rb
class Category < ApplicationRecord
  has_many :services

  validates :name, presence: true, uniqueness: true
end
```

- `has_many :services` — the other half of `Service belongs_to :category`. Rails associations are declared on both ends by hand; there's no way to declare just one side and have the other inferred. This is what makes `some_category.services` work — the association looks up every `Service` row whose `category_id` matches, via the foreign key the migration created.
- `validates :name, presence: true, uniqueness: true` — `presence: true` duplicates what the `null: false` in the migration already guarantees at the database level, but for a different audience: a failed database constraint raises an ugly `ActiveRecord::NotNullViolation` exception, while a failed presence validation gives `@category.errors` a friendly message a view can render. `uniqueness: true` is the model-level half of the belt-and-suspenders pair with the migration's unique index — this one runs a `SELECT` before saving and gives a clean error; the index is what actually stops a duplicate from ever reaching the table if two requests race each other past the validation at the same instant.

## Service, filled in

```ruby
# app/models/service.rb
class Service < ApplicationRecord
  belongs_to :user
  belongs_to :category

  validates :title, presence: true
  validates :description, presence: true
  validates :price_cents, presence: true, numericality: { greater_than: 0, only_integer: true }

  # price_cents is what's stored and compared (no float rounding surprises),
  # but nobody wants to type cents into a form. This speaks euros on the
  # way in and out, so the form field can just be "price".
  def price
    price_cents && price_cents / 100.0
  end

  def price=(value)
    self.price_cents = value.present? ? (value.to_f * 100).round : nil
  end
end
```

- `validates :title, presence: true` / `validates :description, presence: true` — same reasoning as `Category#name`: the migration's `null: false` is the last line of defense, this is the friendly one that runs first.
- `validates :price_cents, presence: true, numericality: { greater_than: 0, only_integer: true }` — three checks bundled into one line. `presence: true` rejects `nil`. `numericality:` on its own would just reject anything that isn't a number at all (a string like `"abc"`); the `greater_than: 0` option adds the business rule that a free service isn't a thing this validation allows, and `only_integer: true` rejects fractional cents (`4250.5`), which shouldn't be reachable anyway since `price_cents` is a database `integer` column, but the validation makes the rule explicit rather than relying on the column type alone to enforce it before the value ever reaches the database.
- `def price` / `def price=` — covered in full in the next section; this is the piece that lets a form talk in euros while the column underneath stays cents.

## A decision worth slowing down on: price_cents, not price

The generator wrote `price_cents:integer`, not `price:decimal`, on the command line — that phrasing was chosen deliberately going in, and it's the same category of decision episode 1 made about `Booking` storing its own price rather than reading `service.price` live: money handled as a float, or even a naive `decimal`, eventually produces a rounding error that shows up as a few cents off on an invoice, at the worst possible moment. Storing the amount as an integer number of cents sidesteps the whole class of bug — there's no fractional part to round, because there's no fraction at all. €42.50 is stored as the integer `4250`, full stop. Naming it `price_cents` rather than, say, `price_usd_cents` is deliberate too — "cents" is the minor unit of euros just as much as it is of dollars, so nothing about the column itself is tied to a currency; only the formatting layer, covered later in this post, has any idea which one is in use.

The cost is that nobody wants to type "4250" into a form and mean €42.50. So `Service` gets a small virtual accessor — two plain Ruby methods, not a database column — that translates euros to cents and back:

```ruby
def price
  price_cents && price_cents / 100.0
end

def price=(value)
  self.price_cents = value.present? ? (value.to_f * 100).round : nil
end
```

Walking through both:

- `def price` — reads for display. `price_cents && price_cents / 100.0` uses Ruby's `&&` for its short-circuit behavior, not as a boolean check: if `price_cents` is `nil` (a brand-new, unsaved `Service`), the whole expression short-circuits to `nil` without attempting `nil / 100.0`, which would raise. If it's a real integer, `&&` evaluates and returns the right-hand side — the division. Dividing by `100.0` (a float literal, not `100`) forces Ruby to do floating-point division rather than integer division, so `4250 / 100.0` gives `42.5`, not `42` truncated.
- `def price=(value)` — the setter, called automatically whenever something does `service.price = "42.50"` or, just as automatically, whenever a form submits a field named `price` through mass assignment (`Service.new(price: "42.50", ...)`) — Rails calls the setter method for every permitted attribute, it doesn't care whether that method backs a real column or not. `value.present?` guards against blank input (an empty string from a cleared form field) rather than trying to convert `""` into a number. When there *is* a value, `value.to_f * 100` converts to euros-as-a-float and multiplies by 100 to get cents, and `.round` turns that back into a whole number — `.to_f` on user input can produce things like `42.499999999999996` due to ordinary floating-point imprecision, and `.round` is what cleans that back up to exactly `4250` before it ever reaches `price_cents=`.

Because `price=` is a normal Ruby method and not an ActiveRecord-backed attribute, it runs immediately when attributes are assigned — before `save`, before any validation. By the time `validates :price_cents, ...` runs, `price_cents` has already been populated by this setter. The form field discussed later in this post is just `form.text_field :price` — the view never has any idea cents exist.

## Categories are seeded, not created

A marketplace where anyone can invent a new category ends up with fifty categories that mean the same thing, spelled five different ways, and browsing-by-category stops being useful. VicinoTe curates a fixed list instead, in `db/seeds.rb`:

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

- The literal array of eight strings *is* the list of categories, in plain Ruby, right there in the seed file — no admin UI, no separate config format, just a source-controlled array a future episode could extend by adding a ninth string.
- `.each do |name| ... end` iterates the array once per category.
- `Category.find_or_create_by!(name: name)` — this single method does two jobs depending on what it finds: if a `Category` with that `name` already exists, it returns it untouched; if not, it builds and saves a new one. The trailing `!` means it raises `ActiveRecord::RecordInvalid` on a validation failure instead of silently returning an unsaved, invalid record — which matters here, because a seed file failing loudly is much easier to debug than one that fails quietly and leaves the database half-seeded.

That combination — a fixed array plus `find_or_create_by!` — is what makes the file safe to run more than once. Running `db:seed` again after adding a ninth category to the array later creates only the new one; the first eight are matched by name and left alone, not duplicated.

```bash
bin/rails db:migrate
bin/rails db:seed
```

`db:seed` just executes `db/seeds.rb` as a plain Ruby script inside the app's environment — nothing more mysterious than that.

## Writing the association episode 1 only sketched

Episode 1's domain design showed this code as an illustration of the "role emerges from the association" decision — the whole point being that a `User` isn't tagged with a `role: "provider"` column, it's a provider *because* it has services. It was never actually in the `User` model, though; episode 2 was about authentication and didn't touch it. It goes in now:

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password
  has_many :sessions, dependent: :destroy
  has_many :services, dependent: :destroy

  normalizes :email_address, with: ->(e) { e.strip.downcase }
end
```

Only one line changed — `has_many :services, dependent: :destroy` was added, everything else (`has_secure_password`, the `sessions` association, the email normalization) is untouched from episode 2.

`dependent: :destroy` matches what `sessions` already does on the line above it, for the same reason: without it, deleting a `User` would either fail outright (the database's `foreign_key: true` constraint from the migration would reject the delete, since `services` rows would still point at a `user_id` that's about to stop existing) or, if the constraint were relaxed, leave orphaned `Service` rows in the table forever, pointing at nobody. `dependent: :destroy` tells Rails to delete every associated `Service` first, automatically, whenever a `User` is destroyed — the cleanup is one word, not something every future call site has to remember to do by hand.

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

Line by line:

- `class ServicesController < ApplicationController` — every controller in a Rails app inherits from `ApplicationController`, which is where episode 2's `Authentication` concern is included. That's what makes the next line meaningful.
- `allow_unauthenticated_access only: :index` — episode 2's `Authentication` concern runs `before_action :require_authentication` for every action on every controller by default, which means *without this line*, an anonymous visitor hitting any action here would be redirected straight to the sign-in page. That's correct for `new` and `create` — nobody should be able to list a service anonymously — but wrong for `index`: browsing the marketplace has to work for someone who hasn't signed up yet, or there's no reason to sign up in the first place. `only: :index` opts that one action back out of the requirement while leaving `new` and `create` protected. This is the exact same mechanism episode 2 used, with no argument at all (`allow_unauthenticated_access`, no `only:`), to make the entire `PagesController` public — here we only want one of three actions exempted, so `only:` narrows it.
- `def index` / `@services = Service.includes(:category, :user).order(created_at: :desc)` — `Service.includes(:category, :user)` isn't filtering anything; `includes` is ActiveRecord's eager-loading method. Without it, a view that loops over `@services` and calls `service.category.name` and `service.user.email_address` for each one would fire one additional `SELECT` per service, per association — the classic N+1 query problem, where displaying 50 services silently means 101 queries (1 for the list, plus 50 for categories, plus 50 for users) instead of 3. `includes` loads the categories and users for every service up front, in a small fixed number of extra queries, regardless of how many services there are. `.order(created_at: :desc)` sorts newest-first, so a new listing shows up at the top of the page rather than the bottom.
- `def new` / `@service = Current.user.services.new` — this is the payoff of wiring `has_many :services` onto `User` a section ago. `Current.user` is the currently authenticated user (from episode 2's `Current` model, an `ActiveSupport::CurrentAttributes` subclass). Calling `.services.new` *through* the association, rather than `Service.new` on its own, pre-fills the new record's `user_id` with `Current.user.id` automatically — the instance this line builds already knows who it belongs to before a single form field has been filled in.
- `def create` — `Current.user.services.new(service_params)` does the same association-scoped build as `new`, but this time passing in the submitted form data as well. Because the `user` side of the association is set by `Current.user.services`, not by anything in `service_params`, there's no `user_id` field anywhere in the permitted params below for a malicious visitor to tamper with and claim a listing on someone else's behalf — the provider is whoever `Current.user` says it is, full stop, not whatever a hidden form field might claim.
- `if @service.save` — `save` runs validations and, if they all pass, performs the `INSERT` and returns `true`; if any validation fails, it does nothing to the database and returns `false` — no exception raised, which is why this is a plain `if`, not a `begin/rescue`.
- `redirect_to services_path, notice: "Your service is live."` — on success, a real HTTP redirect to the index (this is the same Post/Redirect/Get pattern episode 3 of *ai-with-ruby* discusses in more depth: redirecting after a state-changing POST means refreshing the result page never re-submits the form). `notice:` stores a one-time message in the `flash` — it survives exactly one redirect and then clears itself, which is how "Your service is live." shows up on the very next page load and is gone if you refresh again.
- `render :new, status: :unprocessable_entity` — on failure, re-render the *same* form (not redirect anywhere) so `@service` — now populated with both what the user typed and the validation errors attached to it — can be shown back to them with their input intact and the specific problems called out. `status: :unprocessable_entity` sets the HTTP status code to 422 instead of the default 200; the browser still renders the HTML either way, but a 422 correctly tells any tooling watching the response (browser dev tools, Turbo, a test suite) that this was a failed submission, not a successful page load that happens to contain a form.
- `private` — everything below this line is only callable from within the controller itself, not reachable as a route.
- `def service_params` / `params.require(:service).permit(:title, :description, :price, :category_id)` — Rails' strong parameters. `params.require(:service)` raises immediately if the submitted form data has no top-level `service` key at all (a malformed or missing request), and `.permit(...)` is an allowlist: only these four keys are allowed through; anything else present in the raw request params — including, say, a `user_id` someone tried to inject by editing the form's HTML in their browser before submitting — is silently stripped and never reaches `Service.new`. Notice this permits `:price`, not `:price_cents` — the controller talks to the same euros-facing interface the form does; it has no idea `price_cents` exists either.

## Routes and the two placeholder buttons

```ruby
# config/routes.rb
resources :services, only: [:index, :new, :create]
```

`resources :services` is Rails' RESTful routing shorthand — on its own it would generate all seven conventional routes (`index`, `new`, `create`, `show`, `edit`, `update`, `destroy`). `only: [:index, :new, :create]` narrows that down to exactly the three this episode implements. Running `bin/rails routes -g services` shows exactly what got generated:

```
     Prefix Verb URI Pattern             Controller#Action
   services GET  /services(.:format)     services#index
            POST /services(.:format)     services#create
new_service GET  /services/new(.:format) services#new
```

The `Prefix` column is where path helper names like `services_path` and `new_service_path` — used throughout the controller and views in this episode — come from: Rails derives them automatically from the prefix plus `_path` or `_url`. There's no `show` route yet, on purpose — there's no individual service page to link to until a later episode builds one.

Episode 1's landing page shipped with two buttons that did nothing on purpose, styled to look disabled:

```erb
<span class="rounded-md bg-indigo-600 px-4 py-2 text-white font-medium opacity-60 cursor-not-allowed">
  Browse services
</span>
<span class="rounded-md border border-gray-300 px-4 py-2 text-gray-700 font-medium opacity-60 cursor-not-allowed">
  Offer a service
</span>
```

They finally go somewhere:

```erb
<%= link_to "Browse services", services_path, class: "rounded-md bg-indigo-600 px-4 py-2 text-white font-medium hover:bg-indigo-700" %>
<%= link_to "Offer a service", authenticated? ? new_service_path : new_registration_path, class: "rounded-md border border-gray-300 px-4 py-2 text-gray-700 font-medium hover:bg-gray-50" %>
```

Both `<span>` placeholders became `link_to` calls, and the `opacity-60 cursor-not-allowed` classes that visually signalled "not clickable yet" are gone along with the disabled state itself.

- `link_to "Browse services", services_path, ...` — unconditional. The index is public (`allow_unauthenticated_access only: :index` from the controller), so this link always makes sense, signed in or not.
- `link_to "Offer a service", authenticated? ? new_service_path : new_registration_path, ...` — the destination itself is a ternary. `authenticated?` is the `helper_method` episode 2's `Authentication` concern exposes to views (it's just `resume_session`, reused to answer "is anyone signed in" without triggering a redirect the way `require_authentication` would). Signed in, the link goes straight to `new_service_path` — the form. Signed out, it goes to `new_registration_path` — sign-up — instead of straight to the service form, which would just immediately bounce them to the sign-in page anyway via `require_authentication`. Same eventual destination either way; one fewer redirect for the common case of a visitor who isn't signed in yet.

## The "offer a service" form

```erb
<%# app/views/services/new.html.erb %>
<div class="max-w-xl mx-auto">
  <h1 class="text-3xl font-bold text-gray-900">Offer a service</h1>
  <p class="mt-2 text-gray-600">Tell your neighbours what you can help with.</p>

  <%= form_with model: @service, url: services_path, class: "mt-8 space-y-5" do |form| %>
    <% if @service.errors.any? %>
      <div class="rounded-md bg-red-50 px-4 py-3 text-sm text-red-700">
        <ul class="list-disc list-inside">
          <% @service.errors.full_messages.each do |message| %>
            <li><%= message %></li>
          <% end %>
        </ul>
      </div>
    <% end %>

    <div>
      <%= form.label :title, class: "block text-sm font-medium text-gray-700" %>
      <%= form.text_field :title, placeholder: "Guitar lessons for beginners", class: "mt-1 block w-full rounded-md border border-gray-300 px-3 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500" %>
    </div>

    <div>
      <%= form.label :category_id, "Category", class: "block text-sm font-medium text-gray-700" %>
      <%= form.collection_select :category_id, Category.order(:name), :id, :name, { prompt: "Choose a category" }, class: "mt-1 block w-full rounded-md border border-gray-300 px-3 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500" %>
    </div>

    <div>
      <%= form.label :description, class: "block text-sm font-medium text-gray-700" %>
      <%= form.text_area :description, rows: 4, placeholder: "What you offer, your experience, anything a neighbour should know.", class: "mt-1 block w-full rounded-md border border-gray-300 px-3 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500" %>
    </div>

    <div>
      <%= form.label :price, "Price (EUR)", class: "block text-sm font-medium text-gray-700" %>
      <%= form.text_field :price, placeholder: "45.00", inputmode: "decimal", class: "mt-1 block w-40 rounded-md border border-gray-300 px-3 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500" %>
    </div>

    <%= form.submit "Publish", class: "rounded-md bg-indigo-600 px-4 py-2 text-white font-medium hover:bg-indigo-700 cursor-pointer" %>
  <% end %>
</div>
```

- `form_with model: @service, url: services_path, ...` — `model: @service` is what makes this one form work for both a brand-new, unsaved `Service` (from `ServicesController#new`) and, later, one that failed to save and is being re-rendered with errors attached (from `#create`'s `render :new`) — Rails inspects whether the record is persisted to decide the form's HTTP method, though since the routes only define `create`, this form only ever needs to POST. `url: services_path` is explicit here rather than left to be inferred, pointing the submission at the `services#create` route regardless.
- `<% if @service.errors.any? %> ... @service.errors.full_messages.each ... <% end %>` — after a failed `create`, `@service` carries whatever the user typed *and* the specific validation failures attached to it by `save`. `errors.full_messages` turns those into ready-to-read strings like "Title can't be blank" — this is exactly the block that rendered the error lists shown further down this post.
- `form.label :title, ...` / `form.text_field :title, ...` — a standard labeled text input. `form.label :title` without an explicit label text infers "Title" from the attribute name automatically (via the same `humanize` mechanism discussed in the I18n section below); `placeholder:` is just an HTML attribute passed straight through.
- `form.collection_select :category_id, Category.order(:name), :id, :name, { prompt: "Choose a category" }, class: "..."` — this is the one helper in the form worth slowing down on, because it takes five separate arguments before the HTML options hash:
  1. `:category_id` — the attribute being set on submit (matches `belongs_to :category`'s foreign key column).
  2. `Category.order(:name)` — the collection of records to build `<option>` tags from, alphabetized so the dropdown doesn't list categories in whatever order they happened to be seeded.
  3. `:id` — the *value* method: for each `Category` in the collection, call `.id` to get what actually gets submitted as `category_id`.
  4. `:name` — the *text* method: call `.name` to get what's displayed to a human inside the dropdown.
  5. `{ prompt: "Choose a category" }` — options for the select itself; `prompt:` inserts a disabled, unselected placeholder option with that text, so the dropdown doesn't silently default to the first category in the list if someone submits without touching it.
  Only after that fifth argument does the ordinary `class: "..."` HTML-attributes hash appear — a `collection_select` call always separates "how to build the options" from "what HTML attributes to put on the `<select>` tag" this way.
- `form.label :price, "Price (EUR)", ...` — here the label text *is* given explicitly ("Price (EUR)"), overriding what `humanize`-from-attribute-name would have produced ("Price"), because the form needs to communicate the currency and the model attribute name has no way to carry that on its own.
- `form.text_field :price, placeholder: "45.00", inputmode: "decimal", ...` — this is the field that calls `Service#price=` on submit, not `price_cents=` — the view genuinely never mentions cents anywhere. `inputmode: "decimal"` is a plain HTML attribute (nothing Rails-specific) that hints mobile keyboards to show a numeric keypad with a decimal point instead of the full alphabetic keyboard.
- `form.submit "Publish", ...` — renders an `<input type="submit">` with the given label; `cursor-pointer` is purely cosmetic, since a submit button is clickable by default but browsers don't always render the pointer cursor on it without being told to.

## The services listing

```erb
<%# app/views/services/index.html.erb %>
<div class="max-w-3xl mx-auto w-full">
  <% if notice = flash[:notice] %>
    <p class="py-2 px-3 bg-green-50 mb-5 text-green-700 font-medium rounded-lg inline-block" id="notice"><%= notice %></p>
  <% end %>

  <div class="flex items-center justify-between">
    <h1 class="text-3xl font-bold text-gray-900">Services near you</h1>
    <% if authenticated? %>
      <%= link_to "Offer a service", new_service_path, class: "rounded-md bg-indigo-600 px-4 py-2 text-white font-medium hover:bg-indigo-700" %>
    <% end %>
  </div>

  <% if @services.none? %>
    <p class="mt-8 text-gray-500">No services yet — be the first to offer one.</p>
  <% else %>
    <div class="mt-8 space-y-4">
      <% @services.each do |service| %>
        <div class="rounded-lg border border-gray-200 p-5">
          <div class="flex items-start justify-between gap-4">
            <div>
              <p class="text-xs font-semibold uppercase tracking-wide text-indigo-600"><%= service.category.name %></p>
              <h2 class="mt-1 text-lg font-semibold text-gray-900"><%= service.title %></h2>
              <p class="mt-1 text-sm text-gray-500">by <%= service.user.email_address %></p>
            </div>
            <p class="whitespace-nowrap text-lg font-semibold text-gray-900">
              <%= number_to_currency(service.price) %>
            </p>
          </div>
          <p class="mt-3 text-gray-600"><%= service.description %></p>
        </div>
      <% end %>
    </div>
  <% end %>
</div>
```

- `<% if notice = flash[:notice] %> ... <% end %>` — this is a single `=` on purpose, an assignment used as a condition, not a `==` comparison. `flash[:notice]` is read once, assigned to a local `notice`, and that same local is reused inside the block — this is the exact pattern already used in `sessions/new.html.erb` from episode 2, kept consistent here rather than introducing a different idiom. It's what makes the "Your service is live." message from the controller's `redirect_to ..., notice: "..."` actually show up on screen — nothing renders `flash` automatically anywhere in this layout, each view that wants to show it does so explicitly.
- `<% if authenticated? %> ... <% end %>` around the "Offer a service" button — a signed-out visitor browsing the public index sees the listings but not an invitation to post one; the button only appears once someone is actually signed in.
- `<% if @services.none? %> ... <% else %> ... <% end %>` — `.none?` is a plain ActiveRecord/Enumerable query, true when the relation has zero records. This is the empty-state message, shown instead of an empty page with no explanation the first time the app runs with no listings yet.
- `<% @services.each do |service| %>` — iterates the eager-loaded relation from the controller. Because `Service.includes(:category, :user)` already pulled every category and user into memory alongside the services themselves, `service.category.name` and `service.user.email_address` inside this loop don't trigger any additional queries — this is the payoff of the `includes` call discussed in the controller section landing here, in the view, where the N+1 would otherwise actually happen.
- `service.category.name`, `service.title`, `service.user.email_address`, `service.description` — straightforward attribute and association reads.
- `number_to_currency(service.price)` — a Rails view helper (from `ActionView::Helpers::NumberHelper`) that formats a plain number as currency: `42.5` becomes `"€42.50"` — the correct two decimal places and currency symbol, not whatever number of digits the float happens to have. This calls `service.price`, the euros-facing virtual reader defined on the model — not `service.price_cents` — so the number on screen is genuinely `42.5`, formatted, never `4250`.

Left completely unconfigured, `number_to_currency` defaults to US dollars — VicinoTe's currency is euros, so that default needed overriding once, globally, rather than passing `unit: "€"` at every call site:

```yaml
# config/locales/en.yml
en:
  number:
    currency:
      format:
        unit: "€"
        format: "%u%n"
```

`unit:` is the symbol itself. `format:` is the template that arranges it relative to the number — `%u` is the unit, `%n` is the formatted number, so `"%u%n"` means "symbol, then number, no space," which is what produces `€42.50` rather than `€ 42.50` or `42.50€`. Both `number_to_currency` calls in the app — there's only the one, in `index.html.erb` — pick this up automatically with no argument changes needed, because the default itself moved, not the call site.

## A views-only gotcha: the error said "price cents"

Submitting the form empty for the first time, before fixing this, produced:

```
Category must exist
Title can't be blank
Description can't be blank
Price cents can't be blank
Price cents is not a number
```

Everything else reads naturally — "Title can't be blank" — because Rails derives the human-readable label from the attribute name via `humanize`. But the attribute actually being validated is `price_cents`, not `price`, so that's the name that leaked into the message. A user filling out this form has never heard of `price_cents`; the form field right below the error just says "Price (EUR)".

The fix is a one-line I18n override, not a code change to the model — the validation is correct, only its rendered name was wrong:

```yaml
# config/locales/en.yml
en:
  activerecord:
    attributes:
      service:
        price_cents: "Price"
```

This is Rails' I18n lookup for `human_attribute_name`: before falling back to auto-`humanize`-ing an attribute name, ActiveRecord checks `activerecord.attributes.<model>.<attribute>` in the locale files, and uses whatever string is found there verbatim, for every message that mentions that attribute — validation errors, `form.label` with no explicit text, anywhere.

One detail that cost a second pass: writing the override as `price` (lowercase) produced "price can't be blank" — lowercase, inconsistent with "Title" and "Description" right next to it. Rails' default `humanize` capitalizes the first letter automatically as part of what it does; a custom I18n string is used exactly as written, with no capitalization applied on top of it. The fix was just capitalizing the override itself, `"Price"`.

## Trying it

```bash
bin/dev
```

Sign up (or sign in), click "Offer a service", fill in a title, pick a category, write a description, and a price like `42.50`. Submit, and it redirects to `/services` with "Your service is live." in a flash banner, the listing right below it — category, title, whoever posted it, description, and `€42.50`, not `4250`. Sign out and visit `/services` directly: still there, still public. Try `/services/new` while signed out and it redirects to sign-in, same as any other protected page. Submit the form with everything blank and every field's own validation message shows up, in plain, correctly-capitalized English, "price_cents" nowhere in sight.

## What's next

Episode 4 builds `Booking` — the record of an agreement between two users, and the flow that actually lets someone book a service that's been listed.
