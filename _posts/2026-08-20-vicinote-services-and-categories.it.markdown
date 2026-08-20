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
image: /assets/images/vicinote-banner.png
---

L'[episodio 2]({% post_url 2026-08-13-vicinote-authentication-with-rails-8 %}) aveva fatto funzionare gli account da capo a fondo — registrazione, accesso, disconnessione, reset della password — e si era fermato lì di proposito: nulla toccava ancora `Service` o `Booking`. Questo episodio è dove il marketplace inizia davvero a essere un marketplace: un utente autenticato può elencare qualcosa che offre, e chiunque può sfogliare cosa è elencato.

È anche dove l'associazione `has_many :services` dello schizzo di dominio dell'[episodio 1]({% post_url 2026-08-08-vicinote-project-setup-and-domain %}) viene finalmente scritta nel modello `User`. È rimasta in un post del blog come decisione di design per due episodi; oggi diventa codice vero.

Il codice è taggato [`episode-3`](https://github.com/AntoninoScaffidi/vicinote-tutorial/tree/episode-3) nel repo [vicinote-tutorial](https://github.com/AntoninoScaffidi/vicinote-tutorial).

## Generare i due modelli

```bash
bin/rails generate model Category "name:string:uniq"
bin/rails generate model Service title:string description:text price_cents:integer user:references category:references
```

`user:references` e `category:references` fanno due cose insieme: aggiungono una `belongs_to` al modello generato, e aggiungono la colonna di foreign key più l'indice alla migrazione. Per questo il modello `Service` generato ha già entrambe le associazioni scritte dentro — il generator le ha dedotte dai reference field, non da qualcosa che abbiamo scritto a mano.

## Una decisione su cui vale la pena soffermarsi: price_cents, non price

Il generator ha scritto `price_cents:integer`, non `price:decimal`. È deliberato, ed è la stessa categoria di decisione che l'episodio 1 aveva preso su `Booking` che salva il proprio prezzo invece di leggere `service.price` dal vivo: il denaro gestito come float o come decimal ingenuo prima o poi produce un errore di arrotondamento che si presenta come qualche centesimo di scarto su una fattura, nel momento peggiore possibile. Salvare l'importo come numero intero di centesimi evita del tutto questa categoria di bug — non c'è una parte frazionaria da arrotondare.

Il costo è che nessuno vuole scrivere "4250" in un form intendendo 42,50 $. Quindi `Service` ottiene un piccolo accessor virtuale che parla in dollari in entrata e in uscita, mentre `price_cents` resta ciò che viene davvero validato e salvato:

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
  # un form. Questo parla in dollari in entrata e in uscita, così il campo
  # del form può semplicemente essere "price".
  def price
    price_cents && price_cents / 100.0
  end

  def price=(value)
    self.price_cents = value.present? ? (value.to_f * 100).round : nil
  end
end
```

`price=` è un normale setter Ruby, non un attributo ActiveRecord — Rails lo chiama semplicemente come qualsiasi altro metodo quando un form invia `service[price]`, lo stesso meccanismo che fa funzionare `User.new(email_address: "...")`. Poiché gira prima della validazione, `price_cents` è già popolato nel momento in cui `numericality` lo controlla. Il campo del form, più avanti in questo post, è solo `form.text_field :price` — la view non sa mai che esistono i centesimi.

## Le categorie sono seminate, non create

Un marketplace dove chiunque può inventare una nuova categoria finisce con cinquanta categorie che significano la stessa cosa, scritte in cinque modi diversi, e sfogliare per categoria smette di essere utile. VicinoTe cura invece una lista fissa:

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

```bash
bin/rails db:migrate
bin/rails db:seed
```

`find_or_create_by!` rende sicuro eseguire questo più di una volta — rieseguire `db:seed` dopo aver aggiunto una nona categoria più avanti non duplicherà le prime otto. In questo episodio non si costruisce nessuna interfaccia per gestire le categorie; per ora sono qualcosa con cui l'app viene spedita, non qualcosa che un form `Service` può inventare.

## Scrivere l'associazione che l'episodio 1 aveva solo schizzato

Il design del dominio dell'episodio 1 mostrava questo codice come illustrazione della decisione "il ruolo emerge dall'associazione". Non era mai stato davvero nel modello `User` — l'episodio 2 riguardava l'autenticazione e non l'ha toccato. Ci va ora:

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password
  has_many :sessions, dependent: :destroy
  has_many :services, dependent: :destroy

  normalizes :email_address, with: ->(e) { e.strip.downcase }
end
```

`dependent: :destroy` corrisponde a quello che `sessions` fa già — se l'account di un utente viene mai eliminato, i suoi annunci non dovrebbero restare in giro come righe orfane che puntano a uno `user_id` che non esiste più.

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

Il concern `Authentication` dell'episodio 2 richiede una sessione su ogni azione di default, il che è esattamente giusto per `new` e `create` — nessuno dovrebbe poter elencare un servizio in anonimo — ma sbagliato per `index`: sfogliare il marketplace deve funzionare anche per chi non si è ancora registrato, altrimenti non c'è motivo di registrarsi. `allow_unauthenticated_access only: :index` esclude di nuovo quella singola azione, lo stesso meccanismo che l'episodio 2 aveva usato per rendere pubblica la landing page stessa.

`Current.user.services.new` è il ritorno dell'aver collegato `has_many :services` qualche paragrafo fa: costruisce un nuovo `Service` già associato a chiunque sia autenticato, quindi non c'è nessuno `user_id` in `service_params` che un visitatore possa manomettere — il fornitore è chiunque dica `Current.user`, non quello che dichiara un campo nascosto del form.

`service_params` permette `:price`, non `:price_cents` — il controller parla con la stessa interfaccia rivolta ai dollari del form.

## Le routes e i due bottoni placeholder

```ruby
# config/routes.rb
resources :services, only: [:index, :new, :create]
```

Solo tre azioni — ancora nessuna `show`, dato che non c'è una pagina del singolo servizio a cui linkare finché un episodio successivo non ne costruisce una.

La landing page dell'episodio 1 era stata spedita con due bottoni che non facevano nulla di proposito:

```erb
<span class="... opacity-60 cursor-not-allowed">Browse services</span>
<span class="... opacity-60 cursor-not-allowed">Offer a service</span>
```

Finalmente portano da qualche parte:

```erb
<%= link_to "Browse services", services_path, class: "..." %>
<%= link_to "Offer a service", authenticated? ? new_service_path : new_registration_path, class: "..." %>
```

"Browse services" è incondizionato — l'index è pubblico, quindi il link ha sempre senso. "Offer a service" controlla `authenticated?` (l'helper che il concern dell'episodio 2 espone) e manda un visitatore non autenticato a registrarsi prima, invece che a un form che lo rimbalzerebbe comunque alla pagina di accesso. Stessa destinazione finale, in entrambi i casi — solo un redirect in meno per il caso comune di chi non si è ancora autenticato.

## Una trappola solo nelle view: l'errore diceva "price cents"

Inviare il form vuoto la prima volta, prima di sistemare questo, produceva:

```
Category must exist
Title can't be blank
Description can't be blank
Price cents can't be blank
Price cents is not a number
```

Tutto il resto si legge naturalmente — "Title can't be blank" — perché Rails deriva l'etichetta leggibile dal nome dell'attributo tramite `humanize`. Ma l'attributo effettivamente validato è `price_cents`, non `price`, quindi è quel nome a essere trapelato nel messaggio. Un utente che compila questo form non ha mai sentito parlare di `price_cents`; il campo del form appena sotto l'errore dice semplicemente "Price (USD)".

La correzione è un override I18n di una riga, non una modifica al modello — la validazione è corretta, solo il suo nome renderizzato era sbagliato:

```yaml
# config/locales/en.yml
en:
  activerecord:
    attributes:
      service:
        price_cents: "Price"
```

Un dettaglio che è costato un secondo passaggio: scrivere l'override come `price` (minuscolo) produceva "price can't be blank" — minuscolo, incoerente con "Title" e "Description" accanto. Lo `humanize` di default di Rails mette la maiuscola automaticamente; una stringa I18n personalizzata viene usata esattamente come scritta, senza nessuna maiuscola applicata sopra. La correzione è stata semplicemente mettere la maiuscola nell'override stesso, `"Price"`.

## Provarlo

```bash
bin/dev
```

Registrati (o accedi), clicca "Offer a service", inserisci un titolo, scegli una categoria, scrivi una descrizione, e un prezzo tipo `42.50`. Invia, e reindirizza a `/services` con "Your service is live." in un banner flash, l'annuncio subito sotto — categoria, titolo, chi l'ha pubblicato, descrizione, e `$42.50`, non `4250`. Disconnettiti e visita `/services` direttamente: è ancora lì, ancora pubblico. Prova `/services/new` da disconnesso e reindirizza all'accesso, come qualsiasi altra pagina protetta.

## Cosa viene dopo

L'episodio 4 costruisce `Booking` — il record di un accordo tra due utenti, e il flusso che permette davvero a qualcuno di prenotare un servizio elencato.
