---
layout: post
title: "Persistere le conversazioni RubyLLM con ActiveRecord"
series: "ai-with-ruby"
episode: 3
lang: it
ref: persisting-conversations-with-activerecord
permalink: /persisting-conversations-with-activerecord/
canonical_url: https://antoninoscaffidi.github.io/it/persisting-conversations-with-activerecord/
date: 2026-08-09 06:00:00 +0200
image: /assets/images/ai-with-ruby-banner.png
---

Nell'[episodio 2]({% post_url 2026-08-07-wiring-rubyllm-into-rails %}) avevamo fatto parlare RubyLLM con Rails tramite un form semplice: scrivi un messaggio, ricevi una risposta, ricarichi la pagina e sparisce tutto. Ogni chiamata a `RubyLLM.chat` creava una sessione nuova, in memoria, che viveva solo per la durata di quella singola richiesta.

Questo episodio risolve il problema. Alla fine, le conversazioni sopravvivono a un refresh della pagina, vivono nel database, e il modello ricorda davvero cosa è stato detto prima nella stessa conversazione — perché gli stiamo inviando la cronologia reale, non solo l'ultimo messaggio.

Il codice è taggato [`episode-3`](https://github.com/AntoninoScaffidi/ai-with-ruby-demo/tree/episode-3) nel repo [ai-with-ruby-demo](https://github.com/AntoninoScaffidi/ai-with-ruby-demo).

## Due modi per persistere una chat

Potresti scrivere a mano i modelli `Conversation` e `Message`: una coppia `has_many`/`belongs_to`, un controller che appende a un array di messaggi, un po' di serializzazione JSON per salvare cosa ha detto il modello. Funzionerebbe, ma staresti ricostruendo qualcosa che RubyLLM offre già come integrazione Rails di prima classe, completa di generator, migrazioni, e il pattern `acts_as_chat` / `acts_as_message` che trovi nella documentazione ufficiale e negli esempi della gemma.

Useremo il generator. Non perché farlo a mano sia sbagliato, ma perché il codice generato **è** il modo idiomatico di usare questa gemma in Rails — leggerlo è istruttivo quanto scriverlo da soli, e finisci con un codice che corrisponde a quello che trovi nella documentazione di RubyLLM e nei progetti di altri.

## Eseguire il generator

```bash
bin/rails generate ruby_llm:install chat:Conversation --skip-active-storage
```

Ci sono due cose non di default in questo comando, quindi rendiamole esplicite entrambe.

**`chat:Conversation`.** Lasciato al default, il generator chiama il modello che rappresenta una sessione di chat `Chat`. È un default ragionevole, ma "conversation" è la parola che useremo davvero per parlare di questa funzionalità, quindi diciamo al generator di chiamarlo `Conversation`. La riga d'uso del generator stesso spiega la sintassi: `bin/rails g ruby_llm:install [chat:ChatName] [message:MessageName] ...` — puoi rinominare allo stesso modo qualsiasi modello che crea.

**`--skip-active-storage`.** Di default il generator esegue anche `active_storage:install` e aggiunge `has_many_attached :attachments` al modello message, così una chat può avere file allegati. Non stiamo costruendo allegati file in questo episodio, quindi lo saltiamo. Nulla impedisce di aggiungerlo dopo — il flag lo salta solo **ora**.

## Cosa crea davvero il generator

Vale la pena passarlo in rassegna file per file, perché crea più di un `Conversation` e un `Message` — ed è meglio sapere perché piuttosto che ritrovarsi file inspiegati nell'app.

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

**Tabella `conversations`.** Solo un guscio vuoto più i timestamp — una conversazione da sola non deve salvare nulla, serve solo a raggruppare i messaggi. Un riferimento `model_id` viene aggiunto più avanti dalla quinta migrazione, per registrare quale modello LLM sta usando quella conversazione.

**Tabella `messages`.** È qui che sta la sostanza: `role` (`"user"`, `"assistant"`, o `"system"`), `content`, e `content_raw` (una colonna JSON che contiene la struttura grezza completa, necessaria per messaggi che non sono testo semplice — come le chiamate a tool). Ci sono anche colonne per il "extended thinking" (`thinking_text`, `thinking_signature`, `thinking_tokens` — per i modelli che espongono il proprio processo di ragionamento) e per il conteggio dei token (`input_tokens`, `output_tokens`, `cached_tokens`, `cache_creation_tokens`). Non toccheremo le colonne di thinking o dei token in questo episodio, ma vale la pena sapere che ci sono: questa stessa tabella è costruita per supportare funzionalità che arriveranno diversi episodi più avanti.

**Tabella `tool_calls`.** Non ancora usata — è per un episodio successivo dedicato al tool calling, quando il modello potrà richiamare il codice della nostra applicazione.

**Tabella `models`.** Una cache locale dei metadati dei modelli — prezzi, dimensione della context window, quali capacità supporta un dato modello. Viene popolata da un rake task separato (`bin/rails ruby_llm:load_models`) che non eseguiamo in questo episodio, dato che nulla dipende ancora da essa. Il riferimento `model_id` aggiunto a `conversations` e `messages` è opzionale (`foreign_key: true`, senza `null: false`), quindi tutto funziona bene anche senza.

**I modelli generati:**

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

`acts_as_chat` e `acts_as_message` sono class method che RubyLLM aggiunge ad `ActiveRecord::Base`. Impostano l'associazione `has_many`/`belongs_to` tra i due modelli e includono i metodi che fanno comportare un `Conversation` come una sessione di chat RubyLLM — soprattutto `.ask`. Poiché abbiamo rinominato il modello in `Conversation`, `acts_as_message` doveva sapere che l'associazione non si chiama più `chat`, da cui `chat: :conversation`. Il generator l'ha capito da solo a partire dalla mappatura `chat:Conversation` che gli abbiamo passato.

**Le directory di convenzione** (`app/agents`, `app/tools`, `app/schemas`, `app/prompts`, ciascuna con solo un `.gitkeep` per ora) sono impalcature vuote per funzionalità più avanti in questa serie — tool che un agente può richiamare, schemi per output strutturati, prompt riutilizzabili. Per ora non ci va nulla.

## Una trappola: l'API legacy deprecata

Ecco una cosa che non è ovvia dal solo README, e in cui siamo incappati direttamente. RubyLLM include **due** implementazioni dell'integrazione ActiveRecord: quella attuale (quella appena descritta) e una legacy, mantenuta solo per retrocompatibilità. Quale delle due viene caricata è controllato da un flag di configurazione, `use_new_acts_as`.

Senza di esso, RubyLLM carica silenziosamente l'implementazione legacy e stampa questo all'avvio:

```
!!! RubyLLM's legacy acts_as API is deprecated and will be removed in RubyLLM 2.0.0.
Please consult the migration guide at https://rubyllm.com/upgrading-to-1-7/
```

È esattamente il warning che avevamo visto nell'episodio 2, prima ancora di toccare la persistenza — viene stampato all'avvio non appena ActiveRecord si carica, indipendentemente dal fatto che tu stia già usando `acts_as_chat`. Il generator lo sa e imposta il flag correttamente da solo:

```ruby
# config/initializers/ruby_llm.rb
RubyLLM.configure do |config|
  config.openai_api_key = ENV.fetch("OPENAI_API_KEY", nil)

  # Usa l'API acts_as attuale, basata sulle associazioni. Senza questo,
  # RubyLLM ricade silenziosamente su un'implementazione deprecata e avvisa.
  config.use_new_acts_as = true
end
```

Se mai aggiungerai l'integrazione ActiveRecord di RubyLLM a un progetto **senza** usare il generator, questa è la riga più importante da copiare a mano — è facile da perdere, e le due implementazioni non sono identiche, quindi codice scritto per una non funziona necessariamente con l'altra.

## Eseguire le migrazioni

```bash
bin/rails db:migrate
```

Questo crea tutte e quattro le tabelle descritte sopra, più la quinta migrazione che collega le chiavi esterne tra loro.

## Provarlo dalla console, prima di toccare il controller

Vale la pena farlo una volta, per vedere la persistenza funzionare senza nient'altro di mezzo:

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

Una chiamata a `.ask`, due righe scritte: il messaggio dell'utente e la risposta del modello, entrambi salvati automaticamente. Nessun warning di deprecazione nemmeno — il flag sta facendo il suo lavoro.

## Collegarlo al controller

Il controller dell'episodio 2 creava una chat nuova e usa-e-getta a ogni richiesta. Ora dobbiamo ritrovare **la stessa** conversazione tra una richiesta e l'altra. Per una demo a singolo utente senza ancora account, la cosa più semplice che resti comunque corretta è tenere l'id della conversazione nella sessione di Rails — un piccolo cookie firmato legato al browser del visitatore.

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

`current_conversation` cerca la conversazione tramite l'id salvato nella sessione; se non ce n'è ancora una (prima visita, o sessione scaduta), ne crea una e ne ricorda l'id per la prossima volta. `create` non renderizza più nulla da solo — fa la domanda, poi reindirizza a `new`, che ora è responsabile di caricare e mostrare la conversazione, messaggi inclusi.

## Mostrare la cronologia

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

`@conversation.messages` arriva gratis da `acts_as_chat` — è l'associazione `has_many`, già ordinata dal più vecchio al più recente. Ci scorriamo sopra, e usiamo `message.role` (una semplice colonna stringa: `"user"` o `"assistant"`) per allineare e colorare diversamente ogni fumetto. Non c'è un metodo helper come `message.user?` — il ruolo è solo una stringa, quindi basta un confronto diretto.

## Chiudere il filo aperto dell'episodio 2: Turbo Drive torna a funzionare

L'episodio 2 si era chiuso con `data: { turbo: false }` sul form, come soluzione temporanea: Turbo Drive intercettava l'invio e si aspettava o una risposta Turbo Stream o un redirect, e il nostro controller non faceva né l'uno né l'altro — renderizzava HTML semplice, che Turbo non sapeva come gestire, quindi la pagina falliva silenziosamente nell'aggiornarsi.

Guarda di nuovo l'azione `create` di questo episodio: non renderizza nulla, reindirizza. Questo è il pattern Post/Redirect/Get — dopo un invio di form che cambia qualcosa, reindirizzi a una pagina che mostra il risultato, invece di renderizzare un risultato direttamente dall'azione POST. È buona pratica di per sé (impedisce che un refresh della pagina reinvii il form), e per giunta è esattamente ciò che Turbo Drive si aspetta. Quindi la soluzione temporanea è sparita — il form di questo episodio non ha alcun `data: { turbo: false }`, e funziona.

Vale la pena soffermarcisi un attimo: la correzione dell'episodio 2 non era sbagliata, ma curava un sintomo. La vera correzione era adottare il pattern su cui Rails (e Turbo) sono costruiti fin dall'inizio.

## Provarlo nel browser

```bash
bin/dev
```

Apri `http://127.0.0.1:3000`, chiedi qualcosa, e la risposta appare sotto il form. Ricarica la pagina — la conversazione è ancora lì. Fai una domanda di follow-up che dipende dal primo messaggio ("cosa ti ho appena chiesto?") e il modello risponde correttamente, perché la cronologia completa viene inviata a ogni richiesta, non solo l'ultima riga.

## Cosa viene dopo

L'episodio 4 copre le risposte in streaming: invece di aspettare la risposta completa prima di mostrare qualcosa, la faremo arrivare man mano che il modello la genera, usando Turbo Streams sul serio questa volta.
