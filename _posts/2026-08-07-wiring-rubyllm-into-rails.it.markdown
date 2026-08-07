---
layout: post
title: "Collegare RubyLLM a Rails: un form di chat minimale"
series: "ai-with-ruby"
episode: 2
lang: it
ref: wiring-rubyllm-into-rails
permalink: /2026/08/07/wiring-rubyllm-into-rails/
canonical_url: https://antoninoscaffidi.github.io/it/2026/08/07/wiring-rubyllm-into-rails/
date: 2026-08-07 12:00:00 +0200
---

Nell'[episodio 1]({% post_url 2026-08-06-introduction-to-rubyllm %}) abbiamo installato RubyLLM e fatto una singola chiamata da Ruby puro. Questa volta lo colleghiamo a una vera app Rails: un form dove scrivi un messaggio e ricevi una risposta dal modello. Nessun database, nessuna cronologia della conversazione ancora — quella è l'episodio 3. L'obiettivo qui è solo vedere RubyLLM funzionare dentro un vero ciclo richiesta/risposta di Rails.

Il codice completo di questo episodio è su GitHub, taggato [`episode-2`](https://github.com/AntoninoScaffidi/ai-with-ruby-demo/tree/episode-2), in un nuovo repo di accompagnamento — [ai-with-ruby-demo](https://github.com/AntoninoScaffidi/ai-with-ruby-demo) — che crescerà a ogni post di questa serie.

## Configurazione dell'app

Una nuova app Rails 8 con Tailwind CSS:

```bash
rails new ai-with-ruby-demo --css tailwind
```

Due gemme nel `Gemfile`:

```ruby
gem "ruby_llm"

group :development do
  gem "dotenv-rails"
end
```

`ruby_llm` è la libreria vera e propria. `dotenv-rails` carica un file `.env` dentro `ENV` in development — Rails non lo fa da solo.

## Tenere la API key fuori da git

Il `.gitignore` di default di Rails 8 esclude già `.env*`, quindi un file `.env` con la tua chiave reale non viene mai committato. Committiamo invece un `.env.example`, come modello per chi clona il repo:

```
OPENAI_API_KEY=sk-your-key-here
```

Copialo in locale e inserisci la tua chiave reale:

```bash
cp .env.example .env
```

## Configurare RubyLLM

I file dentro `config/initializers/` vengono eseguiti una sola volta, all'avvio, prima che qualsiasi richiesta venga gestita — il posto giusto per configurare una gemma come questa:

```ruby
# config/initializers/ruby_llm.rb
RubyLLM.configure do |config|
  config.openai_api_key = ENV.fetch("OPENAI_API_KEY", nil)
end
```

## Rotte e controller

Un `resource` (singolare) è adatto qui — non c'è una lista di chat da elencare o una specifica da recuperare per ID, solo un unico form:

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

`RubyLLM.chat` crea una nuova sessione di chat in memoria — nulla viene salvato da nessuna parte, ed è proprio per questo che la conversazione sparisce con un refresh, per ora. `.ask` invia il messaggio e blocca l'esecuzione finché il modello non risponde; `.content` è il testo della risposta.

## La view

```erb
<%= form_with url: chat_path, method: :post, data: { turbo: false } do %>
  <textarea name="message"><%= @message %></textarea>
  <button type="submit">Send</button>
<% end %>

<% if @response %>
  <p><%= @response %></p>
<% end %>
```

## Il problema con Turbo

La prima versione di questo form non aveva `data: { turbo: false }`, e inviarlo non faceva... nulla di visibile. I log del server mostravano un pulito `200 OK`, ma la pagina non si aggiornava mai.

La causa: Rails 8 include di default Turbo Drive, che intercetta ogni invio di form e lo manda come richiesta in background, aspettandosi o un redirect o una risposta nel formato Turbo Stream. Il nostro controller renderizzava semplice HTML — Turbo lo riceveva ma non sapeva cosa farci, quindi non faceva nulla silenziosamente.

`data: { turbo: false }` dice a Turbo di lasciare stare questo form specifico e inviarlo alla vecchia maniera: un vero caricamento di pagina. È la soluzione giusta per ora. Quando arriveremo alle risposte in streaming più avanti in questa serie, vorremo davvero usare Turbo Streams — ma per una prima versione "funziona o no", disabilitarlo è la strada più semplice.

## Cosa viene dopo

L'episodio 3 aggiunge la persistenza: modelli `Conversation` e `Message`, così la cronologia della chat sopravvive a un refresh della pagina invece di vivere solo dentro una singola richiesta.
