---
layout: post
title: "Ricerca semantica sulle conversazioni RubyLLM con pgvector"
series: "ai-with-ruby"
episode: 5
lang: it
ref: semantic-search-with-pgvector
permalink: /semantic-search-with-pgvector/
canonical_url: https://antoninoscaffidi.github.io/it/semantic-search-with-pgvector/
date: 2026-08-22 07:00:00 +0200
image: /assets/images/ai-with-ruby-banner.png
---

L'[episodio 4]({% post_url 2026-08-19-streaming-responses-with-turbo-streams %}) aveva fatto scrivere la risposta da sola, in diretta. Questo episodio fa qualcosa di diverso con la stessa cronologia delle conversazioni: cercarla per **significato**, non per corrispondenza di parole. Chiedi "cucinare la cena" e ottieni indietro il messaggio in cui l'assistente ha spiegato come arrostire un pollo, anche se la parola "cucinare" non compare mai al suo interno.

Questo è l'episodio per cui la serie si sta preparando dall'[episodio 1]({% post_url 2026-08-06-introduction-to-rubyllm %}), che aveva scelto PostgreSQL apposta perché questo fosse possibile più avanti senza una migrazione di database a metà serie. Oggi è quel "più avanti". Il codice è taggato [`episode-5`](https://github.com/AntoninoScaffidi/ai-with-ruby-demo/tree/episode-5) nel repo [ai-with-ruby-demo](https://github.com/AntoninoScaffidi/ai-with-ruby-demo). Come sempre, niente resta inspiegato — inclusi due errori veri incontrati costruendolo, con i messaggi esatti e le correzioni.

## Cosa tocca questo episodio, in sintesi

```
config/database.yml                 modifica — fuori SQLite, dentro PostgreSQL
Gemfile                             modifica — pg al posto di sqlite3, aggiunge la gemma neighbor
db/migrate/..._install_neighbor_vector.rb   nuovo — abilita l'estensione Postgres pgvector
db/migrate/..._add_embedding_to_messages.rb nuovo — una colonna vector(1536) più un indice HNSW
app/models/message.rb               modifica — has_neighbors, e un after_commit che genera l'embedding dei nuovi contenuti
app/jobs/embed_message_job.rb       nuovo — chiama RubyLLM.embed e salva il vettore
app/controllers/search_controller.rb nuovo — genera l'embedding della query, trova i messaggi più vicini
app/views/search/show.html.erb      nuovo — il form di ricerca e i risultati
config/routes.rb                    modifica — resource :search
app/views/chats/new.html.erb        modifica — un link alla nuova pagina di ricerca
```

## Perché serve Postgres, non SQLite

Un embedding testuale è una lista di numeri — per il modello usato in questo episodio, 1.536 — posizionati in modo che pezzi di testo con **significato** simile finiscano come punti vicini in quello spazio a 1.536 dimensioni. "Trovare messaggi simili" diventa "trovare i punti più vicini," un problema computazionale reale e ben studiato, non qualcosa che si aggiunge a una normale colonna di database.

[pgvector](https://github.com/pgvector/pgvector) è un'estensione PostgreSQL che aggiunge un tipo di colonna `vector` nativo e l'indicizzazione e gli operatori di distanza necessari per cercarla in modo efficiente. Serve PostgreSQL nello specifico — è un'estensione di Postgres, non una gemma — e una versione abbastanza recente (13+). L'app demo girava su SQLite dall'episodio 1, quindi il primo vero passo di questo episodio non è stato codice Ruby: configurare un server PostgreSQL con pgvector disponibile, poi migrarci l'app.

Quella parte è per lo più una configurazione dell'ambiente una tantum piuttosto che qualcosa che questo post deve ripercorrere riga per riga (i passi esatti dipendono da come già usi Postgres — Docker, Homebrew, Postgres.app, un servizio gestito). L'unica cosa da segnalare se ci incappi: compilare pgvector da sorgente su macOS con [Postgres.app](https://postgresapp.com) può fallire con `clang: error: the clang compiler does not support '-march=native'`, perché Postgres.app distribuisce binari universali (`arm64` + `x86_64` combinati) e il `Makefile` di pgvector usa di default un flag di ottimizzazione specifico per architettura che non si applica a una build universale. La correzione, [documentata proprio nel commento del Makefile stesso](https://github.com/pgvector/pgvector/blob/master/Makefile): `make OPTFLAGS=""` invece di un semplice `make`.

## Cambiare database

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

La forma è identica alla versione SQLite dell'episodio 1 — stessi tre environment, stessa produzione divisa in `primary`/`cache`/`queue`/`cable` per il trio Solid — è cambiato solo `adapter:` e i dettagli di connessione. `port: 5433` è specifico a come Postgres si trova a girare sulla macchina su cui è stato costruito questo (un secondo server, accanto a uno esistente sulla porta di default `5432`, così i due non entrano in conflitto) — il numero in sé non è significativo, conta solo che corrisponda a dove il tuo Postgres è davvero in ascolto.

```ruby
# Gemfile
# Use PostgreSQL as the database for Active Record
gem "pg", "~> 1.5"
```

`sqlite3` è uscita, `pg` (il driver PostgreSQL) è entrata. Poi:

```bash
bundle install
bin/rails db:create db:migrate
```

`db:create` crea da zero i database (vuoti) `ai_with_ruby_demo_development` e `_test` — questo è un server Postgres nuovo, quindi non c'è nessun dato da migrare dai vecchi file SQLite, solo lo schema. `db:migrate` rigioca ogni migrazione degli episodi 3 e 4 (conversations, messages, tool_calls, models, le foreign key tra loro) sul nuovo database, finendo esattamente nella stessa forma che aveva SQLite.

## Abilitare pgvector

```ruby
# db/migrate/..._install_neighbor_vector.rb
class InstallNeighborVector < ActiveRecord::Migration[8.1]
  def change
    enable_extension "vector"
  end
end
```

Generata, non scritta a mano — è esattamente l'output di `rails generate neighbor:vector`, dal [gem neighbor](https://github.com/ankane/neighbor), aggiunto al `Gemfile` accanto a `pg`:

```ruby
# Gemfile
gem "neighbor"
```

`neighbor` è ciò che dà ad ActiveRecord un tipo di colonna `vector` e dei metodi di query per i vicini più prossimi sopra pgvector (supporta anche un paio di altri backend, non usati qui). `enable_extension "vector"` è una riga sola, ma fa una cosa vera a livello di database: esegue il `CREATE EXTENSION vector` di Postgres stesso, che ha successo solo se l'estensione pgvector è davvero installata sul server — è la migrazione che fallirebbe per prima se pgvector non fosse configurato correttamente.

## La colonna embedding

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

- `add_column :messages, :embedding, :vector, limit: 1536` — `:vector` qui è un tipo di colonna vero, non un blob JSON o un array serializzato; pgvector lo aggiunge a Postgres stesso, e `neighbor` insegna al DSL delle migrazioni di Rails a parlarlo. `limit: 1536` fissa la lunghezza del vettore. Quel numero non è arbitrario — deve corrispondere a qualsiasi modello di embedding produca davvero i vettori, e `text-embedding-3-small` di OpenAI (il modello di embedding di default di RubyLLM, trattato sotto) restituisce sempre esattamente 1.536 numeri. Sbagliarlo fa fallire ogni inserimento con un errore di dimensioni non corrispondenti, rumorosamente, al primo tentativo.
- `add_index :messages, :embedding, using: :hnsw, opclass: :vector_cosine_ops` — senza un indice, "trova i vettori più vicini" significa confrontare la query con ogni singola riga, una alla volta — va bene per una demo con una manciata di messaggi, rovinoso su scala reale. HNSW (Hierarchical Navigable Small World) è una struttura di indice per vicini-più-prossimi approssimati costruita proprio per questo: lookup veloci che hanno una probabilità altissima di trovare i punti davvero più vicini senza la garanzia di una scansione completa. `opclass: :vector_cosine_ops` dice all'indice per quale funzione di distanza ottimizzare — la **distanza coseno**, che misura l'**angolo** tra due vettori invece della distanza in linea retta tra loro. Per gli embedding testuali, il coseno è la scelta convenzionale: cattura "questi significano cose simili" indipendentemente da quanto fosse lunga ciascuna stringa, cosa che una misura di distanza grezza confonderebbe con la vera dissimilarità.

## RubyLLM.embed, dal sorgente

Prima di scrivere il codice che lo chiama, la stessa abitudine di ogni episodio finora: leggere cosa fa davvero. `RubyLLM.embed` delega a [`RubyLLM::Embedding.embed`](https://github.com/crmne/ruby_llm):

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

Alcune cose che vale la pena tirare fuori:

- `model ||= config.default_embedding_model` — chiama `RubyLLM.embed("qualche testo")` senza nessuna opzione, e ricade su un default configurato invece di richiedere un nome di modello ogni volta. Quel default, dalla `Configuration` di RubyLLM stessa, è `'text-embedding-3-small'` — che è esattamente perché la migrazione sopra aveva `limit: 1536` fisso: è la dimensione di output di questo specifico modello, e nulla nel codice impone che i due restino sincronizzati se uno dei due cambia più avanti.
- `Models.resolve(model, ...)` — lo stesso meccanismo di risoluzione dei modelli che usano `ask`/`complete` per la chat, riusato qui. È anche da dove è arrivato il primo vero errore di questo episodio, trattato subito dopo.
- Il metodo restituisce un `RubyLLM::Embedding`, il cui attributo `.vectors` è il vero array di float — è quello che viene assegnato direttamente nella colonna `embedding`.

## Un errore vero: il modello di embedding non era nel registro

La prima volta che `EmbedMessageJob` ha davvero girato (trattato per intero sotto), è fallito immediatamente:

```
RubyLLM::ModelNotFoundError (Unknown model: "text-embedding-3-small". If the model exists
at the provider, refresh the registry with `RubyLLM.models.refresh!` and persist it with
`RubyLLM.models.save_to_json`. Rails model registries can call `Model.refresh!` instead.)
```

`Models.resolve` sopra cerca il nome del modello nel registro di modelli conosciuti di RubyLLM stesso — non chiedendo a OpenAI al momento della richiesta, ma contro una lista locale. L'episodio 3 aveva menzionato che questo registro esiste (la tabella `models`, popolata da `bin/rails ruby_llm:load_models`) ma non aveva mai davvero eseguito quel task, dato che fino a quel punto nulla ne dipendeva — i modelli di chat passavano dritti senza che nessun lookup nel registro dovesse riuscire su niente di insolito. La ricerca semantica è la prima funzionalità di questa serie ad avere davvero bisogno del registro popolato, e non lo era:

```bash
bin/rails ruby_llm:load_models
```

```
✅ Loaded 1166 models into database
```

Un comando, eseguito una volta, e l'errore è sparito. Vale la pena ricordarlo come forma generale di bug: non tutto ciò che sembra un errore dell'applicazione lo è — questo era un passo di configurazione una tantum di cui l'app aveva sempre avuto bisogno, che semplicemente non aveva avuto nessun sintomo visibile finché questa esatta funzionalità non l'ha messo alla prova.

## Generare un embedding ogni volta che un messaggio ottiene contenuto vero

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

Due aggiunte al file lasciato dall'episodio 4:

- `has_neighbors :embedding` — dal gem `neighbor`, è ciò che rende disponibile `.nearest_neighbors` (usato nel controller di ricerca sotto) su `Message` in primo luogo. Nient'altro che una dichiarazione; la colonna e l'indice veri vengono dalle migrazioni sopra.
- `after_commit :enqueue_embedding, on: %i[create update], if: -> { content.present? && saved_change_to_content? }` — questa riga ha richiesto un secondo passaggio per essere giusta, e il ragionamento dietro ogni pezzo conta:
  - **Perché `after_commit`, non `after_save`.** `enqueue_embedding` chiama `perform_later`, che con l'adapter `:async` usato qui può far partire il job su un altro thread quasi immediatamente. Se il `Message.find_by(id: ...)` di quel job girasse prima che la transazione che ha creato la riga fosse davvero committata, potrebbe non trovare nulla. `after_commit` garantisce che la riga sia salvata in modo durevole prima che venga arruolato qualsiasi cosa contro di essa.
  - **Perché `on: %i[create update]`, non solo `:create`.** Ricorda dall'episodio 4: un `Message` per una risposta assistant viene creato **vuoto** (`content: ''`, nel callback `before_message` di RubyLLM) e riempito con testo vero solo più tardi, tramite un `UPDATE` una volta che il modello finisce lo streaming. La sola creazione vedrebbe sempre e solo contenuto vuoto — l'update è dove il vero testo dell'assistant atterra davvero.
  - **Perché `content.present?`.** Senza, quella creazione a contenuto vuoto farebbe comunque scattare il callback, e `RubyLLM.embed("")` o darebbe errore o sprecherebbe una chiamata API per generare l'embedding di niente.
  - **Perché `saved_change_to_content?`, oltre a `content.present?`.** Questa è quella che non è ovvia. Ricorda dall'episodio 4 che `add_message` fa anche un `.update!(content_raw:)` incidentale subito dopo aver creato il messaggio dell'utente — un secondo `UPDATE` dove `content` in sé non cambia. Senza questo controllo, quell'update incidentale passerebbe il test `content.present?` (il contenuto c'è, solo invariato) e innescherebbe una seconda, sprecata chiamata di embedding per un testo che era già stato embeddato pochi istanti prima alla creazione. `saved_change_to_content?` è il modo di Rails stesso di chiedere "questo specifico attributo è davvero cambiato nel salvataggio appena committato" — vero alla creazione (nil → il testo del messaggio) e vero quando il placeholder dell'assistant viene riempito (`''` → la vera risposta), falso per quell'update incidentale che tocca solo `content_raw`. L'effetto netto: esattamente due chiamate di embedding per scambio, una per la domanda, una per la risposta, mai una terza.

## Il job in background

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

`Message.find_by` invece di `Message.find` — `find_by` restituisce `nil` per una riga mancante invece di sollevare un'eccezione, e `return unless message` tratta questo come un no-op normale e silenzioso invece che come un errore. Un messaggio potrebbe in linea di principio essere eliminato tra il momento in cui il suo job di embedding viene arruolato e il momento in cui gira davvero; non è un bug per cui valga la pena far crashare un job in background. `result.vectors` — l'array di 1.536 float da `RubyLLM::Embedding` — viene assegnato direttamente alla colonna `embedding`, e `neighbor`/pgvector gestiscono il vero formato di salvataggio sotto.

## Cercare

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

- `params[:q].to_s.strip` — `params[:q]` è `nil` alla prima visita, vuota, della pagina di ricerca (nessuna query ancora inviata); `.to_s` lo trasforma in `""` così `.strip` non deve mai proteggersi da un receiver `nil`. `return if @query.blank?` si ferma lì, prima di chiamare mai l'API di embedding, se non c'è nulla da cercare.
- `RubyLLM.embed(@query).vectors` — il testo del box di ricerca viene embeddato attraverso esattamente la stessa chiamata di ogni messaggio. È la parte che rende possibile la ricerca semantica in primo luogo: una domanda è confrontabile con le risposte salvate solo perché entrambe sono state posizionate nello stesso spazio numerico dallo stesso modello.
- `Message.where.not(embedding: nil)` — i messaggi creati prima di questo episodio (o prima che il proprio job di embedding finisse) hanno un `embedding` `nil`; escluderli impedisce a `nearest_neighbors` di provare mai a calcolare una distanza contro nulla.
- `.nearest_neighbors(:embedding, query_embedding, distance: "cosine")` — da `neighbor`. Ogni riga che questo restituisce ottiene un attributo `neighbor_distance` attaccato — un numero che non veniva affatto dalla tabella `messages`, calcolato da Postgres come parte della query e riletto sul record in memoria.
- `.first(10)` — prende i dieci più vicini.

Nessuna route per `new`/`create` e nessun `SearchController#index` — l'intera funzionalità è un'unica azione.

```ruby
# config/routes.rb
resource :search, only: [:show], controller: :search
```

```
Prefix Verb URI Pattern       Controller#Action
search GET  /search(.:format) search#show
```

Una `resource` singolare invece di `resources` — c'è esattamente una "pagina" di ricerca, non una collezione di record di ricerca con URL individuali, quindi la forma singolare (coerente con `resource :chat` degli episodi precedenti) è la forma corretta qui, non solo un modo più corto di scrivere la stessa cosa. Un'unica route `GET /search` gestisce sia la visita iniziale vuota sia una query inviata, distinte solo dalla presenza o meno di `?q=...` — un URL semplice, cliccabile, salvabile nei preferiti, che funziona con il tasto indietro, esattamente ciò che dovrebbe essere una pagina di risultati di ricerca.

## La pagina di ricerca

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

- `form_with url: search_path, method: :get, ...` — `method: :get`, non `:post` come ogni form precedente in questa serie, e di proposito: una ricerca è una lettura, non un cambio di stato, ed è un `GET` che rende l'URL dei risultati (`/search?q=cooking+dinner`) qualcosa che puoi copiare, salvare nei preferiti, o raggiungere col tasto indietro.
- `value="<%= @query %>"` — riempie di nuovo il box con qualunque cosa sia stata cercata, così ricaricare o condividere l'URL mostra la stessa query, non un campo vuoto accanto a risultati popolati.
- `1 - message.neighbor_distance` — distanza coseno e similarità coseno sono complementari (`distanza = 1 - similarità` per l'operatore coseno di pgvector); la distanza grezza dalla query viene qui ribaltata nell'inquadratura più intuitiva "quanto è simile" per la visualizzazione. Non aspettarti però che questo numero si avvicini a `1.0` per una corrispondenza forte — continua a leggere.

## Provarlo, e un numero che sembra più basso di quanto sembri

```bash
bin/dev
```

Chiedi prima qualche cosa non correlata in chat — una sull'arrostire un pollo, una sulla pasta, una sui polpi — così c'è qualcosa da cercare davvero. Poi visita `/search` e prova "cooking dinner". Il messaggio sul pollo e quello sulla pasta tornano per primi, ordinati correttamente sopra quello sui polpi, anche se "cooking dinner" non compare alla lettera in nessuno di essi.

I numeri di similarità in sé atterrano più bassi di quanto l'intuizione si aspetti — intorno a `0.28`–`0.30` per una corrispondenza chiaramente rilevante con `text-embedding-3-small`, non un `0.9` e rotti. Non è un bug o una corrispondenza debole; è solo come è fatto lo spazio vettoriale di questo specifico modello — le similarità coseno tra pezzi di testo **non correlati** con questo modello tendono comunque a stare ben sopra zero, quindi il segnale utile è il **ranking relativo** tra i risultati, non il numero assoluto che ottiene un risultato da solo.

## Cosa viene dopo

L'episodio 6 copre il tool calling: far chiamare al modello il codice Ruby dell'app stessa — non solo rispondere da quello che già sa, ma fare davvero qualcosa.
