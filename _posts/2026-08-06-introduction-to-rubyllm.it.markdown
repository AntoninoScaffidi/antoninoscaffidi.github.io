---
layout: post
title: "Introduzione a RubyLLM: portare l'AI nelle tue applicazioni Ruby"
series: "ai-with-ruby"
episode: 1
lang: it
ref: introduction-to-rubyllm
permalink: /2026/08/06/introduction-to-rubyllm/
date: 2026-08-06 18:00:00 +0200
---

Se negli ultimi anni hai sviluppato qualcosa con Ruby on Rails, probabilmente hai sentito la spinta ad aggiungere qualche funzionalità di AI: un chatbot, una casella di ricerca semantica, uno strumento che permette a un LLM di richiamare la logica della tua applicazione. L'ecosistema Python ha strumenti maturi per questo da un po' di tempo. Ruby, per molto tempo, no.

[RubyLLM](https://rubyllm.com) cambia le cose. È una gemma che offre un'interfaccia Ruby pulita e idiomatica verso i modelli linguistici di grandi dimensioni — OpenAI, Anthropic, Gemini e altri — senza costringerti a scrivere a mano richieste HTTP e parsing JSON ogni volta che vuoi parlare con un modello.

Questo articolo apre una nuova serie su questo blog in cui esploreremo l'AI nel contesto delle applicazioni Ruby e Rails. Partiremo dalle basi per arrivare a pattern più avanzati: risposte in streaming, tool calling e ricerca semantica sui tuoi dati. Questo primo episodio riguarda semplicemente l'installazione di RubyLLM e la prima chiamata.

## Perché RubyLLM

Alcuni aspetti rendono RubyLLM particolarmente adatta alle applicazioni Rails:

- **Un'unica interfaccia, molti provider.** Configuri provider e modello una sola volta, e il codice chiamante non deve cambiare se passi, ad esempio, da GPT-4 a Claude.
- **Pensata per Rails.** Si integra bene con ActiveJob, ActiveRecord e le convenzioni che già conosci, senza chiederti di adottare un framework separato.
- **Nessun boilerplate.** Niente costruzione manuale di JSON, niente parsing manuale degli chunk in streaming — ci pensa la gemma.

## Installazione

Aggiungi la gemma al tuo `Gemfile`:

```ruby
gem "ruby_llm"
```

Poi installala:

```bash
bundle install
```

## Configurazione

RubyLLM ha bisogno di almeno una API key per parlare con un provider. Un approccio comune è un initializer:

```ruby
# config/initializers/ruby_llm.rb
RubyLLM.configure do |config|
  config.openai_api_key = ENV["OPENAI_API_KEY"]
end
```

Tieni la chiave vera fuori dal controllo di versione — usa le Rails credentials o una variabile d'ambiente caricata tramite un file `.env` in sviluppo.

## La tua prima chiamata

Con la configurazione a posto, parlare con un modello è una singola chiamata di metodo:

```ruby
chat = RubyLLM.chat
response = chat.ask "What's a good name for a Ruby gem that talks to LLMs?"

puts response.content
```

`RubyLLM.chat` ti dà una sessione di chat in cui puoi continuare a fare domande, e tiene traccia della cronologia della conversazione per te — quindi una domanda di follow-up come "What about a shorter one?" avrà comunque il contesto precedente.

## Cosa viene dopo

Nel prossimo episodio di questa serie, collegheremo RubyLLM a una vera applicazione Rails e costruiremo un semplice chatbot con la cronologia delle conversazioni salvata tramite ActiveRecord. Gli episodi successivi copriranno la ricerca semantica e il tool calling — permettendo al modello di richiamare il codice della tua applicazione per recuperare dati o compiere azioni.

Se stai seguendo la serie **VicinoTe** su questo blog (un tutorial Rails da zero che costruisce un marketplace di servizi locali), questa serie sull'AI si ricollegherà a essa: il modulo avanzato di VicinoTe usa proprio RubyLLM per le funzionalità descritte sopra.
