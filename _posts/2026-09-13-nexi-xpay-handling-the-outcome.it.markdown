---
layout: post
title: "Nexi XPay con Rails: gestire l'esito"
series: "nexi-xpay-with-rails"
episode: 2
lang: it
ref: nexi-xpay-handling-the-outcome
permalink: /nexi-xpay-handling-the-outcome/
canonical_url: https://antoninoscaffidi.github.io/it/nexi-xpay-handling-the-outcome/
image: /assets/images/nexi-xpay-ep2-banner.png
date: 2026-09-13 07:30:00 +0200
---

L'[episodio 1]({% post_url 2026-09-13-nexi-xpay-setup-and-initiating-a-payment %}) ha portato un cliente da "clicca compra" alla pagina di pagamento hosted di Nexi, lasciando `NexiCallbacksController` come uno stub — due endpoint esistenti solo perché Nexi avesse un posto dove mandare il browser e la notifica server-to-server. Questo episodio li completa per davvero: verificare ciò che torna indietro, segnare gli ordini come pagati, e sopravvivere ai modi in cui un webhook può andare storto nella pratica.

Il codice è taggato [`episode-2`](https://github.com/AntoninoScaffidi/nexi-xpay-with-rails/tree/episode-2) nel repo [nexi-xpay-with-rails](https://github.com/AntoninoScaffidi/nexi-xpay-with-rails).

## Due canali, e solo uno dei due è affidabile

Una volta che un cliente finisce sulla pagina di pagamento di Nexi, Nexi comunica con la tua app attraverso due canali completamente indipendenti:

- **Un redirect del browser** (`url`/`url_back`) — il browser del cliente viene rimandato sul tuo sito. È solo UX: se il cliente chiude la scheda nell'istante in cui il pagamento finisce, questo redirect semplicemente non arriva mai.
- **Una notifica server-to-server** (`urlpost`) — i server di Nexi stessi fanno un POST dell'esito direttamente al tuo backend, senza dipendere dal fatto che il browser del cliente sia ancora aperto. È l'unico canale davvero affidabile, e l'unico di cui questa app si fida per cambiare lo stato di un ordine.

Ecco perché `url` e `url_back` puntano entrambi alla *stessa* route, `nexi_return` — l'azione legge `esito` (`OK`/`KO`/`ANNULLO`/`ERRORE`) solo per decidere quale pagina mostrare al cliente, ma **non scrive mai** lo stato dell'ordine. Solo `#notify` lo fa.

## `OrderPaymentNotification`: un log di audit, non un modello di dominio

```bash
bin/rails generate model OrderPaymentNotification order:references \
  raw_params:jsonb mac_verified:boolean processed:boolean outcome:string
```

```ruby
# app/models/order_payment_notification.rb
class OrderPaymentNotification < ApplicationRecord
  belongs_to :order
end
```

Volutamente minimale — nessuna logica affatto. Esiste per registrare *ogni* notifica che Nexi manda, incluse quelle con firma non valida, prima che venga presa qualunque decisione su cosa farne. L'interpretazione vive nel controller; questo modello è un registro di eventi, non un'entità di dominio con un proprio comportamento.

## `Nexi::Gateway#verify_mac`: l'altra metà della classe

L'episodio 1 ha mostrato `Nexi::Gateway` mentre costruisce e firma la richiesta in uscita. Questo episodio aggiunge il metodo che controlla la firma al ritorno:

```ruby
# app/services/nexi/gateway.rb (aggiunte)
def verify_mac(params)
  received = params["mac"] || params[:mac]
  return false if received.blank?

  ActiveSupport::SecurityUtils.secure_compare(compute_response_mac(params), received.to_s)
end

private

def compute_response_mac(params)
  value = ->(key) { (params[key] || params[key.to_sym]).to_s }
  string_to_sign =
    "codTrans=#{value.call('codTrans')}esito=#{value.call('esito')}importo=#{value.call('importo')}" \
    "divisa=#{value.call('divisa')}data=#{value.call('data')}orario=#{value.call('orario')}" \
    "codAut=#{value.call('codAut')}#{mac_key}"
  Digest::SHA1.hexdigest(string_to_sign)
end
```

Il messaggio di esito firma più campi di quanti ne firmasse la richiesta (`esito`, `data`, `orario`, `codAut` sono tutti nuovi), ma il meccanismo è identico: concatena, esegui l'hash, confronta. Due dettagli su cui vale la pena essere precisi:

- **`ActiveSupport::SecurityUtils.secure_compare` invece di `==`.** Un confronto tra stringhe normale si interrompe non appena trova il primo carattere diverso, il che significa che il tempo del confronto rivela informazioni su *quanti caratteri erano corretti* — un timing attack. `secure_compare` impiega sempre lo stesso tempo, indipendentemente da dove le stringhe divergono per prime.
- **`verify_mac` restituisce `false` anche quando `mac` è del tutto assente dai parametri**, invece di sollevare un errore o saltare il controllo. Non deve mai esistere un percorso in cui "nessuna firma" venga trattato come "firma valida".

## `NexiCallbacksController`, per davvero stavolta

```ruby
# app/controllers/nexi_callbacks_controller.rb
class NexiCallbacksController < ActionController::Base
  skip_before_action :verify_authenticity_token, only: [ :notify ]

  def notify
    order = Order.find_by(order_number: params[:codTrans])
    if order.nil?
      Rails.logger.error("[Nexi] Notification for unknown codTrans: #{params[:codTrans]}")
      head :ok and return
    end

    notification = order.order_payment_notifications.create!(raw_params: raw_notification_params)

    gateway = Nexi::Gateway.new
    mac_valid = gateway.verify_mac(raw_notification_params)
    notification.update!(mac_verified: mac_valid)

    unless mac_valid
      notification.update!(processed: true, outcome: "invalid_mac")
      head :ok and return
    end

    case params[:esito]
    when "OK"
      if order.pending_payment?
        order.mark_paid!(
          nexi_transaction_id: params[:codTrans], nexi_authorization_code: params[:codAut],
          nexi_raw_response: raw_notification_params
        )
        notification.update!(processed: true, outcome: "paid")
      else
        notification.update!(processed: true, outcome: "duplicate") # idempotenza
      end
    else
      order.update!(status: :payment_failed) if order.pending_payment?
      notification.update!(processed: true, outcome: params[:esito].to_s.downcase.presence || "unknown")
    end

    head :ok
  end

  def return
    order = Order.find_by(order_number: params[:codTrans])
    order.nil? ? redirect_to(root_path) : redirect_to(order_path(order))
  end

  private

  def raw_notification_params
    params.to_unsafe_h.except("controller", "action")
  end
end
```

Alcune cose qui sono determinanti in modi non ovvi a una prima lettura:

**`skip_before_action :verify_authenticity_token` è limitato solo a `:notify`.** È un semplice POST da un server esterno, senza cookie di sessione e senza token CSRF — controllarlo rifiuterebbe ogni notifica legittima. `#return` resta protetta, anche se lì è comunque irrilevante: è una `GET`, e Rails impone il CSRF solo sui verbi che modificano lo stato.

**L'ordine delle operazioni dentro `#notify` non è arbitrario.** La notifica grezza viene loggata *prima* che il MAC venga controllato, non dopo. Una notifica con firma non valida viene comunque scritta in `order_payment_notifications` — è esattamente il record che vorresti avere più avanti ("Nexi ci ha davvero contattato per questo `codTrans`, ma con una firma sbagliata: perché?"). Validare prima e loggare solo ciò che passa butterebbe via proprio i casi più utili da indagare.

**`mark_paid!` è un no-op su un ordine già pagato** (`return if paid?`, dal modello `Order` dell'episodio 1), e il ramo `else` sopra registra quel caso come `"duplicate"` invece di rielaborarlo. Questo conta perché Nexi *ritenterà* una notifica se il tuo endpoint non risponde `200` abbastanza in fretta, o non risponde affatto — lo stesso messaggio `codTrans=OK` può legittimamente arrivare più di una volta, e niente in questo dovrebbe far scattare una seconda email di conferma o contare due volte qualcosa.

**`raw_notification_params` non è mass assignment su un modello `ActiveRecord`** — finisce solo in una colonna `jsonb` di audit, mai in `update`/`create` su `Order` stesso, quindi non c'è un percorso da "qualunque cosa Nexi abbia mandato" a sovrascrivere silenziosamente campi che non doveva mai toccare.

## Stato dell'ordine, e perché di solito dice "in attesa"

```erb
<%# app/views/orders/show.html.erb %>
<% if @order.paid? %>
  <%# spunta verde, "Pagamento confermato" %>
<% elsif @order.pending_payment? %>
  <%# "Pagamento in corso", invita a ricaricare %>
<% else %>
  <%# payment_failed / cancelled / refunded %>
<% end %>
```

Nella pratica, `pending_payment?` è il ramo che un cliente vede più spesso, anche dopo un pagamento riuscito. Il redirect del browser (`#return`) atterra frequentemente qui *prima* che il webhook asincrono `#notify` abbia effettivamente segnato l'ordine come pagato — i due canali non hanno alcuna garanzia di ordine relativo tra loro. Ecco perché il testo invita esplicitamente a ricaricare invece di promettere un aggiornamento in tempo reale; non c'è polling né websocket in questo episodio, di proposito, per tenere la superficie piccola.

## Un bug vero: una release appena uscita di `json` che rompe `ActiveSupport::JSON.decode`

Non un bug di logica — uno di ambiente, incontrato nel momento esatto in cui il codice di questo episodio ha provato per la prima volta a scrivere su una colonna `jsonb`:

```
ArgumentError: wrong number of arguments (given 2, expected 1)
  .../json-3.0.2/lib/json/common.rb:296:in 'parse'
  .../activesupport-8.1.3.1/lib/active_support/json/decoding.rb:25:in 'ActiveSupport::JSON.decode'
  .../activerecord-8.1.3.1/lib/active_record/type/json.rb:17:in 'ActiveRecord::Type::Json#deserialize'
```

`json` `3.0.2` — pubblicata su RubyGems pochissimo tempo prima — ha cambiato la firma di `JSON.parse` in un modo che `ActiveSupport::JSON.decode` non si aspetta su questa versione di Rails, e `bundle install` ci era finito dentro senza alcun pin esplicito nel `Gemfile`. La correzione è una riga:

```ruby
# Gemfile
gem "json", "< 3.0"
```

seguita da `bundle update json`. Vale la pena ricordarla come lezione generale, non solo su Nexi: una dipendenza transitiva non pinnata può rompere un'app funzionante con nient'altro che un `bundle install` che si imbatte in una release di un gem uscita un'ora prima. Se un percorso di codice che tocca `jsonb`/JSON solleva improvvisamente un `ArgumentError` sul numero di argomenti senza che tu abbia cambiato nulla, controlla cosa è stato davvero risolto prima di dare per scontato che il bug sia tuo.

## Provarlo

```bash
git checkout episode-2
bundle install
bin/rails db:migrate
bin/rails test
```

19 test ora (erano 11) — i nuovi coprono `verify_mac` che accetta una risposta firmata correttamente e ne rifiuta una manomessa o senza firma, una notifica `OK` valida che segna l'ordine come pagato, la stessa notifica arrivata due volte che non viene rielaborata, un MAC non valido che lascia l'ordine intatto, e `#return` che non scrive mai lo stato in nessun caso.

## Cosa viene dopo

L'[episodio 3]({% post_url 2026-09-13-nexi-xpay-testing-and-going-to-production %}) fa girare tutto il flusso sulla vera sandbox di Nexi dall'inizio alla fine — un tunnel pubblico con ngrok, un vero pagamento con carta di test, e la notifica che arriva da sola, senza alcun intervento — più cosa manca ancora prima di passare in produzione.
