---
layout: post
title: "Stripe con Rails: gestire l'esito"
series: "stripe-with-rails"
episode: 2
lang: it
ref: stripe-handling-the-outcome
permalink: /stripe-handling-the-outcome/
canonical_url: https://antoninoscaffidi.github.io/it/stripe-handling-the-outcome/
image: /assets/images/stripe-ep2-banner.png
date: 2026-09-17 22:44:00 +0200
---

[L'episodio 1]({% post_url 2026-09-16-stripe-setup-and-initiating-a-payment %}) ha portato un cliente da "compra ora" alla pagina di checkout hosted di Stripe, lasciando `StripeCallbacksController#webhook` come uno stub — un endpoint che esisteva solo perché Stripe avesse dove mandare la sua conferma. Questo episodio lo completa per davvero: verificare che un webhook venga genuinamente da Stripe, marcare gli ordini come pagati, e sopravvivere a consegne ripetute — più due bug veri incontrati costruendolo, entrambi utili da conoscere oltre questa singola app.

Il codice è taggato [`episode-2`](https://github.com/AntoninoScaffidi/stripe-with-rails/tree/episode-2) nel repo [stripe-with-rails](https://github.com/AntoninoScaffidi/stripe-with-rails).

## Due canali, stesso principio di Nexi, forma diversa

Una volta che il cliente finisce sulla pagina di checkout di Stripe, Stripe risponde attraverso due canali indipendenti — esattamente la stessa separazione della [serie Nexi]({% post_url 2026-09-13-nexi-xpay-handling-the-outcome %}), solo con nomi diversi:

- **Un redirect del browser** (`success_url`/`cancel_url`) — solo UX. Se il cliente chiude la scheda nell'istante in cui il pagamento finisce, questo redirect semplicemente non avviene mai.
- **Un evento webhook** (`POST /stripe/webhook`) — i server di Stripe stessi chiamano direttamente il tuo backend, senza alcuna dipendenza dal fatto che il browser del cliente sia ancora aperto. È l'unico canale di cui questa app si fida per cambiare lo stato di un ordine.

Nexi instrada entrambi gli esiti attraverso un'unica azione condivisa `#return` che legge un campo `esito` per decidere cosa mostrare. Stripe dà a ciascuno la propria route dedicata — ma `StripeCallbacksController#success` e `#cancel` continuano a non fare altro che un redirect; nessuno dei due si avvicina mai a `Order#mark_paid!`. Solo `#webhook` lo fa.

## `StripeEvent`: un log di audit, non un modello di dominio

```bash
bin/rails generate model StripeEvent stripe_event_id:string event_type:string raw_payload:jsonb
```

```ruby
# app/models/stripe_event.rb
class StripeEvent < ApplicationRecord
  validates :stripe_event_id, presence: true, uniqueness: true
  validates :event_type, presence: true
end
```

Stesso ruolo di `OrderPaymentNotification` nella serie Nexi: un registro di *ogni* evento verificato che Stripe manda, tenuto separato dall'`Order` che potrebbe riguardare. L'indice unique su `stripe_event_id` non è decorazione — è il vero meccanismo di idempotenza, trattato sotto.

## Verificare la firma: stavolta nessun MAC da scrivere a mano

L'episodio 1 aveva già notato che creare una Checkout Session non richiede alcuna firma manuale — il gem autentica le richieste in uscita con un token Bearer. Verificare un webhook in *ingresso* è l'unico punto in cui Stripe chiede davvero di controllare una firma, e il gem ufficiale fa tutta la crittografia:

```ruby
event = Stripe::Webhook.construct_event(payload, sig_header, webhook_secret)
```

Una sola chiamata sostituisce tutto ciò che `Nexi::Gateway#verify_mac` doveva fare a mano — concatenare i campi, `SHA1.hexdigest`, `secure_compare` contro il valore ricevuto. Sotto il cofano `construct_event` fa l'equivalente (HMAC-SHA256 su `"#{timestamp}.#{raw_body}"`, confronto a tempo costante, più un controllo di tolleranza sul timestamp che la versione Nexi scritta a mano non aveva mai avuto), ma come chiamante non vedi nulla di tutto ciò — solo un `Stripe::SignatureVerificationError` se fallisce.

**Un bug incontrato scrivendo i test per questo**: diverse guide Stripe (e versioni più vecchie del gem) citano `Stripe::Webhook.generate_test_header` come il modo per firmare un payload in un test. Non esiste in `stripe` 13.5.1 — un `NoMethodError` lo ha confermato senza ambiguità. L'API reale in questa versione è un livello più sotto:

```ruby
timestamp = Time.current
signature = Stripe::Webhook::Signature.compute_signature(timestamp, payload, secret)
sig_header = Stripe::Webhook::Signature.generate_header(timestamp, signature)
```

Utile da sapere prima di copiare uno snippet di test per i webhook da un tutorial vecchio dentro un `Gemfile` aggiornato.

## `StripeCallbacksController`, per davvero stavolta

```ruby
# app/controllers/stripe_callbacks_controller.rb
class StripeCallbacksController < ActionController::Base
  skip_before_action :verify_authenticity_token, only: [ :webhook ]

  def webhook
    payload = request.body.read
    sig_header = request.headers["Stripe-Signature"]
    webhook_secret = Rails.application.credentials.dig(:stripe, :webhook_secret)

    begin
      event = Stripe::Webhook.construct_event(payload, sig_header, webhook_secret)
    rescue JSON::ParserError, Stripe::SignatureVerificationError => e
      Rails.logger.warn("[Stripe] Webhook rejected: #{e.message}")
      head :bad_request and return
    end

    begin
      StripeEvent.create!(stripe_event_id: event.id, event_type: event.type, raw_payload: event.to_hash)
    rescue ActiveRecord::RecordInvalid, ActiveRecord::RecordNotUnique
      head :ok and return # Stripe ha ritentato una consegna già processata
    end

    handle_event(event)
    head :ok
  end

  def success
    redirect_to root_path
  end

  def cancel
    redirect_to root_path
  end

  private

  def handle_event(event)
    case event.type
    when "checkout.session.completed"
      handle_checkout_completed(event)
    end
  end

  def handle_checkout_completed(event)
    session = event.data.object
    order = Order.find_by(order_number: session.client_reference_id)
    return if order.nil? || !order.pending_payment?

    order.mark_paid!(stripe_payment_intent_id: session.payment_intent, stripe_raw_event: event.to_hash)
  end
end
```

Alcune cose non ovvie a una prima lettura:

**`payload = request.body.read` deve essere il byte grezzo, non `params`.** Stripe firma esattamente i byte che ha inviato; qualunque cosa passi prima attraverso il parsing JSON di Rails (riformattata, ri-serializzata) non corrisponderebbe più alla firma. Leggere il body grezzo funziona qui proprio perché il middleware di parsing dei params di Rails ha già girato prima nello stack e riavvolge lo stream di input una volta finito — quindi `request.body.read` nell'azione ottiene comunque il payload originale per intero, non uno stream vuoto. Confermato per davvero contro un server in esecuzione, non solo in un test — vedi sotto.

**Il fallimento di `StripeEvent.create!` è il meccanismo di idempotenza, non un percorso di errore.** Stripe *ritenterà* una consegna webhook se il tuo endpoint non risponde abbastanza in fretta, quindi lo stesso `event.id` può legittimamente arrivare più di una volta. La validazione di unicità su `stripe_event_id` fa fallire il secondo tentativo — e quel fallimento viene trattato come "già gestito", non come un bug.

**Il secondo bug reale, incontrato scrivendo il test sul retry**: quel rescue inizialmente catturava solo `ActiveRecord::RecordNotUnique` — l'eccezione che solleva una violazione grezza dell'indice *database*. Non scattava mai. `create!` esegue prima le validazioni a livello Ruby, e un controllo `uniqueness: true` fallito solleva `ActiveRecord::RecordInvalid` prima ancora che l'INSERT venga tentato — la mappatura di default delle eccezioni di Rails trasformava poi quell'errore non gestito in un `422` HTTP, non nel `200` idempotente no-op che il codice avrebbe dovuto restituire. `RecordNotUnique` esiste solo come fallback per il caso (qui, puramente ipotetico) in cui quella validazione venga bypassata e sia l'indice unique del database stesso a intercettarlo. Entrambe le eccezioni andavano catturate, non solo una — un'assunzione facile da sbagliare al contrario se si è più abituati alla versione a livello DB di questo pattern.

**La ricerca dell'ordine usa `client_reference_id`, esattamente come pianificato nell'episodio 1.** `session.client_reference_id` è il nostro `order_number`, invariato da quando lo abbiamo impostato costruendo la Checkout Session — questo è l'intero motivo per cui valeva la pena portarlo avanti fin dall'inizio.

## Verificato per davvero — stavolta senza bisogno di mock

L'episodio 1 poteva verificare solo la *forma* della chiamata a Stripe con dei mock, perché creare una Checkout Session reale richiede credenziali API live che questo progetto non ha ancora. Verificare un webhook in ingresso è diverso: è pura crittografia locale, senza alcuna dipendenza dal fatto che l'API di Stripe sia raggiungibile. Quindi invece di un altro mock, questo è stato testato contro un vero `bin/rails server` in esecuzione:

```bash
$ curl -i -X POST http://localhost:3111/stripe/webhook \
    -H "Content-Type: application/json" \
    -H "Stripe-Signature: t=1789677669,v1=0f8c31e3...b5284d6b" \
    --data-binary @webhook_payload.json

HTTP/1.1 200 OK
```

```
Started POST "/stripe/webhook" for ::1
Processing by StripeCallbacksController#webhook
  StripeEvent Create (8.4ms)  INSERT INTO "stripe_events" ...
  Order Update (1.0ms)  UPDATE "orders" SET "status" = 1,
    "stripe_payment_intent_id" = 'pi_live_curl_test', "paid_at" = '2026-09-17 20:41:18' ...
Completed 200 OK in 107ms
```

Un payload firmato a mano con puro Ruby (`OpenSSL::HMAC.hexdigest`, seguendo esattamente lo schema `"#{timestamp}.#{payload}"` di Stripe) contro un processo Puma genuinamente in esecuzione, non un doppio di un request spec. Poi gli altri due rami, allo stesso modo: un header con una firma inventata ha ottenuto un vero `400`, e reinviare la stessa richiesta firmata una seconda volta ha ottenuto un `200` senza una seconda riga in `stripe_events` e senza cambiamenti a `paid_at`.

## Provarlo

```bash
git checkout episode-2
bundle install
bin/rails db:migrate
bin/rails test
```

17 test ora (da 11) — i nuovi coprono un `checkout.session.completed` firmato validamente che marca l'ordine come pagato, una richiesta non firmata/firmata male rifiutata con l'ordine invariato, una consegna ripetuta dello stesso event id che è un vero no-op, e un evento che fa riferimento a un ordine sconosciuto accettato (così Stripe non continua a ritentarlo) senza cambiare nulla.

## Cosa viene dopo

L'episodio 3 copre le parti ancora mancanti prima che questo possa gestire traffico reale: `checkout.session.expired` e `payment_intent.payment_failed`, non solo il percorso felice, più le credenziali live e cosa cambia uscendo dalla modalità test.
