---
layout: post
title: "VicinoTe: autenticazione con il generator integrato di Rails 8"
series: "vicinote"
episode: 2
lang: it
ref: vicinote-authentication-with-rails-8
permalink: /vicinote-authentication-with-rails-8/
canonical_url: https://antoninoscaffidi.github.io/it/vicinote-authentication-with-rails-8/
image: /assets/images/vicinote-banner.png
date: 2026-08-13 09:00:00 +0200
---

L'[episodio 1]({% post_url 2026-08-08-vicinote-project-setup-and-domain %}) si era chiuso con una decisione di progettazione e niente su cui accedere: un solo modello `User`, nessuna colonna ruolo, fornitore e cliente che emergono entrambi da associazioni per cui non avevamo ancora scritto codice. Questo episodio scrive quel modello — e, insieme, tutto il sistema di autenticazione attorno a esso.

Il codice è taggato [`episode-2`](https://github.com/AntoninoScaffidi/vicinote-tutorial/tree/episode-2) nel repo [vicinote-tutorial](https://github.com/AntoninoScaffidi/vicinote-tutorial).

## Non Devise

Ogni versione passata di "aggiungi l'autenticazione a un'app Rails" iniziava con `gem "devise"`. Rails 8 ha cambiato le cose: adesso c'è un generator integrato, `bin/rails generate authentication`, che scrive codice Rails semplice e ordinario — modelli, controller, un concern — direttamente nella tua app. Nessuna gemma, nessun engine, nessun codice generato che vive da qualche parte in una gemma che non puoi leggere facilmente.

Questa distinzione conta più di quanto sembri. Con Devise, capire "come funziona davvero il login" significa leggere il codice sorgente della gemma. Con il generator di Rails 8, il codice che scrive **è** il tuo codice, sta dentro `app/` come tutto il resto, pronto per essere letto, modificato e — come vedremo a metà di questo episodio — esteso, perché il generator deliberatamente non fa tutto.

```bash
bin/rails generate authentication
```

Ecco l'elenco completo di cosa ha prodotto quel singolo comando:

```
create  app/views/passwords/new.html.erb
create  app/views/passwords/edit.html.erb
create  app/views/sessions/new.html.erb
create  app/models/session.rb
create  app/models/user.rb
create  app/models/current.rb
create  app/controllers/sessions_controller.rb
create  app/controllers/concerns/authentication.rb
create  app/controllers/passwords_controller.rb
create  app/mailers/passwords_mailer.rb
create  app/views/passwords_mailer/reset.html.erb
create  app/views/passwords_mailer/reset.text.erb
insert  app/controllers/application_controller.rb
 route  resources :passwords, param: :token
 route  resource :session
  gsub  Gemfile
  create  db/migrate/..._create_users.rb
  create  db/migrate/..._create_sessions.rb
```

Vale la pena leggerli tutti prima di scrivere qualcosa di nostro, perché — a differenza di una gemma — guarderemo direttamente questo codice per il resto della serie.

## Il modello User

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password
  has_many :sessions, dependent: :destroy

  normalizes :email_address, with: ->(e) { e.strip.downcase }
end
```

`has_secure_password` è la riga che fa il lavoro pesante, e non è una scatola nera in stile Devise — è Rails puro, da `ActiveModel::SecurePassword`. Si aspetta una colonna `password_digest`, aggiunge attributi virtuali `password=`/`password_confirmation=` che vengono hashati con bcrypt all'assegnazione, e aggiunge un metodo d'istanza `authenticate`. Ci torneremo più avanti, perché fa più di questo — il meccanismo di reset password più avanti in questo post viene esattamente dalla stessa riga.

(Vale la pena essere precisi su una cosa: `has_secure_password` in sé è codice di Rails, parte della gemma `activemodel` — non fa parte di `bcrypt`. La gemma `bcrypt` nel Gemfile fornisce solo la classe `BCrypt::Password` che Rails usa internamente per hashare e confrontare davvero le password; la macro, le validazioni e la logica del token di reset vivono tutte dentro Rails stesso, leggibili come qualsiasi altro pezzo del framework.)

`normalizes :email_address` è una funzionalità di Rails più piccola ma genuinamente utile: garantisce che `"Mario@Example.com "` e `"mario@example.com"` vengano trattati come lo stesso indirizzo ovunque — al salvataggio, e in ogni ricerca successiva — senza doverci ricordare di chiamare `.downcase.strip` a mano in ogni punto del codice.

```ruby
# db/migrate/..._create_users.rb
create_table :users do |t|
  t.string :email_address, null: false
  t.string :password_digest, null: false
  t.timestamps
end
add_index :users, :email_address, unique: true
```

Nota cosa **non** c'è qui: nessun `name`, nessun ruolo, niente di specifico del marketplace. È volutamente il record autenticabile minimo indispensabile. Tutto quello della progettazione del dominio dell'episodio 1 — `has_many :services`, `has_many :bookings` — viene aggiunto quando costruiremo davvero `Service` e `Booking`, non adesso. Aggiungere quelle associazioni oggi funzionerebbe tecnicamente (Rails risolve i nomi delle classi delle associazioni in modo pigro), ma sarebbe codice che si riferisce a modelli che non esistono ancora, il che è peggio per chi legge questo repo dall'inizio alla fine.

## Sessioni e Current: come si traccia "chi è loggato"

Questa è la parte che sembra più diversa da quello che ti aspetteresti venendo da Devise, e vale la pena rallentarci sopra.

```ruby
# app/models/session.rb
class Session < ApplicationRecord
  belongs_to :user
end
```

```ruby
# db/migrate/..._create_sessions.rb
create_table :sessions do |t|
  t.references :user, null: false, foreign_key: true
  t.string :ip_address
  t.string :user_agent
  t.timestamps
end
```

Una `Session` è una **riga di database**, non solo un cookie cifrato. Ogni volta che qualcuno accede, viene creato un nuovo record `Session`, che salva quale utente, da quale IP, con quale browser. Il browser tiene solo l'**id** della sessione, firmato dentro un cookie così non può essere manomesso — lo stato vero della sessione vive lato server.

Il vantaggio pratico: puoi vedere ogni sessione attiva di un utente (`user.sessions`), e revocarne una singolarmente — disconnettere un dispositivo specifico — semplicemente cancellando quella riga. Una sessione basata solo su cookie non può farlo; puoi solo invalidarle *tutte* insieme (es. ruotando un secret).

```ruby
# app/models/current.rb
class Current < ActiveSupport::CurrentAttributes
  attribute :session
  delegate :user, to: :session, allow_nil: true
end
```

`ActiveSupport::CurrentAttributes` è un meccanismo di Rails per uno stato globale per-richiesta — più sicuro di una semplice variabile globale perché viene azzerato automaticamente tra una richiesta e l'altra (e tra un test e l'altro), quindi non c'è rischio che l'utente di una richiesta finisca dentro la successiva. `Current.session` contiene il record `Session` corrente per questa richiesta; `Current.user` è semplicemente `Current.session.user`, disponibile ovunque nell'app tramite `Current.user`, senza doverlo passare giù attraverso ogni chiamata di metodo.

## Il concern Authentication: sicuro per default

```ruby
# app/controllers/concerns/authentication.rb
module Authentication
  extend ActiveSupport::Concern

  included do
    before_action :require_authentication
    helper_method :authenticated?
  end

  class_methods do
    def allow_unauthenticated_access(**options)
      skip_before_action :require_authentication, **options
    end
  end

  private
    def resume_session
      Current.session ||= find_session_by_cookie
    end

    def find_session_by_cookie
      Session.find_by(id: cookies.signed[:session_id]) if cookies.signed[:session_id]
    end

    def start_new_session_for(user)
      user.sessions.create!(user_agent: request.user_agent, ip_address: request.remote_ip).tap do |session|
        Current.session = session
        cookies.signed.permanent[:session_id] = { value: session.id, httponly: true, same_site: :lax }
      end
    end
    # ...
end
```

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  include Authentication
  # ...
end
```

La riga che conta di più qui è `before_action :require_authentication` — inclusa direttamente in `ApplicationController`, il che significa che **ogni controller dell'app richiede un utente loggato di default**, a meno che non lo escluda esplicitamente con `allow_unauthenticated_access`. È l'opposto di come la maggior parte dei tutorial costruisce l'autenticazione (dove proteggi azioni specifiche), ed è un default deliberato e sensato per un'app il cui scopo intero sono gli account: è molto più sicuro doversi ricordare di rendere pubblica una pagina che doversi ricordare di proteggerla.

Vale la pena leggere con attenzione anche il cookie firmato in sé: `cookies.signed.permanent[...]` — firmato, quindi il valore non può essere falsificato (Rails lo verifica contro un secret prima di fidarsene); `permanent`, quindi impostato per scadere tra 20 anni invece che alla fine della sessione del browser; `httponly: true`, così il JavaScript lato client non può mai leggerlo (chiudendo un'intera categoria di furto di sessione via XSS); `same_site: :lax`, una protezione CSRF di base che impedisce l'invio del cookie su richieste cross-site tranne la navigazione di primo livello.

### Una trappola: la nostra landing page pubblica si è appena rotta

Poiché `require_authentication` è ora il default ovunque, la landing page dell'episodio 1 — pensata per essere la prima cosa che chiunque vede, loggato o no — reindirizzerebbe alla pagina di login nel momento in cui eseguiamo il generator. `PagesController` aveva bisogno di un'esclusione esplicita:

```ruby
# app/controllers/pages_controller.rb
class PagesController < ApplicationController
  allow_unauthenticated_access

  def home
  end
end
```

È esattamente il compromesso che "sicuro per default" fa apposta: ci sbatterai contro la prima volta che aggiungi il generator a un'app con pagine pubbliche già esistenti, e la correzione è una riga sola, ma devi sapere che va cercata.

## Accedere: `authenticate_by` e rate limiting

```ruby
# app/controllers/sessions_controller.rb
class SessionsController < ApplicationController
  allow_unauthenticated_access only: %i[ new create ]
  rate_limit to: 10, within: 3.minutes, only: :create, with: -> { redirect_to new_session_path, alert: "Try again later." }

  def create
    if user = User.authenticate_by(params.permit(:email_address, :password))
      start_new_session_for user
      redirect_to after_authentication_url
    else
      redirect_to new_session_path, alert: "Try another email address or password."
    end
  end
  # ...
end
```

Due dettagli qui vale davvero la pena capirli, non solo copiarli.

**`User.authenticate_by`** sembra solo un `find_by(email_address:) + authenticate(password)` in un'unica chiamata, ma è pensato specificamente per chiudere un attacco a tempo (timing attack). Dal commento nel codice sorgente di Rails:

> Regardless of whether a record is found, `authenticate_by` will cryptographically digest the given password attributes. This behavior helps mitigate timing-based enumeration attacks, wherein an attacker can determine if a passworded record exists even without knowing the password.

In concreto: se prima cercassi l'utente e facessi il confronto bcrypt solo quando ne trovi uno, una richiesta per un'email **inesistente** tornerebbe quasi istantaneamente (nessun lavoro bcrypt), mentre una richiesta per un'email **reale** con password sbagliata impiegherebbe i ~100ms che bcrypt richiede. Quella differenza di tempo basta a un attaccante per enumerare quali email hanno un account, semplicemente misurando i tempi di risposta. `authenticate_by` fa sempre il lavoro costoso di digest, trovato o no, così entrambi i casi impiegano lo stesso tempo.

**`rate_limit`** è una funzionalità nativa di Rails 8 (`ActionController::RateLimiting`, nessuna gemma) — dieci tentativi ogni tre minuti su questa azione, sostenuta da `Rails.cache`, con un redirect e un messaggio invece di fallire silenziosamente. Un attacco a forza bruta con migliaia di combinazioni viene rallentato in modo significativo da questa sola riga.

## Il pezzo mancante: la registrazione

Prova le rotte del generator e troverai `new_session_path` e `new_password_path` (reset), ma nulla per creare un `User` in primo luogo. Non è una dimenticanza — il generator non può indovinare i requisiti di registrazione della tua app (solo su invito? conferma email? OAuth?) quindi lascia la cosa interamente a te.

Per VicinoTe, chiunque deve potersi registrare — è un marketplace, non uno strumento amministrativo — quindi lo aggiungiamo noi, seguendo la stessa forma del codice generato:

```ruby
# app/controllers/registrations_controller.rb
class RegistrationsController < ApplicationController
  allow_unauthenticated_access

  def new
    @user = User.new
  end

  def create
    @user = User.new(user_params)

    if @user.save
      start_new_session_for @user
      redirect_to root_path, notice: "Welcome to VicinoTe!"
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  def user_params
    params.require(:user).permit(:email_address, :password, :password_confirmation)
  end
end
```

```ruby
# config/routes.rb
resource :registration, only: [:new, :create]
```

Nulla qui è meccanismo nuovo — `start_new_session_for` è lo stesso metodo privato che usa `SessionsController`, chiamabile perché definito nel concern `Authentication` incluso in ogni controller. La registrazione, in questo design, è solo "crea uno `User`, poi fai esattamente quello che fa il login".

## Reset password senza una colonna per il token

Questo è il dettaglio che ho trovato più interessante da approfondire. `PasswordsController` chiama `User.find_by_password_reset_token!(token)` — un metodo che non esiste da nessuna parte in `user.rb`. Non è scritto a mano, e non c'è nemmeno una colonna `reset_password_token` nella tabella `users`. Da dove viene, allora?

Viene da `has_secure_password` stesso. Leggendo il codice sorgente di Rails (`ActiveModel::SecurePassword`):

```ruby
def has_secure_password(attribute = :password, validations: true, reset_token: true)
  # ...
  if reset_token && respond_to?(:generates_token_for)
    generates_token_for :"#{attribute}_reset", expires_in: 15.minutes do
      public_send(:"#{attribute}_salt")&.last(10)
    end
    # definisce find_by_password_reset_token / find_by_password_reset_token!
  end
end
```

`generates_token_for` è il meccanismo generico di Rails per generare un token verificabile e con scadenza **senza salvarlo da nessuna parte** — è un payload firmato e codificato (tramite `ActiveSupport::MessageVerifier`), controllato e decodificato al ritorno. Qui, `has_secure_password` lo usa automaticamente, basandosi sugli ultimi 10 caratteri del salt bcrypt della password.

Quel dettaglio è la parte intelligente: il salt cambia ogni volta che la password cambia (bcrypt ne genera uno nuovo a ogni hash), il che significa che **un link di reset viene automaticamente invalidato nell'istante in cui la password per cui è stato generato cambia davvero** — senza nessun flag "usato" in più, nessun job di pulizia dei token, nulla da salvare. Il token scade comunque dopo 15 minuti, e `find_by_password_reset_token!` solleva `ActiveSupport::MessageVerifier::InvalidSignature` su un token scaduto o manomesso, che `PasswordsController` intercetta trasformandolo in un redirect con un messaggio comprensibile.

Non costruiamo il flusso completo oggi (serve una configurazione mailer vera, che avrà più senso quando VicinoTe sarà distribuito da qualche parte), ma vale la pena sapere che esiste, completamente collegato, nel momento stesso in cui `has_secure_password` è sul modello — una riga, e arriva insieme un meccanismo di reset password funzionante e sicuro.

## Una piccola barra di navigazione, per raggiungere tutto questo

Niente di quanto sopra ha un'interfaccia per arrivarci finché qualcosa non ci punta, quindi il layout riceve un header minimale:

```erb
<% if authenticated? %>
  <span><%= Current.user.email_address %></span>
  <%= button_to "Sign out", session_path, method: :delete %>
<% else %>
  <%= link_to "Sign in", new_session_path %>
  <%= link_to "Sign up", new_registration_path %>
<% end %>
```

`authenticated?` è l'`helper_method` che il concern `Authentication` espone — è semplicemente `resume_session`, riutilizzato per rispondere a "qualcuno è loggato" senza reindirizzare.

## Provarlo

```bash
bin/dev
```

Registrati con qualsiasi email e password — atterri di nuovo sulla homepage, già loggato, con la tua email nella barra in alto. Esci, poi rientra; prova una password sbagliata e otterrai l'errore comprensibile invece di uno stack trace. Nulla di tutto questo tocca ancora `Service` o `Booking` — il punto di questo episodio è che gli account funzionano, dall'inizio alla fine, prima che ci si costruisca sopra qualsiasi cosa.

## Cosa viene dopo

L'episodio 3 costruisce `Service` e `Category`, e finalmente collega l'associazione `has_many :services` della progettazione dell'episodio 1 — il momento in cui un utente loggato può davvero elencare qualcosa che offre.
