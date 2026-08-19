---
layout: post
title: "Streaming delle risposte RubyLLM con Turbo Streams"
series: "ai-with-ruby"
episode: 4
lang: it
ref: streaming-responses-with-turbo-streams
permalink: /streaming-responses-with-turbo-streams/
canonical_url: https://antoninoscaffidi.github.io/it/streaming-responses-with-turbo-streams/
date: 2026-08-19 07:00:00 +0200
image: /assets/images/ai-with-ruby-ep4-banner.png
---

Nell'[episodio 3]({% post_url 2026-08-09-persisting-conversations-with-activerecord %}) le conversazioni avevano iniziato a sopravvivere a un refresh della pagina, ma la richiesta in sé restava una scatola nera: premi Invia, e l'intera pagina resta lì ferma — nessuno spinner, niente — finché l'intera risposta non torna dal modello, e solo allora la pagina reindirizza e la mostra, tutta insieme. Per una risposta di una frase sono uno o due secondi di nulla. Per una più lunga, possono essere cinque, sei secondi di una pagina che sembra bloccata.

Questo episodio risolve il problema: la risposta ora si scrive da sola sulla pagina man mano che il modello la genera, un token alla volta, senza ricaricare l'intera pagina. Nel frattempo chiudiamo anche un TODO rimasto dall'episodio 3 — non c'era modo di iniziare una conversazione nuova; `session[:conversation_id]` teneva sempre la stessa per sempre.

