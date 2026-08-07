---
layout: post
title: "Wiring RubyLLM into Rails: A Minimal Chat Form"
series: "ai-with-ruby"
episode: 2
lang: en
ref: wiring-rubyllm-into-rails
permalink: /2026/08/07/wiring-rubyllm-into-rails/
canonical_url: https://antoninoscaffidi.github.io/2026/08/07/wiring-rubyllm-into-rails/
date: 2026-08-07 12:00:00 +0200
---

In [episode 1]({% post_url 2026-08-06-introduction-to-rubyllm %}) we installed RubyLLM and made a single call from plain Ruby. This time we wire it into an actual Rails app: a form where you type a message and get a response back from the model. No database, no conversation history yet — that's episode 3. The goal here is just to see RubyLLM working inside a real Rails request/response cycle.

The full code for this episode is on GitHub, tagged [`episode-2`](https://github.com/AntoninoScaffidi/ai-with-ruby-demo/tree/episode-2), in a new companion repo — [ai-with-ruby-demo](https://github.com/AntoninoScaffidi/ai-with-ruby-demo) — that will grow with each post in this series.

## Setting up the app

A fresh Rails 8 app with Tailwind CSS:

```bash
rails new ai-with-ruby-demo --css tailwind
```

Two gems go in the `Gemfile`:

```ruby
gem "ruby_llm"

group :development do
  gem "dotenv-rails"
end
```

`ruby_llm` is the library itself. `dotenv-rails` loads a `.env` file into `ENV` in development — Rails doesn't do this on its own.

## Keeping the API key out of git

Rails 8's default `.gitignore` already excludes `.env*`, so a `.env` file with your real key never gets committed. We commit an `.env.example` instead, as a template for anyone cloning the repo:

```
OPENAI_API_KEY=sk-your-key-here
```

Copy it locally and fill in your real key:

```bash
cp .env.example .env
```

## Configuring RubyLLM

Files in `config/initializers/` run once, at boot, before any request is handled — the right place to configure a gem like this:

```ruby
# config/initializers/ruby_llm.rb
RubyLLM.configure do |config|
  config.openai_api_key = ENV.fetch("OPENAI_API_KEY", nil)
end
```

## Routes and controller

A `resource` (singular) fits here — there's no list of chats to index or a specific one to fetch by ID, just a single form:

```ruby
# config/routes.rb
resource :chat, only: [:new, :create]
root "chats#new"
```

```ruby
# app/controllers/chats_controller.rb
class ChatsController < ApplicationController
  def new
  end

  def create
    @message = params[:message]
    chat = RubyLLM.chat
    @response = chat.ask(@message).content
    render :new
  end
end
```

`RubyLLM.chat` creates a new, in-memory chat session — nothing persisted anywhere, which is exactly why the conversation disappears on refresh right now. `.ask` sends the message and blocks until the model replies; `.content` is the reply text.

## The view

```erb
<%= form_with url: chat_path, method: :post, data: { turbo: false } do %>
  <textarea name="message"><%= @message %></textarea>
  <button type="submit">Send</button>
<% end %>

<% if @response %>
  <p><%= @response %></p>
<% end %>
```

## The Turbo gotcha

The first version of this form didn't have `data: { turbo: false }`, and submitting it did... nothing visible. The server logs showed a clean `200 OK`, but the page never updated.

The cause: Rails 8 ships with Turbo Drive by default, which intercepts every form submission and sends it as a background request, expecting either a redirect or a response in the Turbo Stream format. Our controller just rendered plain HTML — Turbo received it but didn't know what to do with it, so it silently did nothing.

`data: { turbo: false }` tells Turbo to leave this specific form alone and submit it the old-fashioned way: a real page load. That's the right fix for now. When we get to streaming responses later in this series, we'll actually want Turbo Streams — but for a first "does this work at all" version, disabling it is the simplest path.

## What's next

Episode 3 adds persistence: `Conversation` and `Message` models, so the chat history survives a page refresh instead of living only inside a single request.
