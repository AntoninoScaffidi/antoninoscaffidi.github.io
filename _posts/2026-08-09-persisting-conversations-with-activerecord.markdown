---
layout: post
title: "Persisting RubyLLM Conversations with ActiveRecord"
series: "ai-with-ruby"
episode: 3
lang: en
ref: persisting-conversations-with-activerecord
permalink: /persisting-conversations-with-activerecord/
canonical_url: https://antoninoscaffidi.github.io/persisting-conversations-with-activerecord/
date: 2026-08-09 06:00:00 +0200
image: /assets/images/ai-with-ruby-banner.png
---

In [episode 2]({% post_url 2026-08-07-wiring-rubyllm-into-rails %}) we got RubyLLM talking to Rails through a plain form: type a message, get a reply, refresh the page and it's gone. Every `RubyLLM.chat` call created a brand new, in-memory session that lived only for the duration of that one request.

This episode fixes that. By the end, conversations survive a page refresh, live in the database, and the model actually remembers what was said earlier in the same conversation — because we're sending it the real history, not just the latest message.

The code is tagged [`episode-3`](https://github.com/AntoninoScaffidi/ai-with-ruby-demo/tree/episode-3) in the [ai-with-ruby-demo](https://github.com/AntoninoScaffidi/ai-with-ruby-demo) repo.

## Two ways to persist a chat

You could write `Conversation` and `Message` models by hand: a `has_many`/`belongs_to` pair, a controller that appends to an array of messages, some JSON serialization to store what the model said. It would work, but you'd be rebuilding something RubyLLM already ships as a first-class Rails integration, complete with a generator, migrations, and the `acts_as_chat` / `acts_as_message` pattern you'll see in RubyLLM's own documentation and examples.

We're using the generator. Not because doing it by hand is wrong, but because the generated code *is* the idiomatic way to use this gem in Rails — reading it is as instructive as writing it yourself, and you end up with code that matches what you'll find in RubyLLM's docs and in other people's projects.

## Running the generator

```bash
bin/rails generate ruby_llm:install chat:Conversation --skip-active-storage
```

Two things are non-default here, so let's be explicit about both.

**`chat:Conversation`.** Left alone, the generator names the model that represents a chat session `Chat`. That's a reasonable default, but "conversation" is the word we'll actually use when we talk about this feature, so we tell the generator to call it `Conversation` instead. The generator's own usage line spells out the syntax: `bin/rails g ruby_llm:install [chat:ChatName] [message:MessageName] ...` — you can rename any of the models it creates the same way.

**`--skip-active-storage`.** By default the generator also runs `active_storage:install` and adds `has_many_attached :attachments` to the message model, so a chat can have file uploads attached to it. We're not building file attachments in this episode, so we skip it. Nothing stops you from adding it later — the flag only skips it *now*.

## What the generator actually creates

This is worth going through file by file, because it creates more than a `Conversation` and a `Message` — and it's better to know why than to have unexplained files sitting in the app.

```
create  db/migrate/..._create_conversations.rb
create  db/migrate/..._create_messages.rb
create  db/migrate/..._create_tool_calls.rb
create  db/migrate/..._create_models.rb
create  db/migrate/..._add_references_to_conversations_tool_calls_and_messages.rb
create  app/models/conversation.rb
create  app/models/message.rb
create  app/models/tool_call.rb
create  app/models/model.rb
force   config/initializers/ruby_llm.rb
create  app/agents/.gitkeep
create  app/tools/.gitkeep
create  app/schemas/.gitkeep
create  app/prompts/.gitkeep
```

**`conversations` table.** Just an empty shell plus timestamps — a conversation on its own doesn't need to store anything, it just groups messages together. A `model_id` reference gets added later by the fifth migration, recording which LLM model that conversation is using.

**`messages` table.** This is where the substance lives: `role` (`"user"`, `"assistant"`, or `"system"`), `content`, and `content_raw` (a JSON column holding the full raw structure, needed for messages that aren't plain text — like tool calls). There are also columns for extended thinking (`thinking_text`, `thinking_signature`, `thinking_tokens` — for models that expose their reasoning process) and token accounting (`input_tokens`, `output_tokens`, `cached_tokens`, `cache_creation_tokens`). We won't touch the thinking or token columns in this episode, but it's worth knowing they're there: this same table is built to support features several episodes away.

**`tool_calls` table.** Not used yet — this is for a later episode on tool calling, when the model will be able to call into our own application code. The table stores which tool was called (`name`), with what (`arguments`, as JSON), and links back to the message that triggered it.

**`models` table.** A local cache of model metadata — pricing, context window size, which capabilities a given model supports. It's populated by a separate rake task (`bin/rails ruby_llm:load_models`) that we're not running in this episode, since nothing here depends on it yet. The `model_id` reference added to `conversations` and `messages` is optional (`foreign_key: true`, no `null: false`), so everything works fine without it.

**The generated models:**

```ruby
# app/models/conversation.rb
class Conversation < ApplicationRecord
  acts_as_chat
end
```

```ruby
# app/models/message.rb
class Message < ApplicationRecord
  acts_as_message chat: :conversation
end
```

`acts_as_chat` and `acts_as_message` are class methods RubyLLM adds to `ActiveRecord::Base`. They set up the `has_many`/`belongs_to` association between the two models and mix in the methods that make a `Conversation` behave like a RubyLLM chat session — most importantly, `.ask`. Because we renamed the model to `Conversation`, `acts_as_message` needed to know the association isn't called `chat` anymore, hence `chat: :conversation`. The generator worked this out on its own from the `chat:Conversation` mapping we passed it.

**The convention directories** (`app/agents`, `app/tools`, `app/schemas`, `app/prompts`, each with just a `.gitkeep` for now) are empty scaffolding for features later in this series — tools an agent can call, structured output schemas, reusable prompts. Nothing goes in them yet.

## A gotcha: the deprecated legacy API

Here's something that isn't obvious from the README alone, and that we ran into directly. RubyLLM ships *two* implementations of the ActiveRecord integration: the current one (what we just described) and a legacy one, kept only for backward compatibility. Which one loads is controlled by a config flag, `use_new_acts_as`.

Without it, RubyLLM silently loads the legacy implementation and prints this at boot:

```
!!! RubyLLM's legacy acts_as API is deprecated and will be removed in RubyLLM 2.0.0.
Please consult the migration guide at https://rubyllm.com/upgrading-to-1-7/
```

This is exactly the warning we saw back in episode 2, before we'd even touched persistence — it's printed at boot as soon as ActiveRecord loads, regardless of whether you're using `acts_as_chat` yet. The generator knows about this and sets the flag correctly on its own:

```ruby
# config/initializers/ruby_llm.rb
RubyLLM.configure do |config|
  config.openai_api_key = ENV.fetch("OPENAI_API_KEY", nil)

  # Use the current, association-based acts_as API. Without this, RubyLLM
  # silently falls back to a deprecated implementation and warns about it.
  config.use_new_acts_as = true
end
```

If you ever add RubyLLM's ActiveRecord integration to a project *without* using the generator, this is the one line most worth copying by hand — it's easy to miss, and the two implementations aren't identical, so code written against one doesn't necessarily work against the other.

## Running the migrations

```bash
bin/rails db:migrate
```

This creates all four tables described above, plus the fifth migration that wires up the foreign keys between them.

## Trying it from the console, before touching the controller

Worth doing once, to see persistence working with nothing else in the way:

```bash
bin/rails runner '
conversation = Conversation.create!
response = conversation.ask "Reply with exactly one word: pong"
puts "RESPONSE: #{response.content}"
puts "MESSAGES IN DB: #{conversation.messages.count}"
puts "ROLES: #{conversation.messages.pluck(:role).join(%q{, })}"
'
```

```
RESPONSE: pong
MESSAGES IN DB: 2
ROLES: user, assistant
```

One `.ask` call, two rows written: the user's message and the model's reply, both persisted automatically. No deprecation warning either — the flag is doing its job.

## Wiring it into the controller

Episode 2's controller created a fresh, throwaway chat on every request. Now we need to find *the same* conversation across requests. For a single-user demo with no accounts yet, the simplest thing that's still correct is to keep the conversation's id in the Rails session — a small, signed cookie tied to the visitor's browser.

```ruby
# app/controllers/chats_controller.rb
class ChatsController < ApplicationController
  def new
    @conversation = current_conversation
  end

  def create
    current_conversation.ask(params[:message])
    redirect_to new_chat_path
  end

  private

  def current_conversation
    Conversation.find_by(id: session[:conversation_id]) || create_conversation
  end

  def create_conversation
    conversation = Conversation.create!
    session[:conversation_id] = conversation.id
    conversation
  end
end
```

`current_conversation` looks up the conversation by the id stored in the session; if there isn't one yet (first visit, or the session expired), it creates one and remembers its id for next time. `create` no longer renders anything itself — it asks the question, then redirects back to `new`, which is now responsible for loading and displaying the conversation, messages included.

## Showing the history

```erb
<div class="max-w-xl mx-auto mt-16 px-4">
  <h1 class="text-2xl font-semibold mb-6">RubyLLM chat demo</h1>

  <div class="space-y-4 mb-8">
    <% @conversation.messages.each do |message| %>
      <% is_user = message.role == "user" %>
      <div class="<%= is_user ? "text-right" : "text-left" %>">
        <p class="text-xs text-gray-500 mb-1"><%= message.role %></p>
        <p class="inline-block rounded-md px-3 py-2 <%= is_user ? "bg-indigo-600 text-white" : "bg-gray-100" %>">
          <%= message.content %>
        </p>
      </div>
    <% end %>
  </div>

  <%= form_with url: chat_path, method: :post, class: "flex flex-col gap-3" do %>
    <textarea name="message" rows="3" placeholder="Ask something..."
      class="border border-gray-300 rounded-md p-3 focus:outline-none focus:ring-2 focus:ring-indigo-500"></textarea>
    <button type="submit" class="self-start bg-indigo-600 text-white px-4 py-2 rounded-md hover:bg-indigo-700">
      Send
    </button>
  <% end %>
</div>
```

`@conversation.messages` comes for free from `acts_as_chat` — it's the `has_many` association, already ordered oldest-first. We loop over it, and use `message.role` (a plain string column: `"user"` or `"assistant"`) to align and colour each bubble differently. There's no helper method like `message.user?` — the role is just a string, so a direct comparison is all we need.

## Closing episode 2's open thread: Turbo Drive works again

Episode 2 ended with `data: { turbo: false }` on the form, as a workaround: Turbo Drive intercepted the submission and expected either a Turbo Stream response or a redirect, and our controller was doing neither — it rendered plain HTML, which Turbo didn't know how to handle, so the page silently failed to update.

Look at this episode's `create` action again: it doesn't render anything, it redirects. That's the Post/Redirect/Get pattern — after a form submission that changes something, redirect to a page that shows the result, rather than rendering a result directly from the POST action. It's good practice on its own (it stops a page refresh from re-submitting the form), and it also happens to be exactly what Turbo Drive expects. So the workaround is gone — the form in this episode has no `data: { turbo: false }` at all, and it works.

This is worth sitting with for a second: the fix in episode 2 wasn't wrong, but it was treating a symptom. The real fix was adopting the pattern Rails (and Turbo) are built around in the first place.

## Trying it in the browser

```bash
bin/dev
```

Open `http://127.0.0.1:3000`, ask something, and the reply appears below the form. Refresh the page — the conversation is still there. Ask a follow-up that depends on the first message ("what did I just ask you?") and the model answers correctly, because the full history is sent with every request, not just the latest line.

## What's next

Episode 4 covers streaming responses: instead of waiting for the full reply before showing anything, we'll stream it in as the model generates it, using Turbo Streams properly this time.
