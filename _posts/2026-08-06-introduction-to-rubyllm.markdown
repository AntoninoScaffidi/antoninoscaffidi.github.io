---
layout: post
title: "Introduction to RubyLLM: Bringing AI to Your Ruby Applications"
series: "ai-with-ruby"
episode: 1
lang: en
ref: introduction-to-rubyllm
permalink: /2026/08/06/introduction-to-rubyllm/
date: 2026-08-06 18:00:00 +0200
---

If you've built anything with Ruby on Rails over the last couple of years, you've probably felt the pull toward adding some kind of AI feature: a chatbot, a semantic search box, a tool that lets an LLM call into your app's own logic. The Python ecosystem has had mature tooling for this for a while. Ruby, for a long time, did not.

[RubyLLM](https://rubyllm.com) changes that. It's a gem that gives you a clean, idiomatic Ruby interface to large language models — OpenAI, Anthropic, Gemini, and others — without forcing you to hand-roll HTTP requests and JSON parsing every time you want to talk to a model.

This post kicks off a new series on this blog where we'll explore AI in the context of Ruby and Rails applications. We'll start simple and build up to more advanced patterns: streaming responses, tool calling, and semantic search over your own data. This first episode is just about getting RubyLLM installed and making your first call.

## Why RubyLLM

A few things make RubyLLM a good fit for Rails apps specifically:

- **One interface, many providers.** You configure a provider and model once, and the calling code doesn't need to change if you switch from, say, GPT-4 to Claude.
- **Rails-friendly.** It plays well with ActiveJob, ActiveRecord, and the conventions you already know, instead of asking you to adopt a separate framework.
- **No boilerplate.** No manual JSON building, no manually parsing streaming chunks — the gem handles that.

## Installation

Add the gem to your `Gemfile`:

```ruby
gem "ruby_llm"
```

Then install it:

```bash
bundle install
```

## Configuration

RubyLLM needs at least one API key to talk to a provider. A common approach is an initializer:

```ruby
# config/initializers/ruby_llm.rb
RubyLLM.configure do |config|
  config.openai_api_key = ENV["OPENAI_API_KEY"]
end
```

Keep the actual key out of source control — use Rails credentials or an environment variable loaded via a `.env` file in development.

## Your First Call

With configuration in place, talking to a model is a single method call:

```ruby
chat = RubyLLM.chat
response = chat.ask "What's a good name for a Ruby gem that talks to LLMs?"

puts response.content
```

`RubyLLM.chat` gives you a chat session you can keep asking questions in, and it keeps track of the conversation history for you — so a follow-up question like "What about a shorter one?" will still have the earlier context.

## What's Next

In the next episode of this series, we'll wire RubyLLM into a real Rails app and build a simple chatbot backed by ActiveRecord-stored conversation history. Later episodes will cover semantic search and tool calling — letting the model call into your own application code to fetch data or take actions.

If you're following the **VicinoTe** series on this blog (a from-scratch Rails tutorial building a local services marketplace), this AI series will eventually connect back to it: VicinoTe's advanced module uses RubyLLM for exactly the features described above.
