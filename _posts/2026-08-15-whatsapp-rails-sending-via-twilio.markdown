---
layout: post
title: "WhatsApp with Rails: Sending a Real Message via Twilio"
series: "whatsapp-with-rails"
episode: 2
lang: en
ref: whatsapp-rails-sending-via-twilio
permalink: /whatsapp-rails-sending-via-twilio/
canonical_url: https://antoninoscaffidi.github.io/whatsapp-rails-sending-via-twilio/
image: /assets/images/whatsapp-with-rails-banner.png
date: 2026-08-15 09:00:00 +0200
---

[Episode 1]({% post_url 2026-08-11-whatsapp-rails-setup-and-contacts %}) got us a `Contact` model and a form to add people. Nothing talked to Twilio yet. This episode closes that gap end to end: a form to compose a message, a real API call, and a message that actually lands on a real phone over WhatsApp — which I tested for real while writing this post, including the ways it fails.

Code is tagged [`episode-2`](https://github.com/AntoninoScaffidi/whatsapp-with-rails/tree/episode-2) in the [whatsapp-with-rails](https://github.com/AntoninoScaffidi/whatsapp-with-rails) repo. This is a long one — the goal is to leave nothing unexplained: every gem, every line of every migration, every line of the model and controller, and the exact errors you'll hit and why.

## What the WhatsApp Sandbox actually is, and why you need it

Sending a WhatsApp message through the real WhatsApp Business Platform requires a phone number registered and approved through Meta — a process with review steps and waiting time. Twilio's **Sandbox** exists so you don't have to go through that just to write and test code. It's a shared, pre-approved Twilio number (`+14155238886` for everyone using Twilio's sandbox) that can send and receive WhatsApp messages immediately, with one restriction: it will only talk to phone numbers that have explicitly *joined* it.

Joining means sending a specific message — `join <two-word-code>`, e.g. `join vowel-purpose`, a code Twilio generates per account — from WhatsApp, from the phone number you want to test with, to that sandbox number. You can do this by hand from WhatsApp, or by opening the link/QR code Twilio's console shows you (Console → Messaging → Try it out → Send a WhatsApp message). Once a number joins, Twilio can send it messages via the API; a number that never joined will reject them, every time, with an error we'll walk through further down.

Two details worth knowing, because you'll run into both eventually:

- **The join lasts 3 days of inactivity**, not forever. If nobody sends anything to the sandbox from that number for 3 days, it has to `join` again.
- **This is strictly a testing mechanism.** In production you send from a WhatsApp-enabled sender you own (still set up through Twilio, but without the "only pre-joined numbers" restriction) — the sandbox is for development, not for talking to real customers.

## The two gems

```ruby
# Gemfile
gem "twilio-ruby"
```

```ruby
group :development do
  gem "web-console"
  gem "dotenv-rails"
end
```

`twilio-ruby` is Twilio's official Ruby client — a wrapper around Twilio's REST API, giving you `Twilio::REST::Client.new(...).messages.create(...)` instead of hand-building HTTP requests and parsing JSON. `dotenv-rails`, same as in the [ai-with-ruby series]({% post_url 2026-08-07-wiring-rubyllm-into-rails %}), loads a `.env` file into `ENV` in development, so credentials live outside the codebase.

```bash
bundle install
```

## Credentials: `.env`, `.env.example`, and `.gitignore`

Three real values are needed, all secrets: your Twilio Account SID, your Auth Token, and the sandbox WhatsApp number. Rails 8's default `.gitignore` already excludes `.env*`, so a real `.env` file never gets committed. We commit a template instead:

```
# .env.example
TWILIO_ACCOUNT_SID=ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
TWILIO_AUTH_TOKEN=your-auth-token-here
TWILIO_WHATSAPP_NUMBER=+14155238886
```

Because `.gitignore`'s `/.env*` pattern is broad enough to also catch `.env.example`, it needs an explicit exception — this exact gotcha showed up already in the ai-with-ruby series:

```
# .gitignore
/.env*
!/.env.example
```

To actually run this app, copy the template and fill in real values:

```bash
cp .env.example .env
```

**On the Account SID and Auth Token specifically**: these are your Twilio account's master credentials — anyone with them can send messages (and be billed) as you. They're found on the main Twilio Console dashboard the moment you log in. If you ever paste one somewhere public by accident, Twilio lets you regenerate the Auth Token from the console; the old one stops working immediately.

## The Twilio client: one initializer, one line

```ruby
# config/initializers/twilio.rb
Rails.application.config.x.twilio_client = Twilio::REST::Client.new(
  ENV.fetch("TWILIO_ACCOUNT_SID"),
  ENV.fetch("TWILIO_AUTH_TOKEN")
)
```

Same reasoning as the RubyLLM initializer in the AI with Ruby series: files in `config/initializers/` run once, at boot, before any request — the right place to construct something that talks to a third-party API and hand it your credentials.

