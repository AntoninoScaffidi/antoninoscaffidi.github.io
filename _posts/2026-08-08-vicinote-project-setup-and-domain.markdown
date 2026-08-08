---
layout: post
title: "VicinoTe: Setting Up a Rails Marketplace and Designing Its Domain"
series: "vicinote"
episode: 1
lang: en
ref: vicinote-project-setup-and-domain
permalink: /vicinote-project-setup-and-domain/
canonical_url: https://antoninoscaffidi.github.io/vicinote-project-setup-and-domain/
date: 2026-08-08 10:00:00 +0200
image: /assets/images/vicinote-banner.png
---

This is the first episode of **VicinoTe**, a series where we build a complete Rails application from an empty directory to something with real features — including, in the later episodes, an AI module powered by RubyLLM.

The name comes from the Italian *vicino a te*, "close to you". VicinoTe is a marketplace for local services: you find someone nearby to fix a tap, teach guitar, walk a dog or renovate a bathroom — or you offer your own skills to the people around you.

In this episode we set up the project and, more importantly, decide what we're actually building. The code is on GitHub, tagged [`episode-1`](https://github.com/AntoninoScaffidi/vicinote-tutorial/tree/episode-1) in the [vicinote-tutorial](https://github.com/AntoninoScaffidi/vicinote-tutorial) repo, which will grow with every post.

## Why a marketplace

Tutorials often build a blog or a to-do list. Those are fine for learning syntax, but they're too simple to run into the problems that make Rails interesting: the same record playing different roles depending on context, money, availability, permissions, search that has to actually be good.

A services marketplace hits all of them, and it does so gradually — you can have something working after two episodes and still have plenty left to build after ten.

## Creating the app

```bash
rails new vicinote-tutorial -d postgresql --css tailwind
```

Three decisions are baked into that line, so let's be explicit about them.

**`-d postgresql`.** Rails 8 defaults to SQLite, which is genuinely a good default now. I'm choosing PostgreSQL anyway for one specific reason: the AI episodes later in this series need **vector search** to power semantic search over services. The standard way to do that in Postgres is the `pgvector` extension, and having it available without migrating the database mid-series is worth the small extra setup now.

**`--css tailwind`.** Personal preference, and it keeps the views readable without a separate stylesheet to maintain alongside the tutorial. Nothing in the series depends on Tailwind specifically — if you prefer plain CSS, the markup will still make sense.

**Rails 8 defaults we're keeping.** The generator also brings in Propshaft, Importmap, Turbo, Stimulus, and the Solid trio (Solid Queue, Solid Cache, Solid Cable). We're not configuring any of them yet, but they're the reason there's no Redis and no Node build step in this project.

Then create the databases:

```bash
bin/rails db:create
```

If that command fails, PostgreSQL isn't running or isn't reachable — that's the one piece of setup you have to sort out on your own machine before continuing.

## Designing the domain

This is the part worth slowing down on. Getting the model right now saves a lot of painful migrations later.

Here's the shape we're aiming for:

```
User            can offer services AND book them
 ├─ Service     something a user offers (title, description, price)
 │   └─ Category
 └─ Booking     a user books another user's service
     └─ Review  left after the booking is completed
```

### The important decision: one User, two roles

The central question in any marketplace is how to represent the two sides of the transaction. There are three common answers.

**Option A — separate models.** A `Provider` model and a `Customer` model, each with its own table.

This is the first idea most people have, and it's usually wrong. You immediately duplicate everything that isn't role-specific: email, password digest, name, avatar, phone number, address. Then you need authentication that works for both, which means either two login flows or a polymorphic mess. And the moment someone who offers guitar lessons wants to book a plumber, they need two accounts with two passwords — which is absurd, and exactly the situation a neighbourhood marketplace runs into constantly.

**Option B — a `role` column on User.** One `users` table with `role: "provider"` or `role: "customer"`.

Better, but it encodes a false assumption: that being a provider or a customer is a permanent property of a person. It isn't. It's a property of *a particular relationship*. The same person is a provider in the booking where they teach guitar and a customer in the booking where they hire a plumber. A single `role` column can't express that, and you'll end up fighting it.

**Option C — role emerges from the association.** One `User` model, no role column. You're a provider *of the services you created*, and a customer *of the bookings you made*.

This is what we'll use:

```ruby
class User < ApplicationRecord
  has_many :services                                        # things I offer
  has_many :bookings, foreign_key: :customer_id             # things I booked
  has_many :received_bookings, through: :services, source: :bookings
end
```

No duplication, one login, and a user who does both is the normal case rather than an edge case. When we later need "provider-only" behaviour, it's a question about the data (`user.services.any?`), not about a flag we have to keep in sync.

To be fair about the trade-off: this approach makes some queries slightly more involved. "Show me everything happening in my account" has to look at two associations instead of one. In exchange, we never have to answer "what happens when a customer becomes a provider" — because nothing does. That's a good deal.

### Bookings hold the money

A `Booking` isn't just a link between a user and a service. It's the record of an agreement at a point in time: the date, the agreed price, the status. It needs its own price column rather than reading `service.price`, because the service's price can change tomorrow and that must not silently rewrite what someone agreed to pay last week.

This kind of thing is easy to get wrong and painful to fix once real data exists, which is why we're deciding it before writing a single migration.

### Reviews belong to bookings, not to services

It's tempting to attach a `Review` directly to a `Service`. Attaching it to a `Booking` instead gives us something valuable for free: only someone who actually booked and completed a service can review it. The constraint is built into the shape of the data rather than enforced by validation logic we have to remember to write.

## A landing page

To finish with something visible, a static home page:

```bash
bin/rails generate controller Pages home --skip-routes --no-helper --no-assets
```

```ruby
# config/routes.rb
root "pages#home"
```

I used `--skip-routes` because the generator would otherwise add `get "pages/home"`, and we want this at the root instead. `--no-helper` and `--no-assets` just avoid creating files we won't use.

The view itself is plain markup describing the project, with two buttons deliberately left inert — placeholders for the flows we build next.

```bash
bin/dev
```

Open `http://127.0.0.1:3000` and there it is: an empty app that knows what it wants to become.

## What's next

Episode 2 turns the domain sketch into real code: the `User` model, authentication, and the first migrations. From there the marketplace starts taking shape — services, categories, and the flows that connect the two sides of a booking.
