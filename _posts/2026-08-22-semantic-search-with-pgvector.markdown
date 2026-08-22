---
layout: post
title: "Semantic Search over RubyLLM Conversations with pgvector"
series: "ai-with-ruby"
episode: 5
lang: en
ref: semantic-search-with-pgvector
permalink: /semantic-search-with-pgvector/
canonical_url: https://antoninoscaffidi.github.io/semantic-search-with-pgvector/
date: 2026-08-22 07:00:00 +0200
image: /assets/images/ai-with-ruby-ep5-banner.png
---

[Episode 4]({% post_url 2026-08-19-streaming-responses-with-turbo-streams %}) made the reply type itself in live. This episode does something different with the same conversation history: search it by *meaning*, not by matching words. Ask for "cooking dinner" and get back the message where the assistant explained how to roast a chicken, even though the word "cooking" never appears in it.

This series' sibling project, [VicinoTe]({% post_url 2026-08-08-vicinote-project-setup-and-domain %}), chose PostgreSQL from its very first episode specifically because it knew an AI module with semantic search was coming later. `ai-with-ruby-demo` didn't get that same foresight — it's been on SQLite, Rails 8's own default, since [episode 2]({% post_url 2026-08-07-wiring-rubyllm-into-rails %}), because nothing up to this point needed anything else. This is the episode where that catches up, paying now the migration cost VicinoTe avoided by planning ahead. Code is tagged [`episode-5`](https://github.com/AntoninoScaffidi/ai-with-ruby-demo/tree/episode-5) in the [ai-with-ruby-demo](https://github.com/AntoninoScaffidi/ai-with-ruby-demo) repo. As always, nothing here is left unexplained — including two real errors hit building it, with the exact messages and fixes.

## What this episode touches, at a glance

```
config/database.yml                 edit — SQLite out, PostgreSQL in
Gemfile                             edit — pg instead of sqlite3, adds the neighbor gem
db/migrate/..._install_neighbor_vector.rb   new — enables the pgvector Postgres extension
db/migrate/..._add_embedding_to_messages.rb new — a vector(1536) column + an HNSW index
app/models/message.rb               edit — has_neighbors, and an after_commit that embeds new content
app/jobs/embed_message_job.rb       new — calls RubyLLM.embed and saves the vector
app/controllers/search_controller.rb new — embeds the query, finds the nearest messages
app/views/search/show.html.erb      new — the search form and results
config/routes.rb                    edit — resource :search
app/views/chats/new.html.erb        edit — a link to the new search page
```

## What "semantic" actually means here

Every search built so far on the web works the same way underneath: it looks for the literal characters you typed. SQL's `LIKE '%cooking%'`, Postgres full-text search, `grep` — all of them, dressed up differently, are asking "does this text contain that text?" That question has a hard limit: if the word "cooking" never appears anywhere in a message about roasting a chicken, no amount of cleverness in the matching logic finds it, because the word genuinely isn't there.

An embedding model sidesteps the question entirely by not comparing text to text at all. Given a piece of text, it outputs a fixed-length list of numbers — for the model this episode uses, 1,536 of them — which together act as *coordinates* for that text in a 1,536-dimensional space. The model was trained on an enormous amount of text, and in the process learned which words and phrases tend to show up in similar contexts as which others. "Roast a chicken" and "cooking dinner" get placed near each other in that space not because they share letters, but because the patterns of language around them — recipes, meal prep, kitchens, ovens — overlap heavily across everything the model has read. Two pieces of text end up close together in this space when they mean similar things, close to independently of whether they share a single word.

"Search by meaning" then reduces to a question with actual geometry behind it: given the coordinates of a new query, which stored coordinates are *nearest* to it? That's nearest-neighbor search, a genuinely well-studied problem in computer science, unrelated to anything a normal indexed database column does — which is exactly why this episode needs a piece of infrastructure built specifically for it, rather than a clever query against the existing `content` text column.

## Setting up PostgreSQL and pgvector, for real

[pgvector](https://github.com/pgvector/pgvector) is the piece of infrastructure: a PostgreSQL extension that adds a native `vector` column type, plus the indexing and distance operators needed to answer "which stored vectors are nearest to this one" efficiently instead of comparing against every row one at a time. It needs PostgreSQL specifically — it's a Postgres extension, compiled against Postgres's own extension API, not a Ruby gem — and a fairly recent version (13+).

The demo app had been running on SQLite since episode 2, so the actual first step of this episode wasn't Ruby code at all — it was deciding whether to keep SQLite or switch, and then acting on that.

**Why migrate at all, instead of staying on SQLite?** It's not that SQLite is structurally incapable of this — an extension called [`sqlite-vec`](https://github.com/asg017/sqlite-vec) does roughly the same job pgvector does, for SQLite. Three reasons tipped this toward Postgres specifically, though:

1. **It's what the sibling project already proved out.** VicinoTe's own first episode chose PostgreSQL from day one for exactly this reason — so that adding `pgvector` later, when its own AI module arrives, needs no database migration at all. This app didn't get that same foresight (SQLite was just Rails 8's own default, picked without semantic search in view yet), so it's paying that migration cost now instead of never — but the underlying lesson is the same one either way: know what a project will eventually need, and it's cheaper to build on the right foundation from the start than to migrate onto it later.
2. **pgvector is the ecosystem default.** It's older, far more widely deployed, and the thing nearly every Rails + AI tutorial, gem README, and managed-database "add vector search" button assumes. Code and advice written against pgvector transfers directly to a real production setup; `sqlite-vec` is newer and less universally supported by the surrounding tooling.
3. **Production Rails apps mostly run Postgres already.** A demo that migrates to Postgres for this feature is closer to what deploying this for real would actually look like than one that stayed on SQLite specifically to dodge a migration.

None of that makes `sqlite-vec` a wrong choice in general — for a project committed to staying on SQLite for other reasons, it's a legitimate way to get the same capability. It just wasn't the better fit here, once a sibling project in this same blog had already shown what planning ahead for it looks like.

Here's exactly what the Postgres switch involved, on the machine this was built on (macOS, [Postgres.app](https://postgresapp.com) already running an older PostgreSQL 12 for other projects, kept untouched):

**A second, newer Postgres, side by side with the old one.** Postgres.app supports running several server versions from separate app copies, each on its own port, so nothing about the existing PG12 setup had to change:

```bash
curl -L -o Postgres-17.dmg \
  https://github.com/PostgresApp/PostgresApp/releases/download/v2.9.5/Postgres-2.9.5-17.dmg
hdiutil attach Postgres-17.dmg -nobrowse -mountpoint ./mnt
cp -R ./mnt/Postgres.app /Applications/Postgres17.app
hdiutil detach ./mnt

/Applications/Postgres17.app/Contents/Versions/17/bin/initdb \
  -D "$HOME/Library/Application Support/Postgres/var-17" -U "$(whoami)" -E UTF8
```

`initdb` creates a brand-new, empty PostgreSQL data directory — a fresh server, entirely separate from the PG12 one already in use, listening on a different port (`5433` here, chosen only because `5432` was already taken by the old one) once started.

**Building pgvector against it.** Not a gem install — pgvector is C code compiled directly against the target Postgres installation's headers:

```bash
export PATH="/Applications/Postgres17.app/Contents/Versions/17/bin:$PATH"
git clone --depth 1 https://github.com/pgvector/pgvector.git
cd pgvector
make OPTFLAGS=""
make install
```

The first attempt here used a plain `make`, and failed immediately:

```
clang: error: the clang compiler does not support '-march=native'
```

pgvector's `Makefile` defaults to `OPTFLAGS = -march=native`, an architecture-specific CPU optimization flag. Postgres.app ships **universal binaries** — a single build containing both `arm64` and `x86_64` code — and `-march=native` doesn't mean anything for a build that isn't targeting one specific architecture, so Apple's clang refuses it outright. [The Makefile's own comment documents the fix](https://github.com/pgvector/pgvector/blob/master/Makefile): `make OPTFLAGS=""`, which was the version that actually built cleanly and installed the extension's files into the new Postgres's own `share/postgresql/extension` directory.

**Confirming it actually works**, before writing a single line of Rails code, on a disposable database:

```bash
createdb -p 5433 pgvector_test
psql -p 5433 -d pgvector_test -c "CREATE EXTENSION vector; SELECT extversion FROM pg_extension WHERE extname='vector';"
```

```
 extversion
------------
 0.8.6
(1 row)
```

Only once that came back clean did the actual Rails-side migration in the rest of this post begin. If you already run Postgres 13+ with pgvector available — Docker, Homebrew, a managed database — none of the above applies to you at all; it's specific to getting there on a Mac running Postgres.app with an older server already in use, included here because "it depends on your setup" would have skipped over a real afternoon of debugging a compiler error that has nothing to do with Rails, RubyLLM, or Ruby.

## Switching the database

```yaml
# config/database.yml
default: &default
  adapter: postgresql
  encoding: unicode
  host: localhost
  port: 5433
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

development:
  <<: *default
  database: ai_with_ruby_demo_development

test:
  <<: *default
  database: ai_with_ruby_demo_test

production:
  primary:
    <<: *default
    database: ai_with_ruby_demo_production
  cache:
    <<: *default
    database: ai_with_ruby_demo_production_cache
    migrations_paths: db/cache_migrate
  queue:
    <<: *default
    database: ai_with_ruby_demo_production_queue
    migrations_paths: db/queue_migrate
  cable:
    <<: *default
    database: ai_with_ruby_demo_production_cable
    migrations_paths: db/cable_migrate
```

The shape is identical to the SQLite version from episode 2 — same three environments, same production split into `primary`/`cache`/`queue`/`cable` for the Solid trio — only `adapter:` and the connection details changed. `port: 5433` is specific to how Postgres happens to be running on the machine this was built on (a second server, alongside an existing one on the default `5432`, so the two don't conflict) — the number itself isn't meaningful, only that it has to match wherever your own Postgres is actually listening.

```ruby
# Gemfile
# Use PostgreSQL as the database for Active Record
gem "pg", "~> 1.5"
```

`sqlite3` came out, `pg` (the PostgreSQL driver) went in. Then:

```bash
bundle install
bin/rails db:create db:migrate
```

`db:create` makes the (empty) `ai_with_ruby_demo_development` and `_test` databases from scratch — this is a fresh Postgres server, so there's no data to migrate over from the old SQLite files, only the schema. `db:migrate` replays every migration from episode 3 (conversations, messages, tool_calls, models, the foreign keys between them — episode 4 added no schema changes of its own) against the new database, ending in exactly the same shape SQLite had.

## Enabling pgvector

```ruby
# db/migrate/..._install_neighbor_vector.rb
class InstallNeighborVector < ActiveRecord::Migration[8.1]
  def change
    enable_extension "vector"
  end
end
```

Generated, not hand-written — this is the exact output of `rails generate neighbor:vector`, from the [neighbor gem](https://github.com/ankane/neighbor), which was added to the `Gemfile` alongside `pg`:

```ruby
# Gemfile
gem "neighbor"
```

`neighbor` is what gives ActiveRecord a `vector` column type and nearest-neighbor query methods on top of pgvector (it also supports a couple of other backends, unused here). `enable_extension "vector"` is one line, but it's doing a real thing at the database level: it runs Postgres's own `CREATE EXTENSION vector`, which only succeeds if the pgvector extension was actually installed on the server — this is the migration that would fail first if pgvector weren't set up correctly.

## The embedding column

```ruby
# db/migrate/..._add_embedding_to_messages.rb
class AddEmbeddingToMessages < ActiveRecord::Migration[8.1]
  def change
    # 1536 dimensions matches OpenAI's text-embedding-3-small, RubyLLM's default.
    add_column :messages, :embedding, :vector, limit: 1536
    add_index :messages, :embedding, using: :hnsw, opclass: :vector_cosine_ops
  end
end
```

- `add_column :messages, :embedding, :vector, limit: 1536` — `:vector` is a real column type here, not a JSON blob or a serialized array; pgvector adds it to Postgres itself, and `neighbor` teaches Rails' migration DSL to speak it. `limit: 1536` fixes the vector's length. That number isn't arbitrary — it has to match whatever embedding model actually produces the vectors, and OpenAI's `text-embedding-3-small` (RubyLLM's default embedding model, covered below) always returns exactly 1,536 numbers. Get this wrong and every insert fails with a dimension mismatch, loudly, the first time it's tried.
- `add_index :messages, :embedding, using: :hnsw, opclass: :vector_cosine_ops` — without an index, "find the nearest vectors" means comparing the query against every single row, one at a time — fine for a demo with a handful of messages, ruinous at real scale. HNSW (Hierarchical Navigable Small World) is an approximate-nearest-neighbor index structure built for exactly this: fast lookups that are extremely likely to find the true nearest points without the guarantee of a full scan. `opclass: :vector_cosine_ops` tells the index which distance function to optimize for — **cosine distance**, which measures the *angle* between two vectors rather than the straight-line distance between them. For text embeddings, cosine is the conventional choice: it captures "these mean similar things" independent of how long either string was, which a raw distance measurement would conflate with actual dissimilarity.

## RubyLLM.embed, from the source

Before writing the code that calls it, the same habit as every episode so far: read what it actually does. `RubyLLM.embed` delegates to [`RubyLLM::Embedding.embed`](https://github.com/crmne/ruby_llm):

```ruby
def self.embed(text, model: nil, provider: nil, assume_model_exists: false, context: nil, dimensions: nil)
  config = context&.config || RubyLLM.config
  model ||= config.default_embedding_model
  model, provider_instance = Models.resolve(model, provider: provider, assume_exists: assume_model_exists, config: config)
  model_id = model.id

  payload = { provider: provider_instance.slug, provider_class: provider_instance.class.name,
              model: model_id, model_info: model, input: text, dimensions: dimensions }

  RubyLLM.instrument('embedding.ruby_llm', payload, config: config) do |event|
    result = provider_instance.embed(text, model: model_id, dimensions:)
    event[:result] = result
    result
  end
end
```

A few things worth pulling out:

- `model ||= config.default_embedding_model` — call `RubyLLM.embed("some text")` with no options at all, and it falls back to a configured default rather than requiring a model name every time. That default, from RubyLLM's own `Configuration`, is `'text-embedding-3-small'` — which is exactly why the migration above hardcoded `limit: 1536`: that's this specific model's output size, and nothing in the code enforces that the two stay in sync if either one changes later.
- `Models.resolve(model, ...)` — the same model-resolution machinery `ask`/`complete` use for chat, reused here. This is also where the very first real error of this episode came from, covered next.
- The method returns a `RubyLLM::Embedding`, whose `.vectors` attribute is the actual array of floats — that's the thing that gets assigned straight into the `embedding` column.

## A real error: the embedding model wasn't in the registry

The first time `EmbedMessageJob` actually ran (covered fully below), it failed immediately:

```
RubyLLM::ModelNotFoundError (Unknown model: "text-embedding-3-small". If the model exists
at the provider, refresh the registry with `RubyLLM.models.refresh!` and persist it with
`RubyLLM.models.save_to_json`. Rails model registries can call `Model.refresh!` instead.)
```

`Models.resolve` above looks the model name up in RubyLLM's own registry of known models — not by asking OpenAI at request time, but against a local list. Episode 3 mentioned this registry exists (the `models` table, populated by `bin/rails ruby_llm:load_models`) but never actually ran that task, since nothing up to that point depended on it — chat models were being passed straight through without a registry lookup succeeding on anything unusual. Semantic search is the first feature in this series that actually needs the registry populated, and it hadn't been:

```bash
bin/rails ruby_llm:load_models
```

```
✅ Loaded 1166 models into database
```

One command, run once, and the error was gone. Worth remembering as a general shape of bug: not everything that looks like an application error is one — this was a one-time setup step the app had always needed, that just happened to have no visible symptom until this exact feature exercised it.

## Generating an embedding whenever a message gets real content

```ruby
# app/models/message.rb
class Message < ApplicationRecord
  acts_as_message chat: :conversation
  has_neighbors :embedding

  broadcasts_to ->(message) { "conversation_#{message.conversation_id}" }, inserts_by: :append

  after_commit :enqueue_embedding, on: %i[create update], if: -> { content.present? && saved_change_to_content? }

  def broadcast_append_chunk(content)
    broadcast_append_to "conversation_#{conversation_id}",
      target: "message_#{id}_content",
      content: ERB::Util.html_escape(content.to_s)
  end

  private

  def enqueue_embedding
    EmbedMessageJob.perform_later(id)
  end
end
```

Two additions to the file episode 4 left behind:

- `has_neighbors :embedding` — from the `neighbor` gem, this is what makes `.nearest_neighbors` (used in the search controller below) available on `Message` at all. Nothing more than a declaration; the actual column and index come from the migrations above.
- `after_commit :enqueue_embedding, on: %i[create update], if: -> { content.present? && saved_change_to_content? }` — this line took a second pass to get right, and the reasoning behind each piece matters:
  - **Why `after_commit`, not `after_save`.** `enqueue_embedding` calls `perform_later`, which for the `:async` adapter used here can start running the job on another thread almost immediately. If that job's `Message.find_by(id: ...)` ran before the transaction that created the row had actually committed, it could find nothing at all. `after_commit` guarantees the row is durably saved before anything gets enqueued against it.
  - **Why `on: %i[create update]`, not just `:create`.** Recall from episode 4: a `Message` for an assistant reply is created *empty* (`content: ''`, in RubyLLM's `before_message` callback) and only filled in with real text later, via an `UPDATE` once the model finishes streaming. The create alone would only ever see blank content — the update is where the real assistant text actually lands.
  - **Why `content.present?`.** Without it, that empty-content create would still fire the callback, and `RubyLLM.embed("")` would either error or waste an API call embedding nothing.
  - **Why `saved_change_to_content?`, on top of `content.present?`.** This is the one that isn't obvious. Recall from episode 4 that `add_message` also does an incidental `.update!(content_raw:)` right after creating the user's message — a second `UPDATE` where `content` itself doesn't change. Without this check, that incidental update would pass the `content.present?` test (the content is there, just unchanged) and trigger a second, wasted embedding call for text that was already embedded moments earlier at creation. `saved_change_to_content?` is Rails' own way of asking "did *this specific attribute* actually change in the save that just committed" — true at creation (nil → the message text) and true when the assistant's placeholder gets filled in (`''` → the real reply), false for that incidental `content_raw`-only update. The net effect: exactly two embedding calls per exchange, one for the question, one for the answer, never a third.

## The background job

```ruby
# app/jobs/embed_message_job.rb
class EmbedMessageJob < ApplicationJob
  def perform(message_id)
    message = Message.find_by(id: message_id)
    return unless message

    result = RubyLLM.embed(message.content)
    message.update!(embedding: result.vectors)
  end
end
```

`Message.find_by` rather than `Message.find` — `find_by` returns `nil` for a missing row instead of raising, and `return unless message` treats that as a normal, silent no-op rather than an error. A message could in principle be deleted between the moment its embedding job was enqueued and the moment it actually runs; that's not a bug worth crashing a background job over. `result.vectors` — the array of 1,536 floats from `RubyLLM::Embedding` — gets assigned straight to the `embedding` column, and `neighbor`/pgvector handle the actual storage format underneath.

## Searching

```ruby
# app/controllers/search_controller.rb
class SearchController < ApplicationController
  def show
    @query = params[:q].to_s.strip
    return if @query.blank?

    query_embedding = RubyLLM.embed(@query).vectors
    @results = Message.where.not(embedding: nil)
                       .nearest_neighbors(:embedding, query_embedding, distance: "cosine")
                       .first(10)
  end
end
```

- `params[:q].to_s.strip` — `params[:q]` is `nil` on the first, empty visit to the search page (no query submitted yet); `.to_s` turns that into `""` so `.strip` never has to guard against a `nil` receiver. `return if @query.blank?` stops there, before ever calling the embedding API, if there's nothing to search for.
- `RubyLLM.embed(@query).vectors` — the search box's text gets embedded through the exact same call as every message. This is the part that makes semantic search possible at all: a question is only comparable to stored answers because both were placed into the same numeric space by the same model.
- `Message.where.not(embedding: nil)` — messages created before this episode (or before their own embedding job finished) have a `nil` embedding; excluding them keeps `nearest_neighbors` from ever trying to compute a distance against nothing.
- `.nearest_neighbors(:embedding, query_embedding, distance: "cosine")` — from `neighbor`. Every row this returns gets a `neighbor_distance` attribute attached — a number that didn't come from the `messages` table at all, computed by Postgres as part of the query and read back onto the in-memory record.
- `.first(10)` — takes the ten closest.

No route for `new`/`create` and no `SearchController#index` — the whole feature is one action.

```ruby
# config/routes.rb
resource :search, only: [:show], controller: :search
```

```
Prefix Verb URI Pattern       Controller#Action
search GET  /search(.:format) search#show
```

A singular `resource` rather than `resources` — there's exactly one search "page," not a collection of search records with individual URLs, so the singular form (matching `resource :chat` from earlier episodes) is the correct shape here, not just a shorter way to write the same thing. One `GET /search` route handles both the empty initial visit and a submitted query, distinguished only by whether `?q=...` is present — a plain link-clickable, bookmarkable, back-button-friendly URL, exactly what a search results page should be.

## The search page

```erb
<%# app/views/search/show.html.erb %>
<div class="max-w-xl mx-auto mt-16 px-4">
  <h1 class="text-2xl font-semibold mb-6">Search past conversations</h1>

  <%= form_with url: search_path, method: :get, class: "flex gap-3 mb-8" do %>
    <input
      type="text"
      name="q"
      value="<%= @query %>"
      placeholder="What are you looking for?"
      class="flex-1 border border-gray-300 rounded-md p-3 focus:outline-none focus:ring-2 focus:ring-indigo-500"
    >
    <button type="submit" class="bg-indigo-600 text-white px-4 py-2 rounded-md hover:bg-indigo-700">
      Search
    </button>
  <% end %>

  <% if @query.present? %>
    <% if @results.present? %>
      <div class="space-y-4">
        <% @results.each do |message| %>
          <div class="rounded-md border border-gray-200 p-3">
            <div class="flex items-center justify-between mb-1">
              <p class="text-xs text-gray-500"><%= message.role %></p>
              <p class="text-xs text-gray-400">similarity: <%= (1 - message.neighbor_distance).round(3) %></p>
            </div>
            <p><%= message.content %></p>
          </div>
        <% end %>
      </div>
    <% else %>
      <p class="text-gray-500">No matching messages yet — nothing has an embedding, or nothing is close enough.</p>
    <% end %>
  <% end %>
</div>
```

- `form_with url: search_path, method: :get, ...` — `method: :get`, not the `:post` every previous form in this series has used, and deliberately so: a search is a read, not a state change, and a `GET` is what makes the results URL (`/search?q=cooking+dinner`) something you can copy, bookmark, or hit back to.
- `value="<%= @query %>"` — re-fills the box with whatever was searched, so reloading or sharing the URL shows the same query, not an empty field next to populated results.
- `1 - message.neighbor_distance` — cosine *distance* and cosine *similarity* are complements of each other (`distance = 1 - similarity` for pgvector's cosine operator); the raw distance from the query is flipped here into the more intuitive "how similar" framing for display. Don't expect this number to sit close to `1.0` for a strong match, though — read on.

## Trying it, and a number that looks lower than it feels

```bash
bin/dev
```

Ask a few unrelated things in the chat first — one about roasting a chicken, one about pasta, one about octopuses — so there's something to actually search across. Then visit `/search` and try "cooking dinner." The chicken and pasta messages come back first, ranked correctly above the octopus one, even though "cooking dinner" appears verbatim in none of them.

The similarity numbers themselves land lower than intuition expects — around `0.28`–`0.30` for a clearly relevant match with `text-embedding-3-small`, not `0.9`-something. That's not a bug or a weak match; it's just how this particular model's vector space is shaped — cosine similarities between *unrelated* pieces of text with this model still tend to sit well above zero, so the useful signal is the *relative ranking* between results, not the absolute number one result scores on its own.

## What's next

Episode 6 covers tool calling: letting the model call into the app's own Ruby code — not just answer from what it already knows, but actually do something.