Two things worth being precise about here. First, `ENV.fetch("TWILIO_ACCOUNT_SID")` — no default given as a second argument — **raises** if the variable is missing, rather than silently continuing with `nil`. That's deliberate: a misconfigured Twilio client that quietly does nothing is much harder to debug than an app that refuses to boot with a clear `KeyError`. Second, `Rails.application.config.x` is Rails' built-in namespace for custom application configuration — the `x` stands for "custom", and it exists specifically so you're not tempted to stash app-specific config in global constants or `Rails.application.config` directly (which Rails itself uses). Anywhere in the app, `Rails.application.config.x.twilio_client` gets you the same configured client.

## The Message model, and the migration behind it

```bash
bin/rails generate model Message contact:references body:text twilio_sid:string status:string
```

The generator produced a migration that I then edited before running it — worth going through both versions, because the edit is where the real thinking is:

```ruby
# db/migrate/..._create_messages.rb — as generated
create_table :messages do |t|
  t.references :contact, null: false, foreign_key: true
  t.text :body
  t.string :twilio_sid
  t.string :status

  t.timestamps
end
```

```ruby
# db/migrate/..._create_messages.rb — as run
create_table :messages do |t|
  t.references :contact, null: false, foreign_key: true
  t.text :body, null: false
  t.string :twilio_sid
  t.string :status, null: false, default: "queued"

  t.timestamps
end
```

Going column by column:

- **`t.references :contact, null: false, foreign_key: true`** — this is what the `contact:references` argument to the generator produces on its own: an integer `contact_id` column, a database-level index on it, a database-level foreign key constraint (`foreign_key: true`) so the database itself refuses to let a `Message` point at a `Contact` that doesn't exist, and `null: false` because a message with no contact makes no sense.
- **`t.text :body, null: false`** — `text` rather than `string` because a message body has no natural short length limit the way a name does; I added `null: false` by hand because the generator doesn't add presence constraints on its own, and an empty message is never something we want to send.
- **`t.string :twilio_sid`** — deliberately nullable. This gets filled in *after* Twilio accepts the message (more on this below); before that point, there's a real `Message` row with no SID yet.
- **`t.string :status, null: false, default: "queued"`** — the one substantive change. I added `null: false, default: "queued"` so every `Message` has a sensible status the instant it's created, before we've even talked to Twilio. Why this column exists at all is the more interesting question, covered in the next section.

```ruby
# app/models/message.rb
class Message < ApplicationRecord
  belongs_to :contact

  validates :body, presence: true

  def deliver!
    twilio_message = Rails.application.config.x.twilio_client.messages.create(
      from: "whatsapp:#{ENV.fetch('TWILIO_WHATSAPP_NUMBER')}",
      to: "whatsapp:#{contact.whatsapp_number}",
      body: body
    )

    update!(twilio_sid: twilio_message.sid, status: twilio_message.status)
  end
end
```

`belongs_to :contact` is enough on its own to require a contact — Rails 5+ makes `belongs_to` associations required by default, so this line alone gives us most of what `null: false` in the migration was already doing, at the application level.

`validates :body, presence: true` mirrors the `null: false` on `body` — one is the database refusing invalid data no matter what inserts it, the other is a friendly validation error before we even try, which is what lets `MessagesController` show "can't be blank" in the form instead of a database-level crash.

`deliver!` is the one method actually doing something. Look closely at `from:` and `to:` — WhatsApp numbers in the Twilio API are always prefixed with the literal string `whatsapp:`, e.g. `whatsapp:+14155238886`. This is how Twilio's unified messaging API tells the difference between sending the same number as a WhatsApp message versus a plain SMS — the prefix, not a separate endpoint. Forget it on either side and the call either errors or silently tries to send a regular text message instead.

The naming convention — `deliver!`, with a bang — mirrors Rails' own convention for methods that do something with real, possibly-failing side effects (`save!`, `create!`), as opposed to a quiet query. Sending a WhatsApp message is about as real a side effect as a method can have.

## Why `twilio_sid` and `status` exist: WhatsApp sending is asynchronous

This is worth being explicit about, because it's easy to assume a successful API call means the message arrived — it doesn't. When `messages.create` returns without raising, all it means is: *Twilio accepted the request and queued it for delivery.* The `status` Twilio returns at that point is typically `"queued"`, not `"delivered"`. What actually happens to the message after that — sent, delivered, read, or failed — happens asynchronously, and by default your Rails app never hears about it again unless it asks.

That's the whole reason `twilio_sid` and `status` are columns on `Message` rather than being thrown away after the API call: `twilio_sid` is the identifier Twilio gives back (`SM...`), and it's how you look the message up again later to check what actually happened to it. I did exactly that while testing this episode:

```ruby
fresh = Rails.application.config.x.twilio_client.messages(message.twilio_sid).fetch
fresh.status        # => "failed" — updated after the fact, not from the original response
fresh.error_code     # => 63015
```

