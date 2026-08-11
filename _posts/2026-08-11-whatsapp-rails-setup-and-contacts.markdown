---
layout: post
title: "WhatsApp with Rails: Project Setup and Contacts"
series: "whatsapp-with-rails"
episode: 1
lang: en
ref: whatsapp-rails-setup-and-contacts
permalink: /whatsapp-rails-setup-and-contacts/
canonical_url: https://antoninoscaffidi.github.io/whatsapp-rails-setup-and-contacts/
image: /assets/images/whatsapp-with-rails-banner.png
date: 2026-08-11 09:00:00 +0200
---

This is the first episode of **WhatsApp with Rails**, a short series that came out of real client work: a customer needed WhatsApp messaging integrated into a Rails app, and before building the full thing, I wanted a small, working demo to show them — and to make sure I actually understood the integration myself, not just followed a tutorial.

The scope here is deliberately narrow. No CRM, no campaigns, no message templates, no statistics — that's the real project, and it isn't what this series is about. This series is about one thing: getting a real WhatsApp message sent from a Rails app, understanding what's actually happening when you do. We'll build it two ways — through Twilio (this episode and the next), and then, as a separate episode, calling Meta's WhatsApp Cloud API directly, without Twilio in between, so the two approaches can be compared.

Code is on GitHub, tagged [`episode-1`](https://github.com/AntoninoScaffidi/whatsapp-with-rails/tree/episode-1), in the [whatsapp-with-rails](https://github.com/AntoninoScaffidi/whatsapp-with-rails) repo.

## What we're building, and why Twilio first

Before writing any code, it's worth knowing what Twilio actually is here, because "WhatsApp API" is a slightly confusing phrase — there are two ways to get at it.

**Meta owns WhatsApp**, and Meta does offer a direct API (the WhatsApp Business Cloud API) to send and receive messages programmatically. You *can* integrate with it directly. But Meta's API requires a Meta Business account, app review, phone number registration through Meta's own system, and its authentication and webhook setup are Meta-specific.

**Twilio sits on top of that.** It's a communications platform that already has the Meta integration done, wrapped in a simpler, well-documented REST API (and a Ruby gem) that looks and feels the same whether you're sending WhatsApp, SMS, or a voice call. You still need WhatsApp-approved sender registration either way, but Twilio's sandbox mode lets you start sending test messages in minutes, with no waiting on Meta's approval process.

That's why episode 1 and 2 use Twilio: it's the fastest path to a real message actually arriving on a phone, and it's genuinely a legitimate production choice, not just a shortcut — plenty of real products run their WhatsApp messaging through Twilio permanently. Episode 3 then does the same job through Meta's API directly, so the trade-off (simplicity and speed vs. one less service in the middle) is visible in actual code, not just in the abstract.

## Creating the app

```bash
rails new whatsapp-with-rails -d postgresql --css tailwind
```

Same reasoning as the other series on this blog: PostgreSQL because it's a reasonable default for anything that might grow, Tailwind to keep the views readable without a separate stylesheet. Nothing here is Twilio-specific yet — this command is identical to how you'd start any small Rails app.

```bash
bin/rails db:create
```

## The Contact model

The real project has a full CRM — segments, tags, GDPR consent tracking, import/export. Here we need exactly two fields: who, and what number to message.

```bash
bin/rails generate model Contact name:string whatsapp_number:string
```

Two things worth doing before migrating: making both fields required, and validating the phone number format properly — because WhatsApp messaging *requires* a specific number format, and it's much better to catch a malformed number in a form validation than in a failed API call later.

```ruby
# db/migrate/..._create_contacts.rb
create_table :contacts do |t|
  t.string :name, null: false
  t.string :whatsapp_number, null: false

  t.timestamps
end
```

```ruby
# app/models/contact.rb
class Contact < ApplicationRecord
  validates :name, presence: true
  validates :whatsapp_number, presence: true, format: {
    with: /\A\+[1-9]\d{6,14}\z/,
    message: "must be in E.164 format, e.g. +391234567890"
  }
end
```

### Why E.164, specifically

**E.164** is the ITU's international standard for phone number formatting — the format that guarantees a number is unambiguous anywhere in the world. The shape is `+` followed by the country calling code, followed by the subscriber number, no spaces, no dashes, no parentheses, no leading zero on the local part where the country's dialing plan would normally have one.

A concrete example: an Italian mobile number you'd normally write as `333 1234567` becomes `+393331234567` in E.164 — country code `39`, then the number as-is (Italian mobile numbers don't carry a leading trunk zero to begin with).

Why this matters here specifically: Twilio's WhatsApp API — and the underlying WhatsApp protocol itself — requires E.164. Without a single unambiguous format, `333-1234567` is meaningless out of context: is it missing a country code? Does it need a leading zero stripped? E.164 removes every one of those questions.

The regex mirrors the standard directly: `+`, then a digit `1`–`9` (country codes never start with `0`), then 6 to 14 more digits — matching E.164's actual hard limit of 15 digits total after the `+`.

```bash
bin/rails db:migrate
```

## Routes, controller, contacts list

Just enough REST to list contacts and add one — no edit, no delete, not needed for this demo:

```ruby
# config/routes.rb
resources :contacts, only: [:index, :new, :create]
root "contacts#index"
```

```ruby
# app/controllers/contacts_controller.rb
class ContactsController < ApplicationController
  def index
    @contacts = Contact.order(:name)
  end

  def new
    @contact = Contact.new
  end

  def create
    @contact = Contact.new(contact_params)

    if @contact.save
      redirect_to contacts_path, notice: "Contact added."
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  def contact_params
    params.require(:contact).permit(:name, :whatsapp_number)
  end
end
```

Two things worth calling out, both carried over from lessons in earlier series on this blog: `create` **redirects** after a successful save (the Post/Redirect/Get pattern — see [episode 3 of AI with Ruby]({% post_url 2026-08-09-persisting-conversations-with-activerecord %}) for why that specifically matters for Turbo Drive), and a failed save **renders with `status: :unprocessable_entity`** rather than the default `200 OK` — the correct HTTP status for "the request was understood but the data was invalid," and something Turbo itself checks for when deciding whether to treat a form response as an error.

The views are a plain list and a plain form — nothing new here, so I won't repeat them in full; they're in the repo if you want to see the Tailwind markup.

## Trying it

```bash
bin/dev
```

Open `http://127.0.0.1:3000`: an empty contacts list, an "Add contact" button, a form. Try submitting a number without the `+` — the validation catches it, with the exact error message explaining what's expected.

Nothing here talks to Twilio yet. That's deliberate — episode 2 wires in the `twilio-ruby` gem and actually sends a message to one of these contacts.

## What's next

Episode 2 adds the `twilio-ruby` gem, a form to compose a message, and the API call that sends it — plus what a Twilio WhatsApp sandbox actually is and why you need one before Meta approves your own sender.
