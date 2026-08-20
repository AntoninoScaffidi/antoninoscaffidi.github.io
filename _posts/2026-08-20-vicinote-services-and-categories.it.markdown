---
layout: post
title: "VicinoTe: Servizi e categorie"
series: "vicinote"
episode: 3
lang: it
ref: vicinote-services-and-categories
permalink: /vicinote-services-and-categories/
canonical_url: https://antoninoscaffidi.github.io/it/vicinote-services-and-categories/
date: 2026-08-20 07:00:00 +0200
image: /assets/images/vicinote-ep3-banner.png
---

L'[episodio 2]({% post_url 2026-08-13-vicinote-authentication-with-rails-8 %}) aveva fatto funzionare gli account da capo a fondo — registrazione, accesso, disconnessione, reset della password — e si era fermato lì di proposito: nulla toccava ancora `Service` o `Booking`. Questo episodio è dove il marketplace inizia davvero a essere un marketplace: un utente autenticato può elencare qualcosa che offre, e chiunque può sfogliare cosa è elencato.

È anche dove l'associazione `has_many :services` dello schizzo di dominio dell'[episodio 1]({% post_url 2026-08-08-vicinote-project-setup-and-domain %}) viene finalmente scritta nel modello `User`. È rimasta in un post del blog come decisione di design per due episodi; oggi diventa codice vero.

Questo è un episodio lungo di proposito — l'obiettivo è che nulla nel diff resti inspiegato: ogni file generato, ogni riga aggiunta a mano, ogni opzione passata a ogni metodo.