Il codice è taggato [`episode-4`](https://github.com/AntoninoScaffidi/ai-with-ruby-demo/tree/episode-4) nel repo [ai-with-ruby-demo](https://github.com/AntoninoScaffidi/ai-with-ruby-demo). Avviso onesto in anticipo: questo episodio ha incontrato due bug veri costruendolo, e li lascio entrambi nel post con i messaggi di errore esatti e il ragionamento esatto che ha portato alla correzione — quella traccia di debugging è probabilmente più utile del codice finale da solo.

## Come RubyLLM fa streaming di una risposta, dall'interno

Prima di toccare qualsiasi nostro codice, vale la pena aprire il sorgente di RubyLLM e leggere `ask` e `complete`, perché tutto quello che costruiamo in questo episodio si appoggia sul loro comportamento esatto. Ecco la parte rilevante di [`chat_methods.rb`](https://github.com/crmne/ruby_llm), il modulo che `acts_as_chat` mescola in `Conversation`:

```ruby
def ask(message = nil, with: nil, &)
  add_message(role: :user, content: build_content(message, with))
  complete(&)
end

def complete(...)
  to_llm.complete(...)
end

def setup_persistence_callbacks
  return @chat if @chat.instance_variable_get(:@_persistence_callbacks_setup)

  @chat.before_message { persist_new_message }
  @chat.after_message { |msg| persist_message_completion(msg) }

  @chat.instance_variable_set(:@_persistence_callbacks_setup, true)
  @chat
end

def persist_new_message
  @message = messages_association.create!(role: :assistant, content: '')
end
```

Da qui emergono quattro cose che contano molto per quello che stiamo per costruire:

1. **`ask` salva da sola il messaggio dell'utente**, in modo sincrono, prima di fare qualsiasi altra cosa. Se il tuo codice **inserisce anche** una riga `Message` utente prima di chiamare `ask`, ottieni due righe per una sola domanda — vedremo esattamente questo errore più sotto.
2. **Il messaggio assistant vuoto viene creato prima che esista qualsiasi testo.** `persist_new_message` gira in un callback `before_message` e crea un `Message` con `content: ''`. Quindi nel momento in cui il modello inizia a parlare, esiste già una riga nel database — con un id — in attesa di essere riempita.
3. **`ask`/`complete` accettano un blocco**, e quel blocco è l'aggancio di streaming di RubyLLM — viene invocato una volta per ogni chunk di testo man mano che il provider lo restituisce in streaming, ben prima che la risposta sia completa.
4. **Il contenuto finale viene scritto una sola volta, alla fine**, da `persist_message_completion`, in un callback `after_message` — un normale `UPDATE` sulla stessa riga creata vuota.

Messo insieme: una chiamata a `.ask(content, &block)` crea due righe (utente, poi assistant vuoto), poi chiama il tuo blocco ripetutamente man mano che arrivano i chunk, poi fa un `UPDATE` finale con il testo completo e i conteggi dei token. Nulla nella persistenza cambia se passi un blocco — l'unica cosa che un blocco aggiunge è un callback che scatta per ogni chunk. Lo streaming, in altre parole, non è un percorso di codice diverso; è esattamente la stessa `ask` che usiamo dall'episodio 3, con un blocco attaccato.

## Il piano: un job in background, non un'azione controller più lenta

Il ciclo request/response è per sua natura la forma sbagliata per questo. Una risposta HTTP deve concludersi prima che il browser possa renderizzarla — non puoi far uscire una view Rails a piccoli pezzi nel tempo tramite un normale `render`. Quindi la chiamata all'LLM deve uscire del tutto dalla request, in un job in background, e il job deve spingere ogni chunk al browser attraverso un canale laterale man mano che arriva. In un'app Rails 8, quel canale laterale sono i Turbo Stream consegnati via Action Cable — nessun framework JavaScript separato, nessun collegamento manuale di WebSocket.

Invece di inventarmi questo pattern a mano, sono andato a vedere come lo raccomanda RubyLLM stesso. La gemma include un generator, `ruby_llm:chat_ui`, costruito esattamente per questo. Eseguendolo con `--pretend` (non scrive nulla su disco) sulla nostra app, rimappato sul nome reale del nostro modello:

```bash
bin/rails generate ruby_llm:chat_ui chat:Conversation --pretend
```

```
create  app/views/conversations/{index,new,show,_conversation,_form}.html.erb
create  app/views/messages/{_assistant,_user,_system,_tool,_error,_content,_form}.html.erb
create  app/views/messages/tool_calls/_default.html.erb
create  app/views/messages/tool_results/_default.html.erb
create  app/views/messages/create.turbo_stream.erb
create  app/views/models/{index,show,_model}.html.erb
create  app/controllers/conversations_controller.rb
create  app/controllers/messages_controller.rb
create  app/controllers/models_controller.rb
create  app/jobs/conversation_response_job.rb
insert  app/models/message.rb
 route  resources :conversations { resources :messages, only: [:create] }
 route  resources :models, only: [:index, :show] { collection { post :refresh } }
```

Quello è uno scaffold CRUD completo multi-conversazione — una pagina index e una show per conversazione, un controller per sfogliare i record `Model` disponibili, view per tool call e tool result (per un episodio sul tool calling che non abbiamo ancora scritto). La nostra app deliberatamente non è fatta così: dall'episodio 3 esiste esattamente una conversazione implicita per sessione del browser, nessun elenco, nessuna pagina show separata. Adottare l'intero scaffold significherebbe riscrivere la forma dell'app per adattarla al generator, non il contrario.

Quindi invece di eseguirlo, ho letto i template e ne ho tirato fuori i tre pezzi che riguardano davvero lo streaming, adattati per mantenere il design a sessione singola dell'episodio 3:

- Un concern del modello che permette a un `Message` di trasmettere se stesso via Turbo Stream.
- Un job in background che chiama `ask` con un blocco e inoltra ogni chunk al browser.
- Un controller che arruola il job e si toglie di mezzo immediatamente.

## Far sì che Message trasmetta se stesso

```ruby
# app/models/message.rb
class Message < ApplicationRecord
  acts_as_message chat: :conversation

  broadcasts_to ->(message) { "conversation_#{message.conversation_id}" }, inserts_by: :append

  def broadcast_append_chunk(content)
    broadcast_append_to "conversation_#{conversation_id}",
      target: "message_#{id}_content",
      content: ERB::Util.html_escape(content.to_s)
  end
end
```

`broadcasts_to` è l'integrazione ActiveRecord propria di Turbo Streams (dalla gemma `turbo-rails`, non da RubyLLM), e fa più di quanto suggerisca quell'unica riga. La sua definizione reale, in [`turbo/broadcastable.rb`](https://github.com/hotwired/turbo-rails):

```ruby
def broadcasts_to(stream, inserts_by: :append, target: broadcast_target_default, **rendering)
  after_create_commit  -> { broadcast_action_later_to(stream.try(:call, self) || send(stream), action: inserts_by, target: target.try(:call, self) || target, **rendering) }
  after_update_commit  -> { broadcast_replace_later_to(stream.try(:call, self) || send(stream), **rendering) }
  after_destroy_commit -> { broadcast_remove_to(stream.try(:call, self) || send(stream)) }
end
```

Quindi una chiamata a `broadcasts_to` collega **tre** callback, non una:

- **Alla creazione**, appende una copia renderizzata del messaggio nello stream, su `target:` (di default `model_name.plural`, cioè `"messages"` — l'id di un elemento contenitore che la pagina deve fornire).
- **All'aggiornamento**, sostituisce l'elemento del messaggio stesso (target di default: se stesso, `dom_id(message)`) con una copia appena renderizzata.
- **Alla distruzione**, lo rimuove.

Quella di mezzo conta molto qui, ed è facile perdersela a una prima lettura: ogni volta che una riga `Message` viene aggiornata — incluso l'`UPDATE` finale di `persist_message_completion` che scrive il testo completo — l'**intero fumetto del messaggio** viene sostituito con un render fresco. È una bella rete di sicurezza: significa che l'ultimissimo aggiornamento sovrascrive sempre qualunque cosa gli append incrementali avessero lasciato con una copia autorevole, appena renderizzata dal database. Significa anche che un aggiornamento incidentale (come vedremo più sotto) può innescare un broadcast che non avevi chiesto.

`broadcast_append_chunk` è nostro, non di Turbo — non sta appendendo un nuovo **messaggio**, sta appendendo testo grezzo dentro un messaggio che esiste già, puntando a un `<div>` interno specifico (`message_#{id}_content`) invece che al contenitore esterno del messaggio. `ERB::Util.html_escape` conta qui: il contenuto del chunk arriva direttamente dal modello, non escapato, e questo sta renderizzando HTML grezzo nella pagina — saltare l'escape e una risposta contenente `<` o `&` corromperebbe il markup (o, peggio, diventerebbe un vettore di injection se il modello ripetesse mai qualcosa che un utente ha scritto).

## Il job in background

```ruby
# app/jobs/conversation_response_job.rb
class ConversationResponseJob < ApplicationJob
  def perform(conversation_id, content)
    conversation = Conversation.find(conversation_id)
    assistant_message = nil

    conversation.ask(content) do |chunk|
      next if chunk.content.blank?

      assistant_message ||= conversation.messages.last
      assistant_message.broadcast_append_chunk(chunk.content)
    end
  end
end
```

È quasi il job del generator stesso, con una modifica deliberata. La versione del generator chiama `conversation.messages.last` **dentro** il blocco, a ogni singolo chunk — una query SQL fresca per ogni chunk, per un messaggio che non cambia mai lungo l'intera chiamata. Sappiamo già, dalla lettura di `ask`/`complete` qui sopra, esattamente **quando** viene creato quel messaggio assistant: nel callback `before_message`, che scatta una sola volta, prima che venga mai restituito il primo chunk. Quindi la riga esiste e il suo id è fissato prima che il nostro blocco giri anche solo una volta — non c'è nulla da ricercare di nuovo dopo il primo chunk. `assistant_message ||= conversation.messages.last` lo recupera una volta e riusa lo stesso record in memoria per ogni chiamata successiva a `broadcast_append_chunk`.

`chunk.content.blank?` protegge dai chunk che portano metadati ma nessun testo (alcuni provider trasmettono un chunk finale con statistiche d'uso e `content` vuoto) — niente da appendere, e niente da trasmettere.

## Il controller: arruola, poi togliti di mezzo

```ruby
# app/controllers/chats_controller.rb
class ChatsController < ApplicationController
  def new
    @conversation = current_conversation
  end

  def create
    ConversationResponseJob.perform_later(current_conversation.id, params[:message])

    respond_to do |format|
      format.turbo_stream
      format.html { redirect_to new_chat_path }
    end
  end

  def destroy
    session.delete(:conversation_id)
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

La prima versione di questo metodo che ho scritto creava anche la riga `Message` dell'utente direttamente nel controller, prima di arruolare il job — l'errore menzionato proprio all'inizio di questo post. `ask` lo fa già internamente, quindi la riga che aggiungevo a mano era un vero, secondo messaggio utente duplicato per la stessa domanda. La correzione è stata semplicemente cancellare quella riga; `ConversationResponseJob.perform_later` basta, dato che `conversation.ask(content)` dentro il job lo crea.

`create` non reindirizza più in caso di successo. Il pattern Post/Redirect/Get dell'episodio 3 non scompare — resta lì come fallback `format.html` per una richiesta semplice, senza JS — ma con Turbo che fa il suo lavoro, la richiesta che invia il form riceve invece una risposta `format.turbo_stream`, e la pagina in sé non si ricarica mai. Tutto quello che l'utente vede dopo — il proprio messaggio che appare, il fumetto assistant che appare vuoto, il testo che si scrive da solo — arriva più tardi, sulla connessione Action Cable già aperta, non come parte di questa risposta.

`destroy` è l'intera correzione per il TODO rimasto dell'episodio 3: dimentica l'id della conversazione nella sessione e torna a `new`, che ne crea una nuova. Collegarlo alle routes è servita una parola:

```ruby
# config/routes.rb
resource :chat, only: [:new, :create, :destroy]
```

## La risposta turbo_stream: svuotare il form

```erb
<%# app/views/chats/create.turbo_stream.erb %>
<%= turbo_stream.replace "new_message" do %>
  <%= render "form" %>
<% end %>
```

Questo è **l'intero** corpo della risposta HTTP a una `POST /chat` ora. Non tocca affatto la conversazione o i messaggi — il suo unico compito è sostituire con un form nuovo e vuoto, così la textarea si svuota da sola dopo l'invio. Tutto il resto succede più tardi, sulla connessione cable, dal job in background.

Il form stesso è stato spostato nel suo partial, così sia questa risposta sia la pagina iniziale possono renderizzarlo:

```erb
<%# app/views/chats/_form.html.erb %>
<div id="new_message">
  <%= form_with url: chat_path, method: :post, class: "flex flex-col gap-3" do %>
    <textarea
      name="message"
      rows="3"
      placeholder="Ask something..."
      class="border border-gray-300 rounded-md p-3 focus:outline-none focus:ring-2 focus:ring-indigo-500"
    ></textarea>

    <button
      type="submit"
      class="self-start bg-indigo-600 text-white px-4 py-2 rounded-md hover:bg-indigo-700"
    >
      Send
    </button>
  <% end %>
</div>
```

L'`id="new_message"` sul `div` che avvolge tutto è ciò che `turbo_stream.replace "new_message"` prende di mira.

## Bug #1: renderizzare i messaggi per ruolo, e un nome di variabile che cambia sotto i piedi

La view dell'episodio 3 scorreva `@conversation.messages` a mano e diramava su `message.role` inline per scegliere lo stile. Non funziona più nel momento in cui altro codice — `broadcasts_to` — deve renderizzare un `Message` da solo, senza il ciclo della nostra view intorno. Quindi questo episodio sposta il rendering in dei partial, e `render @conversation.messages` usa la convenzione standard di Rails "un partial per record": per un `Message`, normalmente significherebbe `messages/_message.html.erb`. Ho scritto esattamente quel partial per primo, e ho ottenuto questo:

```
ActionView::MissingTemplate in Chats#new
Missing partial messages/_user with {locale: [:en], formats: [:html], ...}
```

Non `_message` — `_user`. Il motivo sta nel concern `Message` di RubyLLM stesso, [`message_methods.rb`](https://github.com/crmne/ruby_llm):

```ruby
def to_partial_path
  partial_prefix = self.class.name.underscore.pluralize
  role_partial = if to_llm.tool_call?
                   'tool_calls'
                 elsif role.to_s == 'tool'
                   'tool'
                 else
                   role.to_s.presence || 'assistant'
                 end
  "#{partial_prefix}/#{role_partial}"
end
```

RubyLLM sovrascrive `to_partial_path` così che un `Message` scelga il proprio partial in base al ruolo — `messages/user`, `messages/assistant`, e così via. È esattamente per questo che il generator ufficiale include partial separati `_user.html.erb`, `_assistant.html.erb`, `_system.html.erb`, `_tool.html.erb` invece di uno solo generico: deve, questa sovrascrittura lo impone. Quindi la correzione è stata rinominare il file — `messages/_user.html.erb` e `messages/_assistant.html.erb` — non cambiare nulla nella chiamata di render.

Questo ha risolto l'errore di template mancante, ma non la pagina. Ricaricando è comparso un errore diverso:

```
NameError in Chats#new
undefined local variable or method 'user' for an instance of #<Class:0x...>
```

Avevo scritto il partial aspettandomi una locale chiamata `message` (il nome naturale, coerente con la variabile usata ovunque altrove in questa app):

```erb
<div id="<%= dom_id(message) %>" class="text-right">
  ...
```

Ma quando Rails renderizza un partial risolto tramite `to_partial_path`, chiama la variabile locale come il **partial**, non come la classe — per `messages/_user.html.erb`, quella è `user`, non `message`. Ho corretto il nome e sono andato avanti, convinto che la storia finisse lì. Non era così.

## Bug #2: lo stesso partial, renderizzato in due modi diversi, con due nomi di variabile diversi

Dopo quella correzione sembrava tutto a posto — finché non ho davvero inviato un messaggio e ho visto i broadcast **live** (quelli innescati da `broadcasts_to`, non il render iniziale della pagina) esplodere nel log del server:

```
Error performing Turbo::Streams::ActionBroadcastJob ...:
ActionView::Template::Error (undefined local variable or method 'user' for an instance of #<Class:0x...>):
app/views/messages/_user.html.erb:2
```

Stesso errore, stesso file — ma il render iniziale della pagina, pochi istanti prima nello stesso identico log, aveva funzionato bene con lo stesso identico partial. Due percorsi di codice diversi stavano renderizzando `messages/_user.html.erb` con due insiemi diversi di variabili locali. `render @conversation.messages` (una collection) deduce il nome della locale dal percorso del partial risolto — `user`. Ma il callback `after_update_commit` di `broadcasts_to` (quello innescato dall'aggiornamento incidentale di `content_raw` dentro `add_message`, menzionato sopra) chiama `broadcast_replace_later_to`, che renderizza lo stesso partial con `locals: {message: ...}` esplicito — la locale prende il nome dalla **classe del modello**, non dal partial.

Stesso partial, due chiamanti, due nomi di variabile locale diversi per lo stesso identico oggetto. La correzione è una prima riga difensiva, che legge quale delle due è effettivamente presente:

```erb
<%# app/views/messages/_user.html.erb %>
<% user = local_assigns[:user] || local_assigns[:message] %>
<div id="<%= dom_id(user) %>" class="text-right">
  <p class="text-xs text-gray-500 mb-1">user</p>
  <div id="<%= dom_id(user) %>_content" class="inline-block rounded-md px-3 py-2 bg-indigo-600 text-white">
    <%= user.content %>
  </div>
</div>
```

```erb
<%# app/views/messages/_assistant.html.erb %>
<% assistant = local_assigns[:assistant] || local_assigns[:message] %>
<div id="<%= dom_id(assistant) %>" class="text-left">
  <p class="text-xs text-gray-500 mb-1">assistant</p>
  <div id="<%= dom_id(assistant) %>_content" class="inline-block rounded-md px-3 py-2 bg-gray-100">
    <%= assistant.content %>
  </div>
</div>
```

È esattamente la forma del fallback nel partial generato da RubyLLM stesso (`assistant ||= local_assigns[:message]`) — non ne avevo notato il motivo finché non ho sbattuto contro lo stesso muro io stesso. Leggere codice generato prima di averne bisogno e leggerlo **dopo** che ti ha appena rotto la pagina insegnano lezioni molto diverse; questa è stata la seconda.

Il contenitore di cui ha bisogno il `render @conversation.messages` iniziale, e che l'append di `broadcasts_to` alla creazione (target di default `model_name.plural`, `"messages"`) deve trovare già presente in pagina:

```erb
<%# app/views/chats/new.html.erb — estratto %>
<div id="messages" class="space-y-4 mb-8">
  <%= render @conversation.messages %>
</div>
```

## Bug #3 (quello vero): due nomi diversi per lo stesso stream

Con entrambi i partial corretti, ho ricaricato, inviato un messaggio, e — niente. Nessun errore da nessuna parte. Il form si svuotava (quindi la risposta `turbo_stream` di `create` aveva funzionato). Ma nessun fumetto utente, nessun fumetto assistant, nemmeno dopo diversi secondi, nemmeno dopo che il job in background aveva chiaramente finito (potevo vedere la risposta completa, salvata correttamente, ricaricando la pagina).

Il log di Rails raccontava una storia **completamente pulita**: il job girava, `conversation.ask` creava entrambe le righe, ogni `Turbo::Streams::ActionBroadcastJob` veniva eseguito con successo, e riga dopo riga di `[ActionCable] Broadcasting to conversation_7: ...` mostrava ogni singolo chunk uscire, fino all'ultimissimo. Lato server, tutto funzionava. Il browser semplicemente non riceveva mai nulla.

Ho controllato la pagina stessa con un po' di JavaScript, dalla console del browser:

```js
const el = document.querySelector('turbo-cable-stream-source');
el.hasAttribute('connected')   // true
el.subscription                // presente
```

Connesso, sottoscritto, nessun errore in console — e ancora niente in arrivo. A questo punto le due parti sembravano individualmente corrette e reciprocamente irraggiungibili, che è esattamente ciò che succede quando ciascuna sta ascoltando / trasmettendo su uno stream **diverso** che appare semplicemente uguale a chi legge il codice.

La sottoscrizione vive nella view:

```erb
<%= turbo_stream_from @conversation %>
```

L'avevo scritta come la scriveresti se avessi visto `turbo_stream_from` usato solo con un semplice oggetto ActiveRecord — passi il record, fatto. Ed è **valido** — `turbo_stream_from` accetta qualsiasi "streamable", incluso un'istanza di modello, e ne deriva un nome di stream (basato sul suo GlobalID). Il problema è che `@conversation` non era mai l'identificatore usato in nessun altro punto di questo episodio. `broadcasts_to` e `broadcast_append_chunk`, nel modello `Message`, costruiscono entrambi il nome dello stream da una semplice stringa:

```ruby
"conversation_#{message.conversation_id}"
```

Una stringa e un oggetto modello non producono lo stesso nome di stream firmato solo perché un umano li legge come "la stessa conversazione". `turbo_stream_from @conversation` e `broadcasts_to ->(message) { "conversation_#{message.conversation_id}" }` stavano silenziosamente sottoscrivendosi e trasmettendo su due canali completamente diversi — nessun errore su nessuno dei due lati, perché nessuno dei due sbaglia nulla isolatamente; semplicemente non si incontrano mai.

Il template della view del generator stesso me l'ha chiarito — [`chats/show.html.erb.tt`](https://github.com/crmne/ruby_llm):

```erb
<%%= turbo_stream_from "<%= chat_variable_name %>_#{@<%= chat_variable_name %>.id}" %>
```

Una stringa, costruita nello stesso modo di quella del modello. Allineandomi:

```erb
<%= turbo_stream_from "conversation_#{@conversation.id}" %>
```

Una riga, e ogni broadcast che stava andando silenziosamente a vuoto ha iniziato ad arrivare all'istante. La lezione sotto il bug: `turbo_stream_from` e `broadcasts_to` non devono per forza concordare su **come** dai un nome a uno stream — oggetto o stringa, non importa quale — ma ogni sottoscrittore e ogni trasmettitore devono assolutamente concordare tra loro, esattamente, perché non c'è un percorso di errore quando non lo fanno. Fallisce restando in silenzio, non sollevando un'eccezione.

La view completa e funzionante:

```erb
<%# app/views/chats/new.html.erb %>
<div class="max-w-xl mx-auto mt-16 px-4">
  <div class="flex items-center justify-between mb-6">
    <h1 class="text-2xl font-semibold">RubyLLM chat demo</h1>
    <%= button_to "New conversation", chat_path, method: :delete, class: "text-sm text-gray-500 hover:text-gray-700" %>
  </div>

  <%= turbo_stream_from "conversation_#{@conversation.id}" %>

  <div id="messages" class="space-y-4 mb-8">
    <%= render @conversation.messages %>
  </div>

  <%= render "form" %>
</div>
```

## Provarlo nel browser

```bash
bin/dev
```

Apri `http://127.0.0.1:3000`, chiedi qualcosa, e questa volta la risposta non appare semplicemente — si riempie progressivamente, come ci si aspetta da un'app di chat. Ecco uno scambio reale catturato durante i test di questo episodio:

![Due scambi domanda/risposta nella demo di chat: si chiede di spiegare Turbo Streams a un bambino di cinque anni, poi un incoraggiamento di due righe per fare debug di ActionCable a mezzanotte — la risposta menziona di controllare che "il nome dello stream corrisponda esattamente", che è esattamente il bug raccontato in questo episodio.](/assets/images/ai-with-ruby-ep4-chat-pep-talk.png)

*Non pianificato, ma calzante: ho chiesto un incoraggiamento sul debug di ActionCable, e la risposta è atterrata su "controlla che il nome dello stream corrisponda esattamente" — esattamente il bug della sezione qui sopra.*

Una sessione più lunga, che mostra il link "New conversation" (in alto a destra) che chiude il TODO dell'episodio 3, e più scambi accumulati nella stessa conversazione:

![La stessa demo di chat dopo un terzo scambio: una breve storia su un messaggio WebSocket che viaggia dal server al browser, con tutte e tre le coppie domanda/risposta impilate in ordine.](/assets/images/ai-with-ruby-ep4-chat-full.png)

## Un avvertimento da conoscere: l'ordine dei broadcast non è garantito

Mentre catturavo quella seconda schermata, mi sono imbattuto in qualcosa che vale la pena segnalare invece di sistemare silenziosamente. Il callback `after_create_commit` di `broadcasts_to` non trasmette in modo sincrono — chiama `broadcast_action_later_to`, che arruola **un altro** job in background (`Turbo::Streams::ActionBroadcastJob`) per fare il rendering e il broadcast effettivi. Quindi una singola chiamata a `.ask`, sotto il cofano, arruola diversi job in rapida successione: uno per appendere il messaggio utente, uno (dall'aggiornamento incidentale di `content_raw`) per sostituirlo di nuovo, uno per appendere il messaggio assistant vuoto, più uno finale per sostituirlo con il testo finito.

Con l'adapter di coda predefinito di Rails, `:async` — un piccolo pool di thread nello stesso processo — questi non hanno la garanzia stretta di finire nell'ordine in cui sono stati arruolati. In un test dal vivo ho visto la risposta di un assistant renderizzarsi **prima** della domanda dell'utente a cui stava rispondendo, semplicemente perché quel job di append è capitato a finire su un thread diverso qualche millisecondo prima. Ricaricare la pagina subito dopo mostrava l'ordine corretto — i messaggi vengono sempre recuperati con `ORDER BY created_at`, quindi il database non sbaglia mai, solo una specifica pagina che si aggiorna dal vivo può momentaneamente mostrare le cose fuori sequenza. In produzione, con una coda vera (Solid Queue, già configurata in questa app) è molto meno probabile che sia visibile alla normale velocità di scrittura e lettura, ma non è nemmeno strutturalmente impossibile. Non è qualcosa che questo episodio risolve — solo qualcosa che vale la pena sapere che esiste, nel caso un messaggio sembri mai saltare la coda sullo schermo.

## Cosa viene dopo

L'episodio 5 copre la ricerca semantica: trasformare il contenuto dei `Message` in embedding, e permettere a un utente di cercare tra le conversazioni passate per significato, non solo per parole esatte.
