---
layout: post
title: "Stripe con Rails: configurazione e avvio di un pagamento"
series: "stripe-with-rails"
episode: 1
lang: it
ref: stripe-setup-and-initiating-a-payment
permalink: /stripe-setup-and-initiating-a-payment/
canonical_url: https://antoninoscaffidi.github.io/it/stripe-setup-and-initiating-a-payment/
image: /assets/images/stripe-ep1-banner.png
date: 2026-09-16 21:00:00 +0200
---

Questo è il primo episodio di una nuova serie: integrare [Stripe](https://stripe.com) — probabilmente il gateway di pagamento più amico degli sviluppatori che esista — in un'app Ruby on Rails, da zero. Se vieni appena da [Nexi XPay con Rails]({% post_url 2026-09-13-nexi-xpay-setup-and-initiating-a-payment %}), questa serie riusa volutamente lo stesso dominio `Product`/`Order`, così ciò che cambia davvero tra un gateway europeo "in casa" e uno americano moderno, developer-first, risalta chiaro, riga per riga — non sepolto sotto due app diverse che dovresti confrontare a mente da solo.

Il codice è nel repo [stripe-with-rails](https://github.com/AntoninoScaffidi/stripe-with-rails), taggato [`episode-1`](https://github.com/AntoninoScaffidi/stripe-with-rails/tree/episode-1).

## Stripe Checkout: l'equivalente della pagina hosted

Come "Pagamento Semplice" di Nexi, Stripe ha una modalità a pagina hosted chiamata **Checkout**: non raccogli mai tu i dati carta, reindirizzi il cliente su una pagina ospitata da Stripe, e Stripe lo rimanda indietro una volta finito. Stessa logica di evitare il PCI-DSS di prima — i dati carta non toccano mai il tuo server — ma il *meccanismo* per arrivarci è diverso in un modo su cui vale la pena essere precisi.

"Pagamento Semplice" di Nexi è un **POST di un form con campi nascosti**, firmato con un MAC scritto a mano, inviato dal *browser* direttamente al `DispatcherServlet` di Nexi. La tua app Rails non parla mai direttamente con i server di Nexi per avviare un pagamento — costruisce solo un form firmato e lo passa al browser del cliente.

Stripe Checkout funziona al contrario: la tua app Rails fa una **chiamata API server-to-server** a Stripe (`Stripe::Checkout::Session.create`), Stripe crea una sessione e restituisce un URL, e *poi* reindirizzi il browser del cliente su quell'URL. Il tuo server parla con Stripe prima ancora che il cliente veda una pagina di pagamento — ed è anche per questo che non c'è nessun MAC da calcolare a mano: il gem `stripe` ufficiale autentica quella chiamata API per te, usando la tua secret key, come funziona qualunque chiamata API autenticata.

## Chiavi API: la modalità test non richiede nessuna configurazione

Prendile dalla [Dashboard Stripe](https://dashboard.stripe.com/test/apikeys) — la modalità test è disponibile nell'istante in cui crei un account gratuito, senza verifica aziendale, senza attese, a differenza della sandbox di Nexi che richiede credenziali emesse tramite un flusso di backoffice separato. Due chiavi contano qui:

- **Secret key** (`sk_test_...`) — usata solo lato server, per autenticare chiamate API come la creazione di una Checkout Session. Mai esposta al browser.
- **Publishable key** (`pk_test_...`) — sicura da esporre lato client, usata se mai incorpori Stripe.js direttamente (non serve in questo episodio, dato che Checkout è un puro redirect).

Entrambe vanno nelle credenziali cifrate di Rails, mai in `.env`:

```bash
bin/rails credentials:edit
```

```yaml
stripe:
  secret_key: sk_test_...
  publishable_key: pk_test_...
  webhook_secret: whsec_...
```

(`webhook_secret` non si usa fino all'episodio 2 — aggiunta ora così c'è un solo posto dove guardare.)

```ruby
# config/initializers/stripe.rb
Stripe.api_key = Rails.application.credentials.dig(:stripe, :secret_key)
```

Una riga sola. `Stripe.api_key=` è un setter globale, a livello di processo, che il gem `stripe` legge prima di ogni chiamata API — non c'è nessun `Stripe::Client.new(env:)` da gestire tra namespace sandbox e produzione come serviva a `Nexi::Gateway`; la modalità test/live di Stripe è determinata interamente da *quale* secret key imposti (`sk_test_...` vs `sk_live_...`), non da un flag di ambiente esplicito nel tuo codice.

## Il dominio: stessa forma, campi diversi

```bash
bin/rails generate model Product name:string price:decimal purchasable:boolean
bin/rails generate model Order order_number:string product:references status:integer \
  unit_price:decimal total_amount:decimal quantity:integer currency:string \
  guest_first_name:string guest_last_name:string guest_email:string guest_phone:string \
  stripe_checkout_session_id:string stripe_payment_intent_id:string stripe_raw_event:jsonb paid_at:datetime
```

Stessa forma dell'`Order` della serie Nexi — `build_from_product!` che legge il prezzo dal database, `generate_order_number` per un identificatore pubblico non sequenziale, le stesse validazioni per il checkout da ospite. Una vera differenza da segnalare: `currency` di default è `"eur"` **minuscolo**. Nexi voleva `"EUR"` maiuscolo; l'API di Stripe rifiuta apertamente un codice valuta maiuscolo. Piccolo, facile da sbagliare una volta e non pensarci mai più — esattamente il tipo di cosa che vale la pena scrivere da qualche parte.

```ruby
# app/models/order.rb (l'aggiunta specifica per Stripe)
# Stripe si aspetta gli importi come intero nell'unità più piccola della valuta
# (centesimi, per EUR/USD) — mai un decimale.
def unit_price_in_cents
  (unit_price * 100).round
end
```

## `StripeCheckoutService`: costruire la sessione

```ruby
# app/services/stripe_checkout_service.rb
class StripeCheckoutService
  def self.create_session(order:, success_url:, cancel_url:)
    Stripe::Checkout::Session.create(
      mode: "payment",
      line_items: [
        {
          price_data: {
            currency: order.currency,
            product_data: { name: order.product.name },
            unit_amount: order.unit_price_in_cents
          },
          quantity: order.quantity
        }
      ],
      customer_email: order.guest_email,
      client_reference_id: order.order_number,
      success_url: success_url,
      cancel_url: cancel_url
    )
  end
end
```

Alcune cose su cui vale la pena essere precisi:

**`line_items` vuole un prezzo per unità più una quantità, non un totale già moltiplicato.** `unit_amount` è `unit_price_in_cents` — non `total_amount` — perché è Stripe stesso a moltiplicare prezzo × quantità per calcolare l'addebito. Passare qui il totale addebiterebbe silenziosamente il doppio su qualunque ordine con `quantity > 1`.

**`client_reference_id` è la risposta di Stripe al `codTrans` di Nexi** — un tuo riferimento, allegato alla sessione, che torna intatto in ogni evento webhook che la riguarda. È così che l'episodio 2 troverà la riga `Order` giusta quando arriva un webhook: non tramite un qualche ID generato da Stripe coniato *prima* che il tuo ordine esistesse, ma tramite l'identificatore che hai passato tu stesso a Stripe.

**Non c'è nessuna firma esplicita da calcolare qui**, a differenza di `Nexi::Gateway#build_payment_params`. L'intera richiesta è autenticata dall'header `Authorization: Bearer sk_test_...` che il gem `stripe` allega automaticamente a partire da `Stripe.api_key`. Questo è il compromesso di un SDK hosted rispetto a scrivere a mano un protocollo: meno codice, meno cose da sbagliare in modo sottile, ma anche meno visibilità su cosa sta succedendo davvero sulla rete, a meno di andare a cercarla apposta.

## Cos'è davvero un webhook, e perché Checkout ne ha bisogno

Vale la pena rallentare su questo, specialmente se "webhook" è ancora una parola a cui annuisci più che una che sapresti spiegare da zero.

Una normale chiamata API è qualcosa che **inizi tu**: la tua app Rails decide di volere qualcosa da Stripe, manda una richiesta, e riceve una risposta — come `Stripe::Checkout::Session.create` sopra. Funziona perché sei *tu* a sapere quando ti serve un'informazione, e puoi semplicemente chiederla.

Un **webhook è la direzione opposta.** Dopo che il cliente finisce di pagare sulla pagina hosted di Stripe, è *Stripe* a scoprirlo per primo — il tuo server Rails non ha modo di sapere, da dove si trova, che un cliente specifico su un browser specifico ha appena inserito con successo un numero di carta tre secondi fa. Non c'è nulla da interrogare periodicamente: potresti chiedere a Stripe "questa sessione è stata pagata?" una volta al secondo per sempre, ma sarebbe uno spreco, lento ad accorgersene, e non scala. Quindi invece dai a Stripe un URL — `POST /stripe/webhook` in questa app — e Stripe **ti chiama**, nel momento in cui succede qualcosa, con un payload JSON che descrive l'evento. Il tuo server non chiede; Stripe dice.

Quell'inversione crea un problema che una normale chiamata API non ha mai: **chiunque su internet può mandare una richiesta POST a un URL pubblico.** Quando la tua app Rails chiama Stripe, la risposta è affidabile perché sei tu ad aver iniziato la connessione verso un host che già sai essere davvero Stripe. Quando è Stripe a chiamare *te*, i ruoli si invertono — il tuo endpoint `/stripe/webhook` deve verificare in modo indipendente che una data richiesta in arrivo venga davvero da Stripe e non da qualcuno che ha trovato l'URL e sta fingendo. È esattamente il ruolo che giocava il MAC scritto a mano di Nexi al ritorno, ed è esattamente quello che fa qui la verifica della firma di Stripe (`Stripe::Webhook.construct_event`, coperta per intero nel prossimo episodio) — stesso problema di fondo, due soluzioni dalla forma molto diversa, che varrà la pena confrontare direttamente una volta che saranno entrambe sulla pagina.

Per ora, `StripeCallbacksController#webhook` di questo episodio è uno stub — l'endpoint esiste, quindi c'è una route per cui Stripe può chiamare, ma non verifica ancora nulla né agisce su ciò che arriva:

```ruby
# app/controllers/stripe_callbacks_controller.rb
class StripeCallbacksController < ActionController::Base
  skip_before_action :verify_authenticity_token, only: [ :webhook ]

  def webhook
    head :ok
  end

  def success
    redirect_to root_path
  end

  def cancel
    redirect_to root_path
  end
end
```

`success` e `cancel` sono l'*altro* canale — il redirect del browser, solo UX, esattamente come `#return` di Nexi. Stripe rimanda il browser del cliente su `success_url`/`cancel_url` a prescindere dal fatto che il webhook sia già arrivato, sia già stato processato, o non si sia ancora fatto vedere affatto. Stessa forma a due canali di Nexi, stesso motivo: il redirect è comodo ma inaffidabile (una scheda chiusa significa che non scatta mai), il webhook è l'unico canale davvero affidabile abbastanza da segnare un ordine come pagato.

## Route e controller

```ruby
# config/routes.rb
resources :products, only: [ :index, :show ]
resources :orders, only: [ :new, :create ], param: :order_number

get "stripe/success", to: "stripe_callbacks#success", as: :stripe_success
get "stripe/cancel", to: "stripe_callbacks#cancel", as: :stripe_cancel
post "stripe/webhook", to: "stripe_callbacks#webhook", as: :stripe_webhook
```

```ruby
# app/controllers/orders_controller.rb
def create
  redirect_to(product_path(@product)) and return unless @product.purchasable?

  @order = Order.build_from_product!(
    product: @product, guest_attributes: order_params.to_h,
    quantity: order_params[:quantity].presence || 1
  )

  session = StripeCheckoutService.create_session(
    order: @order,
    success_url: stripe_success_url(order_number: @order.order_number),
    cancel_url: stripe_cancel_url(order_number: @order.order_number)
  )
  @order.update!(stripe_checkout_session_id: session.id)

  redirect_to session.url, allow_other_host: true
rescue ActiveRecord::RecordInvalid => e
  @order = e.record
  render :new, status: :unprocessable_entity
end
```

`allow_other_host: true` è obbligatorio, non opzionale — Rails 7+ solleva `ActionController::Redirecting::UnsafeRedirectError` su un `redirect_to` puntato a un host che la tua app non riconosce come proprio, come difesa contro le vulnerabilità open-redirect. `session.url` punta legittimamente a `checkout.stripe.com`, quindi il controllo di sicurezza va disattivato esplicitamente per questo caso preciso e voluto — non disabilitato globalmente.

## Un bug incontrato stasera, due volte: un gem appena uscito che rompe tutto al primo boot

Prima ancora che quanto sopra potesse renderizzare, il primissimo caricamento della pagina ha sollevato:

```
ArgumentError (wrong number of arguments (given 2, expected 1))
app/views/layouts/application.html.erb:10 — <%= csp_meta_tag %>
```

Stessa causa di fondo di [un bug della serie Nexi]({% post_url 2026-09-13-nexi-xpay-handling-the-outcome %}): `json` `3.0.2`, appena pubblicata, rompe la chiamata di `ActiveSupport::JSON.decode` verso `JSON.parse` su questa versione di Rails. L'altra volta era emerso solo quando le colonne `jsonb` venivano davvero scritte, nell'episodio 2 — stavolta ha colpito subito, da `csp_meta_tag` stesso, prima ancora che girasse una sola riga di codice di questo episodio. Stessa correzione:

```ruby
# Gemfile
gem "json", "< 3.0"
```

E un secondo bug, non collegato, trovato scrivendo i test di questo episodio: `Stripe::Checkout::Session.create` va mockato per testare `StripeCheckoutService` senza una chiave API vera, usando `Object#stub` di Minitest — solo che non era disponibile. **Minitest `6.0`, anch'essa appena pubblicata, ha rimosso `Minitest::Mock` dal gem core** ("moving minitest/mock.rb to its own gem", dal suo stesso changelog). Stesso tipo di problema del gem `json` — una release appena uscita, risolta da `bundle install` senza alcun pin in essere — stessa correzione:

```ruby
# Gemfile
gem "minitest", "~> 5.25"
```

Due gem non collegati tra loro, due breaking change non collegati, entrambi appena rilasciati, entrambi incontrati al primissimo `bundle install` di un'app nuova di zecca, la stessa sera. Vale la pena interiorizzarlo come uno schema reale, non una coincidenza da alzare le spalle: un `Gemfile` senza pin è una scommessa che qualunque versione `bundle install` risolva *oggi* continuerà a funzionare — e in un giorno qualunque, quella scommessa può semplicemente andare persa.

## Un bug più sottile: Turbo Drive e un redirect cross-origin

Questo ha richiesto una vera indagine per essere rintracciato, ed è più interessante di un semplice "aggiungi di nuovo `data: turbo: false`" — il *motivo* per cui serve qui è genuinamente diverso dal gotcha di Turbo della serie Nexi.

Inviare il form di checkout non faceva nulla. Non un errore — nulla. Nessuna navigazione, nessun errore in console, il bottone semplicemente... non succedeva niente. Ma i log del server raccontavano un'altra storia:

```
Started POST "/orders" for ::1
Processing by OrdersController#create as TURBO_STREAM
...
Redirected to https://checkout.stripe.com/...
Completed 302 Found in 277ms
```

Il server ha fatto il suo lavoro. Ha costruito l'ordine, creato la sessione Stripe, e emesso un vero `302` verso la vera pagina di checkout hosted di Stripe. Il browser semplicemente non ci è mai andato. Per confermare che non fosse qualcosa di specifico dell'URL di Stripe, ho temporaneamente sostituito la destinazione del redirect con un semplice `https://example.com` — stesso risultato, fermo morto, la pagina non si è mai mossa.

Ecco il meccanismo vero: Turbo Drive intercetta gli invii di form ordinari e li ripete come una richiesta `fetch()` invece che come una navigazione completa del browser — è così che evita un reload completo della pagina per i POST di form dello stesso sito. Quando quella risposta `fetch()` è un redirect, il livello di rete del browser lo segue in modo trasparente *a livello HTTP* — ma la risposta con cui `fetch()` si risolve alla fine è il corpo proveniente dalla destinazione *finale* (`checkout.stripe.com`, o `example.com` nel test isolato), indirizzato a una richiesta che Turbo Drive stesso ha iniziato, non una vera navigazione di pagina di primo livello. Turbo prova allora a processare quella risposta come se fosse altro contenuto di *questa* app — e dato che non è né contenuto Turbo Stream valido né nulla che Turbo riconosca, semplicemente rinuncia in silenzio. Il redirect è avvenuto, tecnicamente, interamente dentro una chiamata `fetch()` di cui la barra degli indirizzi non ha mai saputo nulla.

La correzione ha la stessa forma del gotcha di Turbo della serie Nexi, raggiunta per un motivo diverso:

```erb
<%= form_with url: orders_path, method: :post, data: { turbo: false } do |f| %>
```

`data: { turbo: false }` fa sì che questo form venga inviato alla vecchia maniera — una vera navigazione completa del browser, non una `fetch()` intercettata da Turbo Drive. Una vera navigazione segue un redirect cross-origin esattamente come ti aspetteresti: la barra degli indirizzi si aggiorna, e il cliente atterra davvero sulla pagina di Stripe. Confermato reindirizzando temporaneamente a `example.com` e osservando il browser arrivarci davvero una volta messo `turbo: false` — prima di allora, stesso form, stesso redirect lato server, la scheda semplicemente non si muoveva mai.

La lezione generale, da ricordare indipendentemente da Stripe o Nexi nello specifico: **qualunque form la cui azione finisce in un redirect verso un'origine diversa ha bisogno di `data: { turbo: false }`.** I redirect same-origin funzionano bene sotto Turbo Drive anche senza — è specificamente il caso cross-origin dove un redirect basato su `fetch()` e una vera navigazione del browser smettono di essere equivalenti.

## Provarlo

```bash
git clone https://github.com/AntoninoScaffidi/stripe-with-rails.git
cd stripe-with-rails
git checkout episode-1
bundle install
bin/rails db:prepare
bin/rails db:seed
bin/rails test
```

11 test, e nessuno di essi fa una vera chiamata di rete — sia `StripeCheckoutServiceTest` che `OrdersControllerTest` mockano `Stripe::Checkout::Session.create` con `Minitest::Mock`, verificando che i parametri mandati a Stripe siano corretti (prezzo in centesimi, non decimale; `client_reference_id` impostato al nostro `order_number`) senza bisogno di una chiave API vera solo per far girare la suite. Aggiungi le tue chiavi di test e il form di checkout reindirizza davvero a una vera pagina di pagamento hosted da Stripe, funzionante — verificato stasera, incluso rintracciare il problema di Turbo Drive sopra osservandolo fallire contro una vera destinazione di redirect prima di correggerlo.

## Cosa viene dopo

L'episodio 2 copre l'altro lato: i webhook event-based di Stripe (non un singolo campo `esito` come ha Nexi, ma eventi nominati come `checkout.session.completed`), la verifica della firma tramite `Stripe::Webhook.construct_event` dell'SDK ufficiale — e un confronto diretto, riga per riga, con la verifica MAC scritta a mano da zero nella serie Nexi.