Il codice è taggato [`episode-3`](https://github.com/AntoninoScaffidi/vicinote-tutorial/tree/episode-3) nel repo [vicinote-tutorial](https://github.com/AntoninoScaffidi/vicinote-tutorial).

## Cosa tocca questo episodio, in sintesi

Prima di andare file per file, ecco la mappa. Due nuovi modelli, un modello modificato, un nuovo controller, due nuove view, una modifica alle routes, un file di seed, e una correzione di traduzione di una riga:

```
db/migrate/..._create_categories.rb   nuovo — la tabella categories
db/migrate/..._create_services.rb     nuovo — la tabella services
app/models/category.rb                nuovo — Category: has_many :services, validates :name
app/models/service.rb                 nuovo — Service: belongs_to :user/:category, validazioni, accessor price
app/models/user.rb                    modifica — aggiunge has_many :services
app/controllers/services_controller.rb nuovo — index (pubblico), new, create (solo autenticati)
app/views/services/index.html.erb     nuovo — la pagina di elenco
app/views/services/new.html.erb       nuovo — il form "offri un servizio"
config/routes.rb                      modifica — resources :services, only: [:index, :new, :create]
config/locales/en.yml                 modifica — corregge un nome di attributo interno trapelato nei messaggi d'errore, imposta l'euro come valuta di default
db/seeds.rb                           modifica — la lista fissa di categorie
app/views/pages/home.html.erb         modifica — i due bottoni placeholder ora portano da qualche parte
```

## Generare i due modelli

```bash
bin/rails generate model Category "name:string:uniq"
bin/rails generate model Service title:string description:text price_cents:integer user:references category:references
```

Ogni coppia `campo:tipo` dopo il nome del modello diventa una colonna della migrazione. La sintassi ha qualche trucco in più che vale la pena spiegare, dato che entrambi i comandi li usano:

- `name:string:uniq` — il terzo segmento, `:uniq`, non è un tipo. Dice al generator di aggiungere anche un indice univoco su quella colonna, così scrive una chiamata `add_index` per noi invece di doverci ricordare di aggiungerne una più tardi.
- `user:references` e `category:references` — `references` non è nemmeno un tipo di colonna. È una scorciatoia del generator che significa "questo modello appartiene a quello." Si espande in una chiamata `t.references` nella migrazione (una colonna integer di foreign key più un indice), **e** fa sì che il generator scriva direttamente una riga `belongs_to :user` / `belongs_to :category` nel file del modello generato. Per questo il modello `Service` ha già entrambe le associazioni dentro nel momento in cui il generator finisce — non abbiamo scritto `belongs_to` a mano noi.

Ogni comando stampa cosa ha creato:

```
      create    db/migrate/20260820055929_create_categories.rb
      create    app/models/category.rb
      invoke    test_unit
      create      test/models/category_test.rb
      create      test/fixtures/categories.yml
```

```
      create    db/migrate/20260820055946_create_services.rb
      create    app/models/service.rb
      invoke    test_unit
      create      test/models/service_test.rb
      create      test/fixtures/services.yml
```

I file `test_unit` sono l'impalcatura di test predefinita di Rails (una classe di test vuota e un file di fixture vuoto) — questa serie non li usa, quindi restano come generati e non vengono discussi oltre.

## La migrazione categories, riga per riga

Generata, poi modificata a mano per aggiungere una parola:

```ruby
# db/migrate/..._create_categories.rb
class CreateCategories < ActiveRecord::Migration[8.1]
  def change
    create_table :categories do |t|
      t.string :name, null: false

      t.timestamps
    end
    add_index :categories, :name, unique: true
  end
end
```

- `class CreateCategories < ActiveRecord::Migration[8.1]` — ogni migrazione è una sottoclasse di `ActiveRecord::Migration`, versionata sulla release di Rails che l'ha generata (`[8.1]`). Quella versione fissata è ciò che permette a Rails di cambiare il comportamento del DSL delle migrazioni tra versioni major senza rompere silenziosamente le migrazioni scritte per una più vecchia.
- `def change` — l'unico metodo di cui Rails ha bisogno. Per operazioni che può invertire automaticamente (creare una tabella, aggiungere una colonna, aggiungere un indice), `change` basta; Rails deduce come annullarla se mai fai rollback. Le operazioni irreversibili avrebbero bisogno di metodi `up`/`down` separati — qui non serve nulla del genere.
- `create_table :categories do |t|` — apre la definizione della tabella; `t` è l'oggetto su cui viene definita ogni colonna.
- `t.string :name, null: false` — una colonna `VARCHAR`. Il generator aveva scritto `t.string :name`; il `null: false` è l'unica parola che abbiamo aggiunto a mano, con lo stesso rigore che l'episodio 2 aveva messo nella tabella `users`. Senza, Postgres salverebbe volentieri una categoria senza alcun nome, e non è uno stato che l'app ha motivo di usare.
- `t.timestamps` — scorciatoia per due colonne, `created_at` e `updated_at`, entrambe `datetime`, entrambe riempite automaticamente da ActiveRecord alla creazione/modifica. Quasi ogni tabella di questa serie ha questa riga; `Category` non fa eccezione.
- `add_index :categories, :name, unique: true` — questo è ciò che `:uniq` nel comando del generator ha prodotto. È una garanzia di unicità a **livello di database**, imposta da Postgres stesso, non solo da una validazione Rails che in teoria potrebbe essere aggirata da due richieste simultanee in corsa tra loro. `Category` valida l'unicità anche nel modello (sotto) — cintura e bretelle: la validazione del modello dà un errore amichevole nel caso normale, l'indice garantisce la correttezza anche in caso di corsa critica.

## La migrazione services, riga per riga

```ruby
# db/migrate/..._create_services.rb
class CreateServices < ActiveRecord::Migration[8.1]
  def change
    create_table :services do |t|
      t.string :title, null: false
      t.text :description, null: false
      t.integer :price_cents, null: false
      t.references :user, null: false, foreign_key: true
      t.references :category, null: false, foreign_key: true

      t.timestamps
    end
  end
end
```

- `t.string :title, null: false` — stessa forma di `Category#name`: un campo di testo breve obbligatorio.
- `t.text :description, null: false` — `text` invece di `string`. In Postgres è quasi solo una formalità (entrambi mappano allo stesso tipo `text` illimitato sotto il cofano; la distinzione `string` vs `text` di Rails è soprattutto un suggerimento a livello Rails, non una differenza di storage in Postgres), ma segnala un'intenzione: questa colonna contiene un paragrafo, non un'etichetta, e alcuni helper del form (`form.text_area` invece di `form.text_field`) si basano esattamente su questo tipo più avanti.
- `t.integer :price_cents, null: false` — un semplice intero. C'è un'intera sezione più sotto sul perché è un intero e non un `decimal`.
- `t.references :user, null: false, foreign_key: true` — questa riga è ciò che `user:references` sulla riga di comando ha generato, e fa tre cose insieme:
  1. Aggiunge una colonna integer chiamata `user_id` (il suffisso `_id` e la trasformazione da plurale a singolare sono entrambe convenzioni Rails, non qualcosa che abbiamo scritto noi).
  2. `foreign_key: true` — aggiunge un vero e proprio vincolo di foreign key Postgres da `services.user_id` a `users.id`. Il database stesso ora rifiuterà di inserire una riga `Service` il cui `user_id` non corrisponde a un utente reale, e rifiuterà di eliminare una riga `User` che ha ancora servizi che puntano a essa (a meno che l'associazione non dica diversamente — altro su questo più sotto).
  3. Un indice su `user_id`, aggiunto automaticamente, perché una colonna di foreign key non indicizzata rende lenta ogni query che fa join attraverso di essa man mano che la tabella cresce. Questo non era opzionale né qualcosa che abbiamo chiesto — `t.references` indicizza sempre.
  `null: false` qui significa che ogni servizio **deve** appartenere a qualcuno; non esiste un servizio senza un fornitore.
- `t.references :category, null: false, foreign_key: true` — forma identica, per l'altro lato della relazione.

Nota che nessuna delle due migrazioni menziona `belongs_to` o `has_many` — le migrazioni descrivono solo lo **schema del database** (tabelle, colonne, vincoli, indici). Le associazioni sono un concetto separato, a livello Ruby, dichiarato nei modelli, che è la prossima sezione.

## Eseguire le migrazioni

```bash
bin/rails db:migrate
```

Questo esegue entrambe le migrazioni in sospeso in ordine di timestamp (categories, poi services — la foreign key di services verso categories ha bisogno che la tabella categories esista già) e riscrive `db/schema.rb` per riflettere il nuovo stato. `schema.rb` non è qualcosa da modificare a mano; è l'istantanea in cache di Rails di "com'è fatto il database adesso", rigenerata ogni volta che fai una migrazione, ed è ciò che legge il `bin/rails db:setup` di un collega per costruire un database nuovo che corrisponda al tuo senza rieseguire ogni migrazione mai scritta.

## I modelli generati, prima di qualsiasi modifica

Subito dopo che il generator ha girato, prima di toccare qualsiasi cosa:

```ruby
# app/models/category.rb
class Category < ApplicationRecord
end
```

```ruby
# app/models/service.rb
class Service < ApplicationRecord
  belongs_to :user
  belongs_to :category
end
```

`Category` è vuota perché non ha colonne `references` — non c'era nulla da cui il generator potesse dedurre un'associazione. `Service` ha già entrambe le righe `belongs_to` per via dei campi `:references` passati sulla riga di comando, come spiegato sopra. Entrambe ereditano da `ApplicationRecord`, la classe base astratta che ogni modello in un'app Rails condivide (essa stessa una sottile sottoclasse di `ActiveRecord::Base`), che è ciò che dà loro `.find`, `.create`, `.where`, le validazioni, e tutto il resto che ActiveRecord fornisce — niente di tutto ciò è scritto in nessuno dei due file, viene ereditato.

Una cosa che vale la pena nominare esplicitamente: da Rails 5 in poi, `belongs_to` è **obbligatorio per default**. Scrivere `belongs_to :user` non dichiara solo l'associazione, aggiunge implicitamente anche una validazione di presenza — un `Service` senza `user` fallisce la validazione, oltre al fatto che il database lo rifiuta già via `null: false, foreign_key: true` dalla migrazione. Due livelli indipendenti che impongono la stessa regola, a due livelli diversi (validazione Ruby vs vincolo SQL), che è esattamente il messaggio di errore "Category must exist" che vedrai più avanti in questo post — quella frase è il testo di default di Rails per un controllo di presenza fallito su `category` da un `belongs_to`.

## Category, completato

```ruby
# app/models/category.rb
class Category < ApplicationRecord
  has_many :services

  validates :name, presence: true, uniqueness: true
end
```

- `has_many :services` — l'altra metà di `Service belongs_to :category`. Le associazioni Rails si dichiarano su entrambi i lati a mano; non c'è modo di dichiarare solo un lato e far dedurre l'altro. Questo è ciò che fa funzionare `some_category.services` — l'associazione cerca ogni riga `Service` il cui `category_id` corrisponde, tramite la foreign key che la migrazione ha creato.
- `validates :name, presence: true, uniqueness: true` — `presence: true` duplica ciò che il `null: false` nella migrazione già garantisce a livello di database, ma per un pubblico diverso: un vincolo di database fallito solleva una brutta eccezione `ActiveRecord::NotNullViolation`, mentre una validazione di presenza fallita dà a `@category.errors` un messaggio amichevole che una view può renderizzare. `uniqueness: true` è la metà a livello di modello della coppia cintura-e-bretelle con l'indice unico della migrazione — questa esegue una `SELECT` prima di salvare e dà un errore pulito; l'indice è ciò che impedisce davvero a un duplicato di raggiungere mai la tabella se due richieste sono in corsa critica oltre la validazione nello stesso istante.

## Service, completato

```ruby
# app/models/service.rb
class Service < ApplicationRecord
  belongs_to :user
  belongs_to :category

  validates :title, presence: true
  validates :description, presence: true
  validates :price_cents, presence: true, numericality: { greater_than: 0, only_integer: true }

  # price_cents è ciò che viene salvato e confrontato (niente sorprese di
  # arrotondamento con i float), ma nessuno vuole scrivere i centesimi in
  # un form. Questo parla in euro in entrata e in uscita, così il campo
  # del form può semplicemente essere "price".
  def price
    price_cents && price_cents / 100.0
  end

  def price=(value)
    self.price_cents = value.present? ? (value.to_f * 100).round : nil
  end
end
```

- `validates :title, presence: true` / `validates :description, presence: true` — stesso ragionamento di `Category#name`: il `null: false` della migrazione è l'ultima linea di difesa, questa è quella amichevole che gira per prima.
- `validates :price_cents, presence: true, numericality: { greater_than: 0, only_integer: true }` — tre controlli in un'unica riga. `presence: true` rifiuta `nil`. `numericality:` da sola rifiuterebbe solo qualcosa che non è affatto un numero (una stringa come `"abc"`); l'opzione `greater_than: 0` aggiunge la regola di business che un servizio gratuito non è qualcosa che questa validazione permette, e `only_integer: true` rifiuta centesimi frazionari (`4250.5`), cosa che non dovrebbe comunque essere raggiungibile dato che `price_cents` è una colonna database `integer`, ma la validazione rende la regola esplicita invece di affidarsi solo al tipo di colonna per imporla prima che il valore raggiunga il database.
- `def price` / `def price=` — trattati per intero nella prossima sezione; è il pezzo che permette a un form di parlare in euro mentre la colonna sotto resta in centesimi.

## Una decisione su cui vale la pena soffermarsi: price_cents, non price

Il generator ha scritto `price_cents:integer`, non `price:decimal`, sulla riga di comando — quella formulazione è stata scelta deliberatamente in partenza, ed è la stessa categoria di decisione che l'episodio 1 aveva preso su `Booking` che salva il proprio prezzo invece di leggere `service.price` dal vivo: il denaro gestito come float, o persino come `decimal` ingenuo, prima o poi produce un errore di arrotondamento che si presenta come qualche centesimo di scarto su una fattura, nel momento peggiore possibile. Salvare l'importo come numero intero di centesimi evita del tutto questa categoria di bug — non c'è una parte frazionaria da arrotondare, perché non c'è nessuna frazione. 42,50 € viene salvato come l'intero `4250`, punto. Anche chiamarla `price_cents` invece di, per esempio, `price_usd_cents` è deliberato — "cents" è l'unità minore dell'euro tanto quanto lo è del dollaro, quindi nulla nella colonna stessa è legato a una valuta particolare; solo il livello di formattazione, trattato più avanti in questo post, sa quale valuta è in uso.

Il costo è che nessuno vuole scrivere "4250" in un form intendendo 42,50 €. Quindi `Service` ottiene un piccolo accessor virtuale — due normali metodi Ruby, non una colonna del database — che traduce gli euro in centesimi e viceversa:

```ruby
def price
  price_cents && price_cents / 100.0
end

def price=(value)
  self.price_cents = value.present? ? (value.to_f * 100).round : nil
end
```

Passando in rassegna entrambi:

- `def price` — legge per la visualizzazione. `price_cents && price_cents / 100.0` usa il `&&` di Ruby per il suo comportamento di short-circuit, non come controllo booleano: se `price_cents` è `nil` (un `Service` nuovo di zecca, non ancora salvato), l'intera espressione va in short-circuit a `nil` senza tentare `nil / 100.0`, che solleverebbe un'eccezione. Se è un intero vero, `&&` valuta e restituisce il lato destro — la divisione. Dividere per `100.0` (un literal float, non `100`) forza Ruby a fare una divisione in virgola mobile invece che una divisione intera, così `4250 / 100.0` dà `42.5`, non `42` troncato.
- `def price=(value)` — il setter, chiamato automaticamente ogni volta che qualcosa fa `service.price = "42.50"` o, altrettanto automaticamente, ogni volta che un form invia un campo chiamato `price` tramite mass assignment (`Service.new(price: "42.50", ...)`) — Rails chiama il metodo setter per ogni attributo permesso, non gli importa se quel metodo corrisponde a una colonna reale o no. `value.present?` protegge da input vuoto (una stringa vuota da un campo del form svuotato) invece di tentare di convertire `""` in un numero. Quando c'è davvero un valore, `value.to_f * 100` converte in euro-come-float e moltiplica per 100 per ottenere i centesimi, e `.round` lo riporta a un numero intero — `.to_f` su input utente può produrre cose come `42.499999999999996` per la normale imprecisione della virgola mobile, e `.round` è ciò che ripulisce tutto questo a esattamente `4250` prima che raggiunga mai `price_cents=`.

Poiché `price=` è un normale metodo Ruby e non un attributo sostenuto da ActiveRecord, gira immediatamente quando gli attributi vengono assegnati — prima di `save`, prima di qualsiasi validazione. Nel momento in cui `validates :price_cents, ...` gira, `price_cents` è già stato popolato da questo setter. Il campo del form discusso più avanti in questo post è solo `form.text_field :price` — la view non ha mai idea che esistano i centesimi.

## Le categorie sono seminate, non create

Un marketplace dove chiunque può inventare una nuova categoria finisce con cinquanta categorie che significano la stessa cosa, scritte in cinque modi diversi, e sfogliare per categoria smette di essere utile. VicinoTe cura invece una lista fissa, in `db/seeds.rb`:

```ruby
# db/seeds.rb
[
  "Home Repair",
  "Tutoring",
  "Cleaning",
  "Gardening",
  "Pet Care",
  "Moving Help",
  "Tech Support",
  "Beauty & Wellness"
].each do |name|
  Category.find_or_create_by!(name: name)
end
```

- L'array letterale di otto stringhe **è** la lista delle categorie, in puro Ruby, proprio lì nel file di seed — nessuna interfaccia di amministrazione, nessun formato di configurazione separato, solo un array sotto controllo di versione che un episodio futuro potrebbe estendere aggiungendo una nona stringa.
- `.each do |name| ... end` itera l'array una volta per categoria.
- `Category.find_or_create_by!(name: name)` — questo singolo metodo fa due lavori a seconda di cosa trova: se esiste già una `Category` con quel `name`, la restituisce intatta; se no, ne costruisce e ne salva una nuova. Il `!` finale significa che solleva `ActiveRecord::RecordInvalid` in caso di validazione fallita invece di restituire silenziosamente un record non salvato e non valido — cosa che conta qui, perché un file di seed che fallisce rumorosamente è molto più facile da debuggare di uno che fallisce silenziosamente e lascia il database seminato a metà.

Quella combinazione — un array fisso più `find_or_create_by!` — è ciò che rende il file sicuro da eseguire più di una volta. Rieseguire `db:seed` dopo aver aggiunto una nona categoria all'array più avanti crea solo quella nuova; le prime otto vengono trovate per nome e lasciate intatte, non duplicate.

```bash
bin/rails db:migrate
bin/rails db:seed
```

`db:seed` esegue semplicemente `db/seeds.rb` come un normale script Ruby dentro l'environment dell'app — niente di più misterioso di così.

## Scrivere l'associazione che l'episodio 1 aveva solo schizzato

Il design del dominio dell'episodio 1 mostrava questo codice come illustrazione della decisione "il ruolo emerge dall'associazione" — l'intero punto era che uno `User` non è etichettato con una colonna `role: "provider"`, è un fornitore **perché** ha dei servizi. Non era mai stato davvero nel modello `User`, però; l'episodio 2 riguardava l'autenticazione e non l'ha toccato. Ci va ora:

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password
  has_many :sessions, dependent: :destroy
  has_many :services, dependent: :destroy

  normalizes :email_address, with: ->(e) { e.strip.downcase }
end
```

È cambiata solo una riga — `has_many :services, dependent: :destroy` è stata aggiunta, tutto il resto (`has_secure_password`, l'associazione `sessions`, la normalizzazione dell'email) è invariato dall'episodio 2.

`dependent: :destroy` corrisponde a quello che `sessions` fa già nella riga sopra, per lo stesso motivo: senza, eliminare uno `User` fallirebbe direttamente (il vincolo `foreign_key: true` del database dalla migrazione rifiuterebbe l'eliminazione, dato che le righe `services` punterebbero ancora a uno `user_id` che sta per smettere di esistere) oppure, se il vincolo fosse allentato, lascerebbe righe `Service` orfane nella tabella per sempre, che puntano a nessuno. `dependent: :destroy` dice a Rails di eliminare prima ogni `Service` associato, automaticamente, ogni volta che uno `User` viene distrutto — la pulizia è una parola, non qualcosa che ogni futuro punto di chiamata deve ricordarsi di fare a mano.

## Il controller: index pubblico, tutto il resto protetto

```ruby
# app/controllers/services_controller.rb
class ServicesController < ApplicationController
  allow_unauthenticated_access only: :index

  def index
    @services = Service.includes(:category, :user).order(created_at: :desc)
  end

  def new
    @service = Current.user.services.new
  end

  def create
    @service = Current.user.services.new(service_params)

    if @service.save
      redirect_to services_path, notice: "Your service is live."
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  def service_params
    params.require(:service).permit(:title, :description, :price, :category_id)
  end
end
```

Riga per riga:

- `class ServicesController < ApplicationController` — ogni controller in un'app Rails eredita da `ApplicationController`, che è dove è incluso il concern `Authentication` dell'episodio 2. È questo che rende significativa la riga successiva.
- `allow_unauthenticated_access only: :index` — il concern `Authentication` dell'episodio 2 esegue `before_action :require_authentication` per ogni azione su ogni controller di default, il che significa che **senza questa riga**, un visitatore anonimo che raggiunge una qualsiasi azione qui verrebbe reindirizzato dritto alla pagina di accesso. È corretto per `new` e `create` — nessuno dovrebbe poter elencare un servizio in anonimo — ma sbagliato per `index`: sfogliare il marketplace deve funzionare anche per chi non si è ancora registrato, altrimenti non c'è motivo di registrarsi. `only: :index` esclude di nuovo quella singola azione dal requisito lasciando `new` e `create` protette. È esattamente lo stesso meccanismo che l'episodio 2 aveva usato, senza alcun argomento (`allow_unauthenticated_access`, senza `only:`), per rendere pubblico l'intero `PagesController` — qui vogliamo esentare solo una delle tre azioni, quindi `only:` restringe.
- `def index` / `@services = Service.includes(:category, :user).order(created_at: :desc)` — `Service.includes(:category, :user)` non sta filtrando nulla; `includes` è il metodo di eager-loading di ActiveRecord. Senza, una view che scorre `@services` e chiama `service.category.name` e `service.user.email_address` per ognuno farebbe scattare una `SELECT` aggiuntiva per ogni servizio, per ogni associazione — il classico problema delle query N+1, dove mostrare 50 servizi significa silenziosamente 101 query (1 per la lista, più 50 per le categorie, più 50 per gli utenti) invece di 3. `includes` carica le categorie e gli utenti di tutti i servizi in anticipo, in un piccolo numero fisso di query aggiuntive, indipendentemente da quanti servizi ci siano. `.order(created_at: :desc)` ordina dal più recente, così un nuovo annuncio compare in cima alla pagina invece che in fondo.
- `def new` / `@service = Current.user.services.new` — questo è il ritorno dell'aver collegato `has_many :services` a `User` una sezione fa. `Current.user` è l'utente attualmente autenticato (dal modello `Current` dell'episodio 2, una sottoclasse di `ActiveSupport::CurrentAttributes`). Chiamare `.services.new` **attraverso** l'associazione, invece di `Service.new` da solo, pre-riempie automaticamente lo `user_id` del nuovo record con `Current.user.id` — l'istanza che questa riga costruisce sa già a chi appartiene prima ancora che un solo campo del form sia stato compilato.
- `def create` — `Current.user.services.new(service_params)` fa la stessa costruzione scoped-all'associazione di `new`, ma questa volta passando anche i dati del form inviati. Poiché il lato `user` dell'associazione è impostato da `Current.user.services`, non da nulla in `service_params`, non c'è nessun campo `user_id` da nessuna parte nei params permessi sotto che un visitatore malintenzionato potrebbe manomettere per rivendicare un annuncio a nome di qualcun altro — il fornitore è chiunque dica `Current.user`, punto, non quello che potrebbe dichiarare un campo nascosto del form.
- `if @service.save` — `save` esegue le validazioni e, se passano tutte, effettua l'`INSERT` e restituisce `true`; se una validazione fallisce, non fa nulla al database e restituisce `false` — nessuna eccezione sollevata, per questo è un semplice `if`, non un `begin/rescue`.
- `redirect_to services_path, notice: "Your service is live."` — in caso di successo, un vero redirect HTTP verso l'index (questo è lo stesso pattern Post/Redirect/Get discusso più a fondo nell'episodio 3 di *ai-with-ruby*: reindirizzare dopo una POST che cambia lo stato significa che ricaricare la pagina di risultato non reinvia mai il form). `notice:` salva un messaggio one-time nel `flash` — sopravvive esattamente a un redirect e poi si cancella da solo, il che è come "Your service is live." compare al caricamento della pagina immediatamente successiva e sparisce se ricarichi di nuovo.
- `render :new, status: :unprocessable_entity` — in caso di fallimento, ri-renderizza lo **stesso** form (nessun redirect da nessuna parte) così `@service` — ora popolato sia con quello che l'utente ha scritto sia con gli errori di validazione ad esso attaccati — può essere mostrato di nuovo con i suoi dati intatti e i problemi specifici evidenziati. `status: :unprocessable_entity` imposta lo status HTTP a 422 invece del 200 di default; il browser renderizza comunque l'HTML in entrambi i casi, ma un 422 dice correttamente a qualsiasi strumento che osserva la risposta (devtools del browser, Turbo, una suite di test) che questo era un invio fallito, non un caricamento di pagina riuscito che per caso contiene un form.
- `private` — tutto sotto questa riga è chiamabile solo da dentro il controller stesso, non raggiungibile come route.
- `def service_params` / `params.require(:service).permit(:title, :description, :price, :category_id)` — gli strong parameters di Rails. `params.require(:service)` solleva immediatamente un'eccezione se i dati del form inviati non hanno affatto una chiave `service` di primo livello (una richiesta malformata o mancante), e `.permit(...)` è una allowlist: solo queste quattro chiavi sono permesse; qualsiasi altra cosa presente nei params grezzi della richiesta — incluso, per esempio, uno `user_id` che qualcuno ha provato a iniettare modificando l'HTML del form nel proprio browser prima di inviare — viene silenziosamente rimossa e non raggiunge mai `Service.new`. Nota che questo permette `:price`, non `:price_cents` — il controller parla con la stessa interfaccia rivolta agli euro del form; non sa nemmeno lui che `price_cents` esiste.

## Le routes e i due bottoni placeholder

```ruby
# config/routes.rb
resources :services, only: [:index, :new, :create]
```

`resources :services` è la scorciatoia di routing RESTful di Rails — da sola genererebbe tutte e sette le route convenzionali (`index`, `new`, `create`, `show`, `edit`, `update`, `destroy`). `only: [:index, :new, :create]` la restringe esattamente alle tre che questo episodio implementa. Eseguire `bin/rails routes -g services` mostra esattamente cosa è stato generato:

```
     Prefix Verb URI Pattern             Controller#Action
   services GET  /services(.:format)     services#index
            POST /services(.:format)     services#create
new_service GET  /services/new(.:format) services#new
```

La colonna `Prefix` è da dove vengono i nomi degli helper dei path come `services_path` e `new_service_path` — usati per tutto il controller e le view in questo episodio: Rails li deriva automaticamente dal prefisso più `_path` o `_url`. Non c'è ancora una route `show`, di proposito — non c'è una pagina del singolo servizio a cui linkare finché un episodio successivo non ne costruisce una.

La landing page dell'episodio 1 era stata spedita con due bottoni che non facevano nulla di proposito, stilizzati per sembrare disabilitati:

```erb
<span class="rounded-md bg-indigo-600 px-4 py-2 text-white font-medium opacity-60 cursor-not-allowed">
  Browse services
</span>
<span class="rounded-md border border-gray-300 px-4 py-2 text-gray-700 font-medium opacity-60 cursor-not-allowed">
  Offer a service
</span>
```

Finalmente portano da qualche parte:

```erb
<%= link_to "Browse services", services_path, class: "rounded-md bg-indigo-600 px-4 py-2 text-white font-medium hover:bg-indigo-700" %>
<%= link_to "Offer a service", authenticated? ? new_service_path : new_registration_path, class: "rounded-md border border-gray-300 px-4 py-2 text-gray-700 font-medium hover:bg-gray-50" %>
```

Entrambi i placeholder `<span>` sono diventati chiamate `link_to`, e le classi `opacity-60 cursor-not-allowed` che segnalavano visivamente "non ancora cliccabile" sono sparite insieme allo stato disabilitato stesso.

- `link_to "Browse services", services_path, ...` — incondizionato. L'index è pubblico (`allow_unauthenticated_access only: :index` dal controller), quindi questo link ha sempre senso, autenticati o no.
- `link_to "Offer a service", authenticated? ? new_service_path : new_registration_path, ...` — la destinazione stessa è un'espressione ternaria. `authenticated?` è l'`helper_method` che il concern `Authentication` dell'episodio 2 espone alle view (è solo `resume_session`, riusato per rispondere a "qualcuno è autenticato" senza innescare un redirect come farebbe `require_authentication`). Da autenticati, il link va dritto a `new_service_path` — il form. Da disconnessi, va a `new_registration_path` — registrazione — invece che direttamente al form del servizio, che li rimbalzerebbe comunque immediatamente alla pagina di accesso via `require_authentication`. Stessa destinazione finale in entrambi i casi; solo un redirect in meno per il caso comune di un visitatore che non si è ancora autenticato.

## Il form "offri un servizio"

```erb
<%# app/views/services/new.html.erb %>
<div class="max-w-xl mx-auto">
  <h1 class="text-3xl font-bold text-gray-900">Offer a service</h1>
  <p class="mt-2 text-gray-600">Tell your neighbours what you can help with.</p>

  <%= form_with model: @service, url: services_path, class: "mt-8 space-y-5" do |form| %>
    <% if @service.errors.any? %>
      <div class="rounded-md bg-red-50 px-4 py-3 text-sm text-red-700">
        <ul class="list-disc list-inside">
          <% @service.errors.full_messages.each do |message| %>
            <li><%= message %></li>
          <% end %>
        </ul>
      </div>
    <% end %>

    <div>
      <%= form.label :title, class: "block text-sm font-medium text-gray-700" %>
      <%= form.text_field :title, placeholder: "Guitar lessons for beginners", class: "mt-1 block w-full rounded-md border border-gray-300 px-3 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500" %>
    </div>

    <div>
      <%= form.label :category_id, "Category", class: "block text-sm font-medium text-gray-700" %>
      <%= form.collection_select :category_id, Category.order(:name), :id, :name, { prompt: "Choose a category" }, class: "mt-1 block w-full rounded-md border border-gray-300 px-3 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500" %>
    </div>

    <div>
      <%= form.label :description, class: "block text-sm font-medium text-gray-700" %>
      <%= form.text_area :description, rows: 4, placeholder: "What you offer, your experience, anything a neighbour should know.", class: "mt-1 block w-full rounded-md border border-gray-300 px-3 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500" %>
    </div>

    <div>
      <%= form.label :price, "Price (EUR)", class: "block text-sm font-medium text-gray-700" %>
      <%= form.text_field :price, placeholder: "45.00", inputmode: "decimal", class: "mt-1 block w-40 rounded-md border border-gray-300 px-3 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500" %>
    </div>

    <%= form.submit "Publish", class: "rounded-md bg-indigo-600 px-4 py-2 text-white font-medium hover:bg-indigo-700 cursor-pointer" %>
  <% end %>
</div>
```

- `form_with model: @service, url: services_path, ...` — `model: @service` è ciò che fa funzionare questo unico form sia per un `Service` nuovo di zecca, non salvato (da `ServicesController#new`), sia, più avanti, per uno che non è riuscito a salvarsi e viene ri-renderizzato con gli errori attaccati (dal `render :new` di `#create`) — Rails ispeziona se il record è persistito per decidere il metodo HTTP del form, anche se, dato che le routes definiscono solo `create`, questo form deve solo fare POST. `url: services_path` è esplicito qui invece di essere lasciato dedurre, e punta l'invio alla route `services#create` in ogni caso.
- `<% if @service.errors.any? %> ... @service.errors.full_messages.each ... <% end %>` — dopo una `create` fallita, `@service` porta con sé sia quello che l'utente ha scritto **sia** i fallimenti di validazione specifici ad esso attaccati da `save`. `errors.full_messages` li trasforma in stringhe pronte da leggere come "Title can't be blank" — questo è esattamente il blocco che ha renderizzato le liste di errori mostrate più avanti in questo post.
- `form.label :title, ...` / `form.text_field :title, ...` — un normale input di testo con etichetta. `form.label :title` senza un testo esplicito deduce "Title" dal nome dell'attributo automaticamente (tramite lo stesso meccanismo `humanize` discusso nella sezione I18n sotto); `placeholder:` è solo un attributo HTML passato direttamente.
- `form.collection_select :category_id, Category.order(:name), :id, :name, { prompt: "Choose a category" }, class: "..."` — questo è l'unico helper nel form su cui vale la pena soffermarsi, perché prende cinque argomenti separati prima dell'hash di opzioni HTML:
  1. `:category_id` — l'attributo impostato all'invio (corrisponde alla colonna di foreign key di `belongs_to :category`).
  2. `Category.order(:name)` — la collezione di record da cui costruire i tag `<option>`, ordinata alfabeticamente così il menu a tendina non elenca le categorie nell'ordine in cui sono state seminate.
  3. `:id` — il metodo **valore**: per ogni `Category` nella collezione, chiama `.id` per ottenere ciò che viene effettivamente inviato come `category_id`.
  4. `:name` — il metodo **testo**: chiama `.name` per ottenere ciò che viene mostrato a un umano dentro il menu.
  5. `{ prompt: "Choose a category" }` — opzioni per il select stesso; `prompt:` inserisce un'opzione placeholder disabilitata e non selezionata con quel testo, così il menu non finisce silenziosamente per selezionare di default la prima categoria della lista se qualcuno invia senza toccarlo.
  Solo dopo questo quinto argomento appare il normale hash di attributi HTML `class: "..."` — una chiamata a `collection_select` separa sempre "come costruire le opzioni" da "quali attributi HTML mettere sul tag `<select>`" in questo modo.
- `form.label :price, "Price (EUR)", ...` — qui il testo dell'etichetta **è** dato esplicitamente ("Price (EUR)"), sovrascrivendo ciò che `humanize` dal nome dell'attributo avrebbe prodotto ("Price"), perché il form deve comunicare la valuta e il nome dell'attributo del modello non ha modo di portarla da solo.
- `form.text_field :price, placeholder: "45.00", inputmode: "decimal", ...` — questo è il campo che chiama `Service#price=` all'invio, non `price_cents=` — la view davvero non menziona mai i centesimi da nessuna parte. `inputmode: "decimal"` è un normale attributo HTML (niente di specifico di Rails) che suggerisce alle tastiere mobili di mostrare un tastierino numerico con un punto decimale invece della tastiera alfabetica completa.
- `form.submit "Publish", ...` — renderizza un `<input type="submit">` con l'etichetta data; `cursor-pointer` è puramente estetico, dato che un bottone submit è cliccabile di default ma i browser non sempre renderizzano il cursore a puntatore su di esso senza che venga detto loro di farlo.

## L'elenco dei servizi

```erb
<%# app/views/services/index.html.erb %>
<div class="max-w-3xl mx-auto w-full">
  <% if notice = flash[:notice] %>
    <p class="py-2 px-3 bg-green-50 mb-5 text-green-700 font-medium rounded-lg inline-block" id="notice"><%= notice %></p>
  <% end %>

  <div class="flex items-center justify-between">
    <h1 class="text-3xl font-bold text-gray-900">Services near you</h1>
    <% if authenticated? %>
      <%= link_to "Offer a service", new_service_path, class: "rounded-md bg-indigo-600 px-4 py-2 text-white font-medium hover:bg-indigo-700" %>
    <% end %>
  </div>

  <% if @services.none? %>
    <p class="mt-8 text-gray-500">No services yet — be the first to offer one.</p>
  <% else %>
    <div class="mt-8 space-y-4">
      <% @services.each do |service| %>
        <div class="rounded-lg border border-gray-200 p-5">
          <div class="flex items-start justify-between gap-4">
            <div>
              <p class="text-xs font-semibold uppercase tracking-wide text-indigo-600"><%= service.category.name %></p>
              <h2 class="mt-1 text-lg font-semibold text-gray-900"><%= service.title %></h2>
              <p class="mt-1 text-sm text-gray-500">by <%= service.user.email_address %></p>
            </div>
            <p class="whitespace-nowrap text-lg font-semibold text-gray-900">
              <%= number_to_currency(service.price) %>
            </p>
          </div>
          <p class="mt-3 text-gray-600"><%= service.description %></p>
        </div>
      <% end %>
    </div>
  <% end %>
</div>
```

- `<% if notice = flash[:notice] %> ... <% end %>` — un singolo `=` di proposito, un'assegnazione usata come condizione, non un confronto `==`. `flash[:notice]` viene letto una volta, assegnato a una locale `notice`, e quella stessa locale viene riusata dentro il blocco — questo è esattamente il pattern già usato in `sessions/new.html.erb` dall'episodio 2, tenuto coerente qui invece di introdurre un idioma diverso. È ciò che fa comparire davvero sullo schermo il messaggio "Your service is live." dal `redirect_to ..., notice: "..."` del controller — nulla renderizza `flash` automaticamente da nessuna parte in questo layout, ogni view che vuole mostrarlo lo fa esplicitamente.
- `<% if authenticated? %> ... <% end %>` intorno al bottone "Offer a service" — un visitatore disconnesso che sfoglia l'index pubblico vede gli annunci ma non un invito a pubblicarne uno; il bottone appare solo una volta che qualcuno è effettivamente autenticato.
- `<% if @services.none? %> ... <% else %> ... <% end %>` — `.none?` è una normale query ActiveRecord/Enumerable, vera quando la relation ha zero record. Questo è il messaggio di stato vuoto, mostrato al posto di una pagina vuota senza spiegazione la prima volta che l'app gira senza ancora nessun annuncio.
- `<% @services.each do |service| %>` — itera la relation caricata in anticipo dal controller. Poiché `Service.includes(:category, :user)` ha già portato ogni categoria e ogni utente in memoria insieme ai servizi stessi, `service.category.name` e `service.user.email_address` dentro questo loop non innescano nessuna query aggiuntiva — questo è il ritorno della chiamata `includes` discussa nella sezione del controller, che atterra qui, nella view, dove l'N+1 altrimenti scatterebbe davvero.
- `service.category.name`, `service.title`, `service.user.email_address`, `service.description` — normali letture di attributi e associazioni.
- `number_to_currency(service.price)` — un helper di view di Rails (da `ActionView::Helpers::NumberHelper`) che formatta un numero semplice come valuta: `42.5` diventa `"€42.50"` — il numero corretto di due decimali e il simbolo della valuta, non qualsiasi numero di cifre abbia per caso il float. Questo chiama `service.price`, il lettore virtuale rivolto agli euro definito sul modello — non `service.price_cents` — quindi il numero sullo schermo è davvero `42.5`, formattato, mai `4250`.

Lasciato completamente senza configurazione, `number_to_currency` usa di default i dollari americani — la valuta di VicinoTe è l'euro, quindi quel default andava sovrascritto una volta, globalmente, invece di passare `unit: "€"` a ogni chiamata:

```yaml
# config/locales/en.yml
en:
  number:
    currency:
      format:
        unit: "€"
        format: "%u%n"
```

`unit:` è il simbolo stesso. `format:` è il template che lo posiziona rispetto al numero — `%u` è l'unità, `%n` è il numero formattato, quindi `"%u%n"` significa "simbolo, poi numero, senza spazio," che è ciò che produce `€42.50` invece di `€ 42.50` o `42.50€`. Entrambe le chiamate a `number_to_currency` nell'app — ce n'è solo una, in `index.html.erb` — recepiscono questo automaticamente senza modifiche al punto di chiamata, perché è il default stesso a essersi spostato, non la chiamata.

## Una trappola solo nelle view: l'errore diceva "price cents"

Inviare il form vuoto la prima volta, prima di sistemare questo, produceva:

```
Category must exist
Title can't be blank
Description can't be blank
Price cents can't be blank
Price cents is not a number
```

Tutto il resto si legge naturalmente — "Title can't be blank" — perché Rails deriva l'etichetta leggibile dal nome dell'attributo tramite `humanize`. Ma l'attributo effettivamente validato è `price_cents`, non `price`, quindi è quel nome a essere trapelato nel messaggio. Un utente che compila questo form non ha mai sentito parlare di `price_cents`; il campo del form appena sotto l'errore dice semplicemente "Price (EUR)".

La correzione è un override I18n di una riga, non una modifica al modello — la validazione è corretta, solo il suo nome renderizzato era sbagliato:

```yaml
# config/locales/en.yml
en:
  activerecord:
    attributes:
      service:
        price_cents: "Price"
```

Questo è il lookup I18n di Rails per `human_attribute_name`: prima di ricadere su un `humanize` automatico del nome dell'attributo, ActiveRecord controlla `activerecord.attributes.<model>.<attributo>` nei file di locale, e usa qualsiasi stringa trovi lì esattamente com'è, per ogni messaggio che menziona quell'attributo — errori di validazione, `form.label` senza testo esplicito, ovunque.

Un dettaglio che è costato un secondo passaggio: scrivere l'override come `price` (minuscolo) produceva "price can't be blank" — minuscolo, incoerente con "Title" e "Description" accanto. Lo `humanize` di default di Rails mette la maiuscola alla prima lettera automaticamente come parte di ciò che fa; una stringa I18n personalizzata viene usata esattamente come scritta, senza nessuna maiuscola applicata sopra. La correzione è stata semplicemente mettere la maiuscola nell'override stesso, `"Price"`.

## Provarlo

```bash
bin/dev
```

Registrati (o accedi), clicca "Offer a service", inserisci un titolo, scegli una categoria, scrivi una descrizione, e un prezzo tipo `42.50`. Invia, e reindirizza a `/services` con "Your service is live." in un banner flash, l'annuncio subito sotto — categoria, titolo, chi l'ha pubblicato, descrizione, e `€42.50`, non `4250`. Disconnettiti e visita `/services` direttamente: è ancora lì, ancora pubblico. Prova `/services/new` da disconnesso e reindirizza all'accesso, come qualsiasi altra pagina protetta. Invia il form con tutto vuoto e compare il messaggio di validazione di ogni campo, in inglese semplice e correttamente maiuscolo, "price_cents" da nessuna parte.

## Cosa viene dopo

L'episodio 4 costruisce `Booking` — il record di un accordo tra due utenti, e il flusso che permette davvero a qualcuno di prenotare un servizio elencato.
