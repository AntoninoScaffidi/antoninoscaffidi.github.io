---
layout: post
title: "VicinoTe: Authentication with Rails 8's Built-In Generator"
series: "vicinote"
episode: 2
lang: en
ref: vicinote-authentication-with-rails-8
permalink: /vicinote-authentication-with-rails-8/
canonical_url: https://antoninoscaffidi.github.io/vicinote-authentication-with-rails-8/
image: /assets/images/vicinote-banner.png
date: 2026-08-13 09:00:00 +0200
---

[Episode 1]({% post_url 2026-08-08-vicinote-project-setup-and-domain %}) ended with a design decision and nothing to log into: one `User` model, no role column, provider and customer both emerging from associations we hadn't written any code for yet. This episode writes that model — and, with it, the whole authentication system around it.

Code is tagged [`episode-2`](https://github.com/AntoninoScaffidi/vicinote-tutorial/tree/episode-2) in the [vicinote-tutorial](https://github.com/AntoninoScaffidi/vicinote-tutorial) repo.

## Not Devise

Every past version of "add authentication to a Rails app" started with `gem "devise"`. Rails 8 changed that: there's now a built-in generator, `bin/rails generate authentication`, that writes plain, ordinary Rails code — models, controllers, a concern — directly into your app. No gem, no engine, no generated code living somewhere in a gem you can't easily read.

That distinction matters more than it sounds. With Devise, understanding "how sign-in actually works" means reading the gem's source. With the Rails 8 generator, the code it writes *is* your code, sitting in `app/` like everything else, ready to be read, modified, and — as we'll see partway through this episode — extended, because the generator deliberately doesn't do everything.

```bash
bin/rails generate authentication
```

Here's the full list of what that one command produced:

```
create  app/views/passwords/new.html.erb
create  app/views/passwords/edit.html.erb
create  app/views/sessions/new.html.erb
create  app/models/session.rb
create  app/models/user.rb
create  app/models/current.rb
create  app/controllers/sessions_controller.rb
create  app/controllers/concerns/authentication.rb
create  app/controllers/passwords_controller.rb
create  app/mailers/passwords_mailer.rb
create  app/views/passwords_mailer/reset.html.erb
create  app/views/passwords_mailer/reset.text.erb
insert  app/controllers/application_controller.rb
 route  resources :passwords, param: :token
 route  resource :session
  gsub  Gemfile
  create  db/migrate/..._create_users.rb
  create  db/migrate/..._create_sessions.rb
```

Worth reading every one of these before writing anything of our own, because — unlike a gem — we're going to be looking directly at this code for the rest of the series.

## The User model

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password
  has_many :sessions, dependent: :destroy

  normalizes :email_address, with: ->(e) { e.strip.downcase }
end
```

`has_secure_password` is the one line doing the heavy lifting, and it's not a Devise-style black box — it's plain Rails, from `ActiveModel::SecurePassword`. It expects a `password_digest` column, adds `password=`/`password_confirmation=` virtual attributes that hash with bcrypt on assignment, and adds an `authenticate` instance method. We'll come back to it below, because it does more than that — the password reset mechanism later in this post comes from the exact same line.

(Worth being precise about one thing: `has_secure_password` itself is Rails code, part of the `activemodel` gem — not part of `bcrypt`. The `bcrypt` gem in the Gemfile only supplies the `BCrypt::Password` class Rails uses internally to actually hash and compare passwords; the macro, the validations, and the reset-token logic all live in Rails itself, readable like any other framework code.)

`normalizes :email_address` is a smaller but genuinely useful Rails feature: it guarantees `"Mario@Example.com "` and `"mario@example.com"` are treated as the same address everywhere — on save, and on every subsequent lookup — without us having to remember to call `.downcase.strip` by hand at every call site.

```ruby
# db/migrate/..._create_users.rb
create_table :users do |t|
  t.string :email_address, null: false
  t.string :password_digest, null: false
  t.timestamps
end
add_index :users, :email_address, unique: true
```

Note what's *not* here: no `name`, no role, nothing marketplace-specific. This is intentionally the minimum viable authenticatable record. Everything from episode 1's domain design — `has_many :services`, `has_many :bookings` — gets added when we actually build `Service` and `Booking`, not now. Adding those associations today would technically work (Rails resolves association class names lazily), but it would be code referring to models that don't exist yet, which is worse for anyone reading this repo top to bottom.

## Sessions and Current: how "who's logged in" is tracked

This is the part that looks the most different from what you'd expect coming from Devise, and it's worth slowing down on.

```ruby
# app/models/session.rb
class Session < ApplicationRecord
  belongs_to :user
end
```

```ruby
# db/migrate/..._create_sessions.rb
create_table :sessions do |t|
  t.references :user, null: false, foreign_key: true
  t.string :ip_address
  t.string :user_agent
  t.timestamps
end
```

A `Session` is a **database row**, not just an encrypted cookie. Every time someone signs in, a new `Session` record gets created, storing which user, from what IP, with what browser. The browser only ever holds the session's *id*, signed inside a cookie so it can't be tampered with — the actual session state lives server-side.

The practical upside: you can see every active session for a user (`user.sessions`), and revoke one individually — sign a specific device out — just by deleting that row. A pure cookie-based session can't do that; you can only invalidate *all* sessions at once (e.g. by rotating a secret).

```ruby
# app/models/current.rb
class Current < ActiveSupport::CurrentAttributes
  attribute :session
  delegate :user, to: :session, allow_nil: true
end
```

`ActiveSupport::CurrentAttributes` is a Rails mechanism for per-request global state — safer than a plain global variable because it's automatically reset between requests (and between test cases), so there's no risk of one request's user leaking into the next. `Current.session` holds the current `Session` record for this request; `Current.user` is just `Current.session.user`, available anywhere in the app via `Current.user`, no need to pass it down through every method call.

## The Authentication concern: secure by default

```ruby
# app/controllers/concerns/authentication.rb
module Authentication
  extend ActiveSupport::Concern

  included do
    before_action :require_authentication
    helper_method :authenticated?
  end

  class_methods do
    def allow_unauthenticated_access(**options)
      skip_before_action :require_authentication, **options
    end
  end

  private
    def resume_session
      Current.session ||= find_session_by_cookie
    end

    def find_session_by_cookie
      Session.find_by(id: cookies.signed[:session_id]) if cookies.signed[:session_id]
    end

    def start_new_session_for(user)
      user.sessions.create!(user_agent: request.user_agent, ip_address: request.remote_ip).tap do |session|
        Current.session = session
        cookies.signed.permanent[:session_id] = { value: session.id, httponly: true, same_site: :lax }
      end
    end
    # ...
end
```

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  include Authentication
  # ...
end
```

The line that matters most here is `before_action :require_authentication` — included into `ApplicationController` itself, which means **every controller in the app requires a signed-in user by default**, unless it explicitly opts out with `allow_unauthenticated_access`. This is the opposite of how most tutorials build auth (where you protect specific actions), and it's a deliberate, sensible default for an app whose whole point is accounts: it's much safer to have to remember to make a page public than to have to remember to protect it.

The signed cookie itself is worth reading closely too: `cookies.signed.permanent[...]` — signed, so the value can't be forged (Rails verifies it against a secret before trusting it); `permanent`, so it's set to expire 20 years out rather than at the end of the browser session; `httponly: true`, so client-side JavaScript can never read it (closing off a whole category of session-theft via XSS); `same_site: :lax`, a baseline CSRF protection that stops the cookie from being sent on cross-site requests except top-level navigation.

### A gotcha: our public landing page just broke

Because `require_authentication` is now the default everywhere, episode 1's landing page — meant to be the first thing anyone sees, logged in or not — would redirect to the sign-in page the moment we ran the generator. `PagesController` needed an explicit opt-out:

```ruby
# app/controllers/pages_controller.rb
class PagesController < ApplicationController
  allow_unauthenticated_access

  def home
  end
end
```

This is exactly the trade-off "secure by default" makes on purpose: you *will* hit this the first time you add the generator to an app with existing public pages, and the fix is one line, but you have to know to look for it.

## Signing in: `authenticate_by` and rate limiting

```ruby
# app/controllers/sessions_controller.rb
class SessionsController < ApplicationController
  allow_unauthenticated_access only: %i[ new create ]
  rate_limit to: 10, within: 3.minutes, only: :create, with: -> { redirect_to new_session_path, alert: "Try again later." }

  def create
    if user = User.authenticate_by(params.permit(:email_address, :password))
      start_new_session_for user
      redirect_to after_authentication_url
    else
      redirect_to new_session_path, alert: "Try another email address or password."
    end
  end
  # ...
end
```

Two details here are genuinely worth understanding, not just copying.

**`User.authenticate_by`** looks like it's just `find_by(email_address:) + authenticate(password)` in one call, but it's specifically designed to close a timing attack. Straight from the Rails source comment:

> Regardless of whether a record is found, `authenticate_by` will cryptographically digest the given password attributes. This behavior helps mitigate timing-based enumeration attacks, wherein an attacker can determine if a passworded record exists even without knowing the password.

Concretely: if you looked up the user first and only ran the bcrypt comparison when one was found, a request for a *nonexistent* email would return almost instantly (no bcrypt work), while a request for a *real* email with the wrong password would take the ~100ms bcrypt takes. That timing difference is enough for an attacker to enumerate which emails have accounts, purely by measuring response times. `authenticate_by` always does the expensive digest work, found or not, so both cases take the same time.

**`rate_limit`** is a native Rails 8 feature (`ActionController::RateLimiting`, no gem) — ten attempts per three minutes on this action, backed by `Rails.cache`, redirecting with a message instead of just failing silently. Brute-forcing a password by trying thousands of combinations is meaningfully slowed down by this one line.

## The missing piece: sign-up

Try the generator's routes and you'll find `new_session_path` and `new_password_path` (reset), but nothing to create a `User` in the first place. That's not an oversight — the generator can't guess your app's sign-up requirements (invite-only? email confirmation? OAuth?) so it leaves that entirely to you.

For VicinoTe, anyone should be able to sign up — it's a marketplace, not an admin tool — so we add it ourselves, following the same shape as the generated code:

```ruby
# app/controllers/registrations_controller.rb
class RegistrationsController < ApplicationController
  allow_unauthenticated_access

  def new
    @user = User.new
  end

  def create
    @user = User.new(user_params)

    if @user.save
      start_new_session_for @user
      redirect_to root_path, notice: "Welcome to VicinoTe!"
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  def user_params
    params.require(:user).permit(:email_address, :password, :password_confirmation)
  end
end
```

```ruby
# config/routes.rb
resource :registration, only: [:new, :create]
```

Nothing here is new machinery — `start_new_session_for` is the same private method `SessionsController` uses, callable because it's defined in the `Authentication` concern that's mixed into every controller. Sign-up, in this design, is just "create a `User`, then do exactly what signing in does."

## Password reset without a reset-token column

This is the detail I found most worth digging into. `PasswordsController` calls `User.find_by_password_reset_token!(token)` — a method that doesn't exist anywhere in `user.rb`. It isn't hand-written, and there's no `reset_password_token` column in the `users` table either. So where does it come from?

It comes from `has_secure_password` itself. Reading the Rails source (`ActiveModel::SecurePassword`):

```ruby
def has_secure_password(attribute = :password, validations: true, reset_token: true)
  # ...
  if reset_token && respond_to?(:generates_token_for)
    generates_token_for :"#{attribute}_reset", expires_in: 15.minutes do
      public_send(:"#{attribute}_salt")&.last(10)
    end
    # defines find_by_password_reset_token / find_by_password_reset_token!
  end
end
```

`generates_token_for` is Rails' general-purpose mechanism for generating a verifiable, expiring token *without storing it anywhere* — it's a signed, encoded payload (via `ActiveSupport::MessageVerifier`), checked and decoded on the way back in. Here, `has_secure_password` uses it automatically, keyed off the last 10 characters of the password's bcrypt salt.

That detail is the clever part: the salt changes every time the password changes (bcrypt generates a fresh one on every hash), which means **a reset link is automatically invalidated the instant the password it was generated for actually changes** — with no extra "used" flag, no token cleanup job, nothing to store. The token expires after 15 minutes either way, and `find_by_password_reset_token!` raises `ActiveSupport::MessageVerifier::InvalidSignature` on an expired or tampered token, which `PasswordsController` rescues into a friendly redirect.

We're not building the full flow out today (that needs a real mailer setup, which will make more sense once VicinoTe is deployed somewhere), but it's worth knowing this exists, fully wired, the moment `has_secure_password` is on the model — one line, and a working, secure password-reset mechanism came with it.

## A small nav bar, so any of this is reachable

None of the above has a UI to get to unless something links to it, so the layout gets a minimal header:

```erb
<% if authenticated? %>
  <span><%= Current.user.email_address %></span>
  <%= button_to "Sign out", session_path, method: :delete %>
<% else %>
  <%= link_to "Sign in", new_session_path %>
  <%= link_to "Sign up", new_registration_path %>
<% end %>
```

`authenticated?` is the `helper_method` the `Authentication` concern exposes — it's just `resume_session`, reused to answer "is anyone signed in" without redirecting.

## Trying it

```bash
bin/dev
```

Sign up with any email and password — you land back on the homepage, already signed in, your email in the top bar. Sign out, then sign back in; try a wrong password and you'll get the friendly error rather than a stack trace. None of this touches `Service` or `Booking` yet — the point of this episode is that accounts work, end to end, before anything gets built on top of them.

## What's next

Episode 3 builds `Service` and `Category`, and finally wires up the `has_many :services` association from episode 1's design — the moment a signed-in user can actually list something they offer.