The *right* way to keep `status` current without polling by hand is a **status callback webhook** — a URL you give Twilio that it POSTs to every time a message's status changes. That's genuinely more machinery (a public endpoint, a route, request verification) than belongs in an episode about the first successful send, so it's out of scope here — but it's worth knowing `status` exists on this model specifically because that future webhook will have something to update.

## Wiring it into the app: routes, controller, contact list

```ruby
# config/routes.rb
resources :contacts, only: [:index, :new, :create] do
  resources :messages, only: [:new, :create]
end
```

Nested resources here aren't decoration — `messages` genuinely doesn't make sense without a `contact` in context; there's no "compose a message" screen that isn't already about a specific person. This generates paths like `new_contact_message_path(contact)` and `contact_messages_path(contact)`, both carrying the contact's id.

```ruby
# app/controllers/messages_controller.rb
class MessagesController < ApplicationController
  before_action :set_contact

  def new
    @message = @contact.messages.new
  end

  def create
    @message = @contact.messages.new(message_params)

    if @message.save
      begin
        @message.deliver!
        redirect_to contacts_path, notice: "Message sent to #{@contact.name}."
      rescue Twilio::REST::RestError => e
        redirect_to contacts_path, alert: "Twilio couldn't send the message: #{e.error_message}"
      end
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  def set_contact
    @contact = Contact.find(params[:contact_id])
  end

  def message_params
    params.require(:message).permit(:body)
  end
end
```

`before_action :set_contact` runs before both actions, loading `@contact` from `params[:contact_id]` — the nested route parameter, not `:id` (that would be the message's own id, which doesn't exist yet for `new`/`create`).

`create` does three things in order: build the `Message` (not yet saved), save it to the database, *then* attempt delivery. This ordering matters — the message exists as a database row, with `status: "queued"` from the column default, even before we know whether Twilio will accept it. If the process crashed between save and deliver, there'd still be a record of an intended message, not silence.

## The synchronous failure case: `Twilio::REST::RestError`

Here's something I only found by actually breaking it while testing: `deliver!` can fail in two completely different ways, and only one of them is visible where you'd expect.

**Asynchronous failure** — the number never joined the sandbox, the message gets rejected somewhere downstream — doesn't raise anything in Rails at all. `messages.create` returns successfully with `status: "queued"`; the failure only shows up later if you go back and check, exactly as described above. I hit this directly: sending to a contact whose number had never sent `join <code>` to the sandbox came back as a normal, non-raising `queued` response from the app's point of view, and only turned out to be `status: "failed"`, `error_code: 63015` when I fetched the message back from Twilio afterward. **Error 63015, specifically, means "this recipient hasn't joined this sandbox"** — the single most common thing you'll hit while testing this integration, and worth recognizing on sight.

**Synchronous failure** is different: bad credentials, a malformed request, anything Twilio's API rejects immediately. That *does* raise, as `Twilio::REST::RestError` — I confirmed this directly, deliberately constructing a client with a wrong Account SID and Auth Token and watching it raise on the API call:

```ruby
bad_client = Twilio::REST::Client.new("AC0000000000000000000000000000000", "wrongtoken")
bad_client.messages.create(from: "whatsapp:+14155238886", to: "whatsapp:+391234567890", body: "test")
# raises Twilio::REST::RestError
# e.status_code    => 401
# e.error_message  => "Authentication Error - invalid username"
```

Before adding the `rescue` you see in the controller above, this class of error would have bubbled straight up through `deliver!`, through `create`, into an unhandled `500`, in what's meant to be a demo shown to a client. `rescue Twilio::REST::RestError => e` catches it and redirects with a readable `alert` instead — `e.error_message` is the human-readable string Twilio itself sends back, so the message shown is Twilio's own explanation, not a guess on our part.

What this episode does *not* do is turn the asynchronous 63015 case into a nice UI message — that's not possible from inside `create` at all, since the app has already gotten a "successful", `queued` response by the time the real failure happens. Handling that properly means the status callback webhook mentioned above; noting it here so it's clear this is a known, deliberate gap, not an oversight.

## Trying it

```bash
bin/dev
```

Add a contact with a WhatsApp number that has actually joined your sandbox (see the join instructions earlier in this post — a number that hasn't joined will accept the send from the app's perspective and then fail invisibly, exactly as described above). Click "Message" next to them, write something, send it.

Here's the real result from testing this for the post — an actual message, sent from this actual code, received on an actual phone:

> **Twilio:** Test from whatsapp-with-rails episode 2 🎉

If you try sending to a contact whose number never joined the sandbox, you'll get `queued` in the app and nothing on the phone — that's error 63015 waiting to be discovered if you check the message status afterward, not a bug in this code.

## What's next

Episode 3 does the same job a different way: calling Meta's WhatsApp Cloud API directly, with no Twilio in between, so the trade-off between the two approaches — Twilio's simplicity versus one less service in the middle — shows up in actual code, not just in the abstract.
