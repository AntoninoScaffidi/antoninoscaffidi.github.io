---
layout: post
title: "VicinoTe: Prenotare un servizio"
series: "vicinote"
episode: 4
lang: it
ref: vicinote-booking-a-service
permalink: /vicinote-booking-a-service/
canonical_url: https://antoninoscaffidi.github.io/it/vicinote-booking-a-service/
date: 2026-08-28 05:00:00 +0200
image: /assets/images/vicinote-banner.png
---

L'[episodio 3]({% post_url 2026-08-20-vicinote-services-and-categories %}) aveva portato un utente autenticato al punto di elencare qualcosa che offre. Nessuno però poteva davvero prenotarlo — non esisteva una pagina per il singolo servizio, né un record dell'accordo tra due persone. Questo episodio chiude entrambe le lacune: `Service` ottiene finalmente una pagina `show`, e un nuovo modello `Booking` è il record di un utente che accetta di pagare un altro per qualcosa, in un giorno specifico, a un prezzo specifico.

Il codice è taggato [`episode-4`](https://github.com/AntoninoScaffidi/vicinote-tutorial/tree/episode-4) nel repo [vicinote-tutorial](https://github.com/AntoninoScaffidi/vicinote-tutorial).

## Cosa tocca questo episodio, in sintesi

```
db/migrate/..._create_bookings.rb     nuovo — la tabella bookings
app/models/booking.rb                 nuovo — Booking: belongs_to :service/:customer, validazioni
app/models/concerns/priceable.rb      nuovo — l'accessor euro<->centesimi, estratto perché Booking lo condivida
app/models/service.rb                 modifica — has_many :bookings, usa Priceable
app/models/user.rb                    modifica — aggiunge has_many :bookings e :received_bookings
app/controllers/services_controller.rb modifica — aggiunge #show
app/controllers/bookings_controller.rb nuovo — #create e #index
app/views/services/show.html.erb      nuovo — dettagli del servizio più il form di prenotazione
app/views/services/index.html.erb     modifica — i titoli dei servizi ora linkano alla loro pagina show
app/views/bookings/index.html.erb     nuovo — "Le mie prenotazioni", divise tra prenotate e ricevute
app/views/layouts/application.html.erb modifica — un link "My bookings" nella nav
config/routes.rb                      modifica — services show, bookings#create annidata, bookings#index
```

## Generare il modello

```bash
bin/rails generate model Booking service:references customer:references price_cents:integer status:string scheduled_on:date
```

`service:references` si comporta esattamente come per `Service` nell'episodio 3 — una colonna di foreign key, un indice, e un `belongs_to :service` scritto direttamente nel modello generato. `customer:references` fa la stessa cosa, ma in modo sbagliato: in questa app non esiste un modello `Customer`. "Customer" è un ruolo che uno `User` gioca in una data prenotazione, non una classe a sé — la stessa decisione "il ruolo emerge dall'associazione" che l'episodio 1 aveva preso per i fornitori di `Service`, applicata qui all'altro lato di una prenotazione.

Il generator non ha modo di saperlo, e indovina in base al solo nome. Quello che ha effettivamente prodotto:

```ruby
# generato, prima della modifica
t.references :customer, null: false, foreign_key: true
```

```ruby
# generato, prima della modifica
belongs_to :customer
```

Lasciata così, la migrazione avrebbe provato ad aggiungere una foreign key verso una tabella `customers` che non esisterà mai, e il modello avrebbe cercato una classe `Customer` altrettanto inesistente — entrambe avrebbero fallito nel momento esatto in cui una delle due righe fosse davvero girata. Entrambe hanno avuto bisogno di una correzione esplicita.

## La migrazione, corretta

```ruby
# db/migrate/..._create_bookings.rb
class CreateBookings < ActiveRecord::Migration[8.1]
  def change
    create_table :bookings do |t|
      t.references :service, null: false, foreign_key: true
      t.references :customer, null: false, foreign_key: { to_table: :users }
      t.integer :price_cents, null: false
      t.string :status, null: false, default: "pending"
      t.date :scheduled_on, null: false

      t.timestamps
    end
  end
end
```

- `t.references :customer, null: false, foreign_key: { to_table: :users }` — la correzione dell'ipotesi sbagliata. `foreign_key: true` da solo deduce la tabella di destinazione dal nome stesso della reference (`customer` → `customers`); passare un hash con `to_table:` sovrascrive quella deduzione e punta il vincolo alla tabella reale, `users`, mentre la colonna stessa resta chiamata `customer_id` — il nome che conta davvero per il codice che la legge.
- `t.integer :price_cents, null: false` — nessun default qui, deliberatamente. Una prenotazione ha sempre un prezzo concordato; non c'è un valore ragionevole su cui ricadere se non viene fornito.
- `t.string :status, null: false, default: "pending"` — ogni prenotazione parte dallo stesso stato. Questo episodio non lo cambia mai — nulla qui costruisce un'azione "conferma" o "annulla" — ma la colonna, e l'insieme di valori che può contenere (imposto nel modello sotto), sono già al loro posto per quando servirà.
- `t.date :scheduled_on, null: false` — l'unica informazione che il cliente fornisce davvero: per quale giorno è.

## Condividere l'accessor del prezzo: il concern Priceable

L'episodio 3 aveva dato a `Service` una piccola coppia `price`/`price=` per parlare in euro mentre `price_cents` resta ciò che viene salvato. `Booking` ha bisogno esattamente dello stesso comportamento — ha anche lui la propria colonna `price_cents` adesso. Copiare quelle quattro righe una seconda volta funzionerebbe, ma due copie indipendenti della stessa logica tendono a divergere nel momento in cui una delle due ha bisogno di una correzione e l'altra viene dimenticata. È esattamente per questo che esiste `ActiveSupport::Concern`:

```ruby
# app/models/concerns/priceable.rb
module Priceable
  extend ActiveSupport::Concern

  def price
    price_cents && price_cents / 100.0
  end

  def price=(value)
    self.price_cents = value.present? ? (value.to_f * 100).round : nil
  end
end
```

Nulla in questo codice è cambiato rispetto all'episodio 3 — sono gli stessi due metodi, spostati parola per parola dentro `Service` in un file proprio sotto `app/models/concerns/`, una cartella che Rails autocarica per convenzione senza bisogno di alcun `require`. `extend ActiveSupport::Concern` è ciò che fa comportare `include Priceable` in modo prevedibile anche se il concern dovesse in futuro crescere con un blocco `included do ... end` o con propri metodi di classe — per un modulo semplice come questo non è strettamente necessario, ma è la forma standard che assume un concern Rails, e usarla sempre significa non dover mai ricordare quali moduli hanno bisogno del trattamento speciale e quali no.

```ruby
# app/models/service.rb
class Service < ApplicationRecord
  include Priceable

  belongs_to :user
  belongs_to :category
  has_many :bookings

  validates :title, presence: true
  validates :description, presence: true
  validates :price_cents, presence: true, numericality: { greater_than: 0, only_integer: true }
end
```

`Service` ha perso i suoi due metodi per il prezzo e ha guadagnato `include Priceable` e `has_many :bookings` al loro posto — l'altra metà di `Booking belongs_to :service`, necessaria nel momento in cui qualcosa vuole chiedere a un servizio quali prenotazioni sono state fatte contro di esso.

## Il modello Booking

```ruby
# app/models/booking.rb
class Booking < ApplicationRecord
  include Priceable

  belongs_to :service
  belongs_to :customer, class_name: "User"

  validates :scheduled_on, presence: true
  validates :status, presence: true, inclusion: { in: %w[pending confirmed completed cancelled] }
  validates :price_cents, presence: true, numericality: { greater_than: 0, only_integer: true }

  validate :customer_is_not_the_provider

  private

  def customer_is_not_the_provider
    return unless service && customer_id

    errors.add(:customer, "can't book their own service") if service.user_id == customer_id
  end
end
```

- `belongs_to :customer, class_name: "User"` — la metà lato modello della stessa correzione richiesta dalla migrazione. `class_name: "User"` dice a Rails "l'associazione si chiama `customer`, ma la classe reale dall'altra parte è `User`" — senza, Rails proverebbe a fare il constantize di `"Customer"` e fallirebbe. La foreign key (`customer_id`) viene comunque dedotta correttamente dal nome dell'associazione; solo la classe aveva bisogno di essere specificata.
- `validates :status, ..., inclusion: { in: %w[pending confirmed completed cancelled] }` — la colonna del database è una semplice stringa senza alcun vincolo proprio oltre a `null: false`; è questa validazione a impedire davvero che un `Booking` possa mai contenere un refuso o uno stato inventato che Rails non riconosce. Solo `"pending"` è raggiungibile con qualsiasi cosa costruita in questo episodio, ma l'insieme completo viene dichiarato fin da ora invece di crescere una stringa alla volta più avanti.
- `validate :customer_is_not_the_provider` — una validazione che non controlla una colonna contro una regola fissa, ma un'associazione contro un'altra. `return unless service && customer_id` protegge dall'eseguire il confronto quando uno dei due lati manca ancora (un `Booking` nuovo di zecca e vuoto non dovrebbe produrre proprio questo errore solo perché non è stato ancora compilato nulla — le validazioni di presenza sopra coprono già quel caso). Il controllo vero e proprio, `service.user_id == customer_id`, è ciò che trasforma "non puoi prenotare il tuo stesso servizio" da una frase nel documento di design dell'episodio 1 in qualcosa che il database non potrà mai contenere davvero.

Da notare cosa **non** c'è qui: nessun `before_validation` che copia `service.price_cents` nella prenotazione automaticamente. È deliberato, e il ragionamento è nel controller subito dopo — lo scatto del prezzo avviene dove è visibile, non nascosto dentro una callback.

## Collegare le associazioni che l'episodio 1 aveva solo schizzato

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password
  has_many :sessions, dependent: :destroy
  has_many :services, dependent: :destroy
  has_many :bookings, foreign_key: :customer_id, dependent: :destroy, inverse_of: :customer
  has_many :received_bookings, through: :services, source: :bookings

  normalizes :email_address, with: ->(e) { e.strip.downcase }
end
```

Il design di dominio dell'episodio 1 mostrava entrambe queste righe come illustrazione di "un solo `User`, due ruoli che emergono dalle associazioni, non una colonna di ruolo." L'episodio 3 aveva già collegato `has_many :services`; queste due sono quelle rimaste solo uno schizzo fino ad ora.

- `has_many :bookings, foreign_key: :customer_id, dependent: :destroy, inverse_of: :customer` — senza `foreign_key:`, Rails cercherebbe una colonna `user_id` su `bookings`, che non esiste; la colonna reale è `customer_id`, quindi va nominata esplicitamente. `inverse_of: :customer` dice a Rails che questa associazione e `Booking belongs_to :customer` sono due facce della stessa relazione, il che permette a Rails di saltare una query ridondante quando ha già un lato caricato in memoria e ha bisogno dell'altro. `dependent: :destroy` corrisponde alla stessa policy già applicata a `sessions` e `services`: eliminare uno `User` non dovrebbe lasciare righe `Booking` che puntano a un `customer_id` che non esiste più.
- `has_many :received_bookings, through: :services, source: :bookings` — un `has_many :through`, non una foreign key diretta su `users`. Si legge da destra a sinistra: per ognuno dei `services` di questo utente, segui l'associazione `bookings` propria di quel servizio, e tratta il risultato combinato come i `received_bookings` di questo utente. `source: :bookings` serve solo perché il nome dell'associazione da questo lato (`received_bookings`) non corrisponde al nome dell'associazione che viene seguita su `Service` (`bookings`) — senza, Rails cercherebbe un metodo `received_bookings` su `Service`, che non esiste, invece di quello reale, `bookings`.

La distinzione tra queste due conta: `user.bookings` è "cosa ho prenotato, come cliente" — una relazione diretta. `user.received_bookings` è "cosa è stato prenotato da me, su tutti i servizi che offro" — una relazione che esiste solo passando per una seconda tabella intermedia. Stessa tabella `bookings` sottostante, due domande genuinamente diverse.

## Finalmente: una pagina per un singolo servizio

```ruby
# app/controllers/services_controller.rb
class ServicesController < ApplicationController
  allow_unauthenticated_access only: %i[index show]

  def index
    @services = Service.includes(:category, :user).order(created_at: :desc)
  end

  def show
    @service = Service.includes(:category, :user).find(params[:id])
    @booking = Booking.new
  end

  # new/create invariati dall'episodio 3
end
```

`allow_unauthenticated_access only: %i[index show]` — l'episodio 3 esentava solo `index` dall'obbligo di accesso; `show` lo raggiunge qui, per lo stesso motivo: sfogliare un singolo annuncio deve funzionare anche per un visitatore che non si è mai registrato, altrimenti il marketplace è visibile solo a chi si è già impegnato a usarlo. `@booking = Booking.new` costruisce una `Booking` vuota e non salvata puramente perché il `form_with model: @booking` della view ha bisogno di qualcosa attorno a cui costruire un form — nulla qui la salva o compila anche solo uno dei suoi attributi ancora.

```ruby
# config/routes.rb
resources :services, only: [:index, :new, :create, :show] do
  resources :bookings, only: [:create]
end
resources :bookings, only: [:index]
```

```
          Prefix Verb URI Pattern                              Controller#Action
service_bookings POST /services/:service_id/bookings(.:format) bookings#create
        bookings GET  /bookings(.:format)                      bookings#index
```

Annidare `resources :bookings, only: [:create]` dentro `resources :services` è ciò che produce `POST /services/:service_id/bookings` e l'helper `service_bookings_path(service)` usato nella view più sotto — è l'URL stesso a dire per quale servizio è una nuova prenotazione, così il controller non deve mai indovinarlo da nessun'altra parte. Il secondo `resources :bookings, only: [:index]`, non annidato, è separato di proposito: "elenco delle mie prenotazioni" non è ristretto a un singolo servizio, quindi non appartiene affatto sotto `/services/:service_id/`.

## Prenotare, e dove il prezzo viene davvero copiato

```ruby
# app/controllers/bookings_controller.rb
class BookingsController < ApplicationController
  def index
    @my_bookings = Current.user.bookings.includes(:service).order(scheduled_on: :asc)
    @received_bookings = Current.user.received_bookings.includes(:service, :customer).order(scheduled_on: :asc)
  end

  def create
    service = Service.find(params[:service_id])
    booking = Current.user.bookings.new(booking_params.merge(service: service, price_cents: service.price_cents))

    if booking.save
      redirect_to bookings_path, notice: "Booked for #{booking.scheduled_on}."
    else
      redirect_to service_path(service), alert: booking.errors.full_messages.to_sentence
    end
  end

  private

  def booking_params
    params.require(:booking).permit(:scheduled_on)
  end
end
```

Nessun `allow_unauthenticated_access` da nessuna parte in questo controller — entrambe le azioni restano dietro l'obbligo di accesso di default, correttamente: nessuno dovrebbe poter elencare o fare una prenotazione in anonimo.

- `def index` — due query separate assegnate a due variabili di istanza separate, una per ciascun lato in cui un utente autenticato potrebbe trovarsi in una prenotazione. `Current.user.bookings` usa l'associazione diretta; `Current.user.received_bookings` usa quella `has_many :through`. Entrambe ricevono `includes(...)` per lo stesso motivo dell'index dei servizi nell'episodio 3: renderizzare un elenco di prenotazioni e leggere `booking.service.title` (o `booking.customer.email_address`) per ognuna farebbe altrimenti scattare una query in più per ogni riga.
- `service = Service.find(params[:service_id])` — `:service_id`, non `:id`, per via della route annidata: una richiesta a `POST /services/7/bookings` porta l'id del servizio sotto quella chiave, non quello della prenotazione (che ancora non esiste).
- `Current.user.bookings.new(booking_params.merge(service: service, price_cents: service.price_cents))` — questa singola riga è l'intero meccanismo dello scatto del prezzo del design dell'episodio 1, reso concreto. `booking_params` contiene solo `scheduled_on` — l'unico campo che il form invia davvero — quindi `price_cents: service.price_cents` deve essere aggiunto esplicitamente, qui, dal controller, che legge il prezzo **attuale** del servizio nell'esatto momento in cui qualcuno si impegna a prenotarlo. Nulla collega la `Booking` risultante a `Service#price` dopo che questa riga è girata; se il fornitore alza il prezzo domani, ogni prenotazione fatta oggi mantiene il numero che era vero quando è stata fatta. `Current.user.bookings.new(...)`, costruendo attraverso l'associazione invece di `Booking.new(customer: Current.user, ...)`, è anche ciò che fissa il lato cliente a chiunque sia effettivamente autenticato — non c'è nessun `customer_id` da nessuna parte in `booking_params` che un visitatore potrebbe manomettere.
- `if booking.save` / i due rami — in caso di successo, redirect all'elenco delle prenotazioni con un messaggio flash, la stessa forma Post/Redirect/Get usata da ogni azione di scrittura in questa serie fin dall'episodio 2. In caso di fallimento — qualcuno ha provato a prenotare il proprio stesso servizio, oppure la data è stata lasciata vuota — non c'è una view `bookings/new` dedicata da ri-renderizzare con gli errori inline, perché il form vive sulla pagina di qualcun altro (lo `show` del servizio), non sulla propria. Reindirizzare a `service_path(service)` con `alert: booking.errors.full_messages.to_sentence` è l'adattamento più semplice a quella forma: gli errori compaiono come un unico messaggio flash invece che campo per campo, un vero compromesso (feedback meno preciso) fatto deliberatamente invece di forzare un `render` tra controller diversi per evitarlo.

## La pagina del servizio e il suo form di prenotazione

```erb
<%# app/views/services/show.html.erb %>
<div class="max-w-2xl mx-auto w-full">
  <% if alert = flash[:alert] %>
    <p class="py-2 px-3 bg-red-50 mb-5 text-red-700 font-medium rounded-lg inline-block" id="alert"><%= alert %></p>
  <% end %>

  <p class="text-xs font-semibold uppercase tracking-wide text-indigo-600"><%= @service.category.name %></p>
  <div class="mt-1 flex items-start justify-between gap-4">
    <h1 class="text-2xl font-bold text-gray-900"><%= @service.title %></h1>
    <p class="whitespace-nowrap text-2xl font-semibold text-gray-900"><%= number_to_currency(@service.price) %></p>
  </div>
  <p class="mt-1 text-sm text-gray-500">by <%= @service.user.email_address %></p>
  <p class="mt-4 text-gray-600"><%= @service.description %></p>

  <hr class="my-8 border-gray-200">

  <% if !authenticated? %>
    <p class="text-gray-600">
      <%= link_to "Sign in", new_session_path, class: "text-indigo-600 hover:underline" %> to book this service.
    </p>
  <% elsif Current.user == @service.user %>
    <p class="text-gray-500">This is your own service.</p>
  <% else %>
    <h2 class="text-lg font-semibold text-gray-900 mb-3">Book this service</h2>
    <%= form_with model: @booking, url: service_bookings_path(@service), class: "flex items-end gap-3" do |form| %>
      <div>
        <%= form.label :scheduled_on, "Date", class: "block text-sm font-medium text-gray-700" %>
        <%= form.date_field :scheduled_on, class: "mt-1 block rounded-md border border-gray-300 px-3 py-2 focus:outline-none focus:ring-2 focus:ring-indigo-500" %>
      </div>
      <%= form.submit "Book for #{number_to_currency(@service.price)}", class: "rounded-md bg-indigo-600 px-4 py-2 text-white font-medium hover:bg-indigo-700 cursor-pointer" %>
    <% end %>
  <% end %>
</div>
```

- `<% if !authenticated? %> ... <% elsif Current.user == @service.user %> ... <% else %> ... <% end %>` — un ramo a tre vie su chi sta guardando questa pagina: un visitatore anonimo (invitato ad accedere, invece di vedersi mostrare un form che fallirebbe comunque lato server), il fornitore stesso (a cui viene detto chiaramente che questo è il suo annuncio, dato che prenotare il proprio servizio non ha senso e il controller lo rifiuterebbe comunque), oppure chiunque altro (il vero form di prenotazione). La validazione `customer_is_not_the_provider` del modello è l'imposizione reale; nascondere il form qui è la metà rivolta all'utente della stessa regola, non un sostituto — il classico abbinamento comodità-lato-client più verità-lato-server.
- `form_with model: @booking, url: service_bookings_path(@service), ...` — `service_bookings_path(@service)` è l'helper della route annidata, che produce `/services/7/bookings` per il servizio `7`; passare `@service` (un oggetto ActiveRecord) invece del suo `id` lascia che Rails chiami `.to_param` su di esso, che per una chiave primaria `id` semplice e non modificata restituisce semplicemente l'id stesso.
- `form.date_field :scheduled_on` — un input HTML5 nativo di tipo data, che ottiene gratuitamente un selettore in stile calendario in ogni browser moderno, senza una riga di JavaScript scritta per questo.
- `form.submit "Book for #{number_to_currency(@service.price)}", ...` — l'etichetta stessa del bottone dichiara il prezzo esatto a cui ci si sta impegnando, calcolato allo stesso modo della visualizzazione del prezzo più in alto nella pagina, così nessuno invia senza aver appena visto il numero che sta per accettare.

## L'elenco delle prenotazioni

```erb
<%# app/views/bookings/index.html.erb %>
<div class="max-w-2xl mx-auto w-full">
  <% if notice = flash[:notice] %>
    <p class="py-2 px-3 bg-green-50 mb-5 text-green-700 font-medium rounded-lg inline-block" id="notice"><%= notice %></p>
  <% end %>

  <h1 class="text-2xl font-bold text-gray-900 mb-6">My bookings</h1>

  <h2 class="text-sm font-semibold uppercase tracking-wide text-gray-500 mb-3">Services you've booked</h2>
  <% if @my_bookings.none? %>
    <p class="text-gray-500 mb-8">You haven't booked anything yet.</p>
  <% else %>
    <div class="space-y-3 mb-8">
      <% @my_bookings.each do |booking| %>
        <div class="rounded-lg border border-gray-200 p-4 flex items-center justify-between">
          <div>
            <p class="font-medium text-gray-900"><%= booking.service.title %></p>
            <p class="text-sm text-gray-500"><%= booking.scheduled_on.strftime("%-d %B %Y") %> &middot; <%= booking.status %></p>
          </div>
          <p class="font-semibold text-gray-900"><%= number_to_currency(booking.price) %></p>
        </div>
      <% end %>
    </div>
  <% end %>

  <h2 class="text-sm font-semibold uppercase tracking-wide text-gray-500 mb-3">Bookings received</h2>
  <% if @received_bookings.none? %>
    <p class="text-gray-500">Nobody has booked one of your services yet.</p>
  <% else %>
    <div class="space-y-3">
      <% @received_bookings.each do |booking| %>
        <div class="rounded-lg border border-gray-200 p-4 flex items-center justify-between">
          <div>
            <p class="font-medium text-gray-900"><%= booking.service.title %></p>
            <p class="text-sm text-gray-500">
              <%= booking.scheduled_on.strftime("%-d %B %Y") %> &middot; <%= booking.status %> &middot; booked by <%= booking.customer.email_address %>
            </p>
          </div>
          <p class="font-semibold text-gray-900"><%= number_to_currency(booking.price) %></p>
        </div>
      <% end %>
    </div>
  <% end %>
</div>
```

Due elenchi indipendenti su un'unica pagina, ciascuno con il proprio stato vuoto, che rispecchiano esattamente le due variabili di istanza separate del controller — `@my_bookings` si renderizza come "servizi che hai prenotato", `@received_bookings` come "prenotazioni ricevute". `booking.price`, non `booking.service.price` — ogni numero su questa pagina è il prezzo congelato al momento della prenotazione, deliberatamente mai quello attuale del servizio. `booking.customer.email_address` compare solo nell'elenco ricevute — sai sempre chi ha prenotato da te; non hai bisogno di ricordare chi sei *tu* nel tuo stesso elenco delle prenotazioni fatte.

## Provarlo, senza browser

Il solito giro con `bin/dev` funziona anche qui — accedi come un utente, visita un servizio elencato da qualcun altro, scegli una data, prenota, poi controlla `/bookings` sia come cliente sia, accedendo come fornitore, dal lato ricevente. La verifica reale di questo episodio è avvenuta in un modo diverso, direttamente con `bin/rails runner`, apposta per dimostrare il comportamento che conta di più ed è il più difficile da vedere a occhio da uno screenshot — che il prezzo di una prenotazione non si muove davvero mai dopo che è stata fatta:

```ruby
maria = User.find_by(email_address: "maria@example.com")
service = maria.services.first
customer = User.find_or_create_by!(email_address: "luca@example.com") { |u| u.password = "supersecret123" }

booking = customer.bookings.new(service: service, scheduled_on: Date.tomorrow, price_cents: service.price_cents)
booking.save!
puts "Booked at: #{booking.price}"

service.update!(price: 99.99)
puts "Service price now: #{service.reload.price}, booking price still: #{booking.reload.price}"
```

```
Booked at: 42.5
Service price now: 99.99, booking price still: 42.5
```

Insieme a questo, provare `maria.bookings.new(service: service, ...)` — Maria che prova a prenotare le proprie stesse lezioni di chitarra — ha restituito `valid? false`, con esattamente il messaggio che il modello era stato scritto per produrre: `"Customer can't book their own service"`.

## Cosa viene dopo

L'episodio 5 copre `Review` — lasciata dopo che una prenotazione è completata, appartenente alla prenotazione stessa piuttosto che al servizio, esattamente come aveva deciso il design dell'episodio 1: solo chi ha davvero portato a termine una prenotazione può lasciarne una.
