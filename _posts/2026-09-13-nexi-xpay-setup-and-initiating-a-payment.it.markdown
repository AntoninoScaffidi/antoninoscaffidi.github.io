---
layout: post
title: "Nexi XPay con Rails: configurazione e avvio di un pagamento"
series: "nexi-xpay-with-rails"
episode: 1
lang: it
ref: nexi-xpay-setup-and-initiating-a-payment
permalink: /nexi-xpay-setup-and-initiating-a-payment/
canonical_url: https://antoninoscaffidi.github.io/it/nexi-xpay-setup-and-initiating-a-payment/
image: /assets/images/nexi-xpay-ep1-banner.png
date: 2026-09-13 07:00:00 +0200
---

Questo è il primo episodio di una nuova serie: integrare [Nexi XPay](https://ecommerce.nexi.it/specifiche-tecniche/) — uno dei gateway di pagamento con carta più comuni per gli esercenti italiani — in un'app Ruby on Rails, da zero, dall'inizio alla fine. Tutto in questa serie è verificato su un vero ambiente sandbox Nexi, incluso un vero pagamento di prova andato a buon fine.

Il codice è nel repo [nexi-xpay-with-rails](https://github.com/AntoninoScaffidi/nexi-xpay-with-rails), taggato [`episode-1`](https://github.com/AntoninoScaffidi/nexi-xpay-with-rails/tree/episode-1). Questo episodio porta un cliente da "clicca compra" alla pagina di pagamento di Nexi, con il prezzo letto dal database, mai da ciò che manda il browser.

## Quattro modi per integrare, e perché "Pagamento Semplice"

La documentazione Nexi offre diverse modalità di integrazione, e quella giusta dipende interamente da quanto vuoi gestire tu stesso:

| Modalità | Dati carta | Complessità |
|---|---|---|
| **Pagamento Semplice** | Mai sul tuo server (pagina hosted) | Bassa — un POST + una firma |
| Pagamento OneClick | Mai sul tuo server, ma serve un token da un pagamento "Semplice" precedente | Media |
| Server-to-Server | **Transitano dal tuo server** | Alta |
| Lightbox / XPay Build | Mai sul tuo server, ma via iframe/JS SDK | Medio-alta |

Il fattore decisivo è **Server-to-Server**: richiede che il numero di carta transiti — anche solo per un istante — dal tuo backend, il che porta la tua app nello scope della certificazione PCI-DSS. È un costo reale (audit, requisiti infrastrutturali) non giustificato a meno che tu non gestisca volumi seri. **Pagamento Semplice** — reindirizzare il browser del cliente su una pagina ospitata da Nexi — significa che i dati carta non raggiungono mai il tuo server. La tua app vede solo l'esito finale, e anche allora solo una versione mascherata del numero di carta (`453997******0006`), mai il numero in chiaro. È questa la modalità che costruisce questa serie.

## Il dominio: `Product` e `Order`

Due modelli, volutamente piccoli:

```bash
bin/rails generate model Product name:string price:decimal purchasable:boolean
bin/rails generate model Order order_number:string product:references status:integer \
  unit_price:decimal total_amount:decimal quantity:integer currency:string \
  guest_first_name:string guest_last_name:string guest_email:string guest_phone:string \
  nexi_transaction_id:string nexi_authorization_code:string nexi_raw_response:jsonb paid_at:datetime
```

`Product#purchasable` è un semplice booleano, default `false` — è l'interruttore che decide se un prodotto mostra un bottone "Compra ora". `Order` porta sia un `unit_price` che un `total_amount` invece di limitarsi a leggere `product.price` al volo: una volta che un ordine esiste, è la registrazione di ciò che è stato effettivamente concordato, e un cambio di prezzo successivo sul prodotto non deve mai riscriverlo.

```ruby
# app/models/order.rb
class Order < ApplicationRecord
  belongs_to :product

  enum :status, { pending_payment: 0, paid: 1, payment_failed: 2, cancelled: 3, refunded: 4 }

  before_validation :generate_order_number, on: :create

  validates :order_number, presence: true, uniqueness: true
  validates :unit_price, :total_amount, numericality: { greater_than: 0 }
  validates :guest_first_name, :guest_last_name, presence: true
  validates :guest_email, presence: true, format: { with: VALID_EMAIL_REGEX }

  def to_param
    order_number
  end

  # L'unico punto che legge il prezzo dal database e calcola il totale.
  def self.build_from_product!(product:, guest_attributes:, quantity: 1)
    raise ArgumentError, "Product is not purchasable" unless product.purchasable?

    unit_price = product.price
    create!(
      product: product, quantity: quantity.to_i, unit_price: unit_price,
      total_amount: unit_price * quantity.to_i,
      **guest_attributes
    )
  end

  private

  def generate_order_number
    self.order_number ||= loop do
      candidate = SecureRandom.alphanumeric(12).upcase
      break candidate unless Order.exists?(order_number: candidate)
    end
  end
end
```

`build_from_product!` è l'unico punto di passaggio obbligato per ogni ordine, ed è il motivo per cui un prezzo manomesso in un POST del form non può mai cambiare davvero quanto viene addebitato — `unit_price` arriva da `product.price`, punto, a prescindere da cos'altro manda un client malevolo insieme. `order_number` — 12 caratteri alfanumerici maiuscoli casuali, verificati contro collisioni nel database invece di essere semplicemente presunti unici per fortuna — è ciò che compare negli URL pubblici (`to_param` sovrascrive il default), mai l'`id` sequenziale del database.

## Credenziali, e il gotcha sandbox-vs-produzione

Le credenziali Nexi — un `alias`, una `mac_key`, e l'URL dell'endpoint — vivono nelle credenziali cifrate di Rails, mai in `.env` o nel codice sorgente:

```bash
bin/rails credentials:edit
```

```yaml
nexi:
  sandbox:
    alias: YOUR_SANDBOX_ALIAS
    mac_key: your_sandbox_mac_key
    pay_url: https://int-ecommerce.nexi.it/ecomm/ecomm/DispatcherServlet
```

**Un vero gotcha che vale la pena conoscere prima di incapparci da soli**: l'alias per la sandbox (`int-ecommerce.nexi.it`) *non* è lo stesso alias che avrai per la produzione. È un terminale di test separato, che si trova nel backoffice Nexi sotto **Area test → Pagamento Semplice/OneClick**, ed è facile prendere quello sbagliato — l'alias del terminale di produzione, con la ragione sociale e l'indirizzo reali — e vedersi rifiutare ogni richiesta sandbox con `"Alias non valido per l'operazione richiesta"`. La stessa pagina del backoffice fornisce anche i numeri di carta di test (l'episodio 3 li usa per un pagamento vero).

## `Nexi::Gateway`: l'unico punto che conosce il protocollo

Tutto ciò che è specifico di Nexi vive in un unico service object — costruire la richiesta, e più avanti (episodio 2) verificare ciò che torna indietro:

```ruby
# app/services/nexi/gateway.rb
module Nexi
  class Gateway
    class MissingCredentials < StandardError; end

    def initialize(env: Rails.env.production? ? :production : :sandbox)
      @credentials = Rails.application.credentials.dig(:nexi, env)
      raise MissingCredentials, "Missing Nexi credentials for environment #{env}" if @credentials.blank?
    end

    def payment_form_url
      @credentials[:pay_url]
    end

    def build_payment_params(order, return_url:, cancel_url:, notify_url:)
      cod_trans = order.order_number
      divisa = order.currency
      importo = amount_in_cents(order.total_amount)

      {
        "alias" => alias_code, "importo" => importo, "divisa" => divisa, "codTrans" => cod_trans,
        "url" => return_url, "url_back" => cancel_url, "urlpost" => notify_url,
        "mail" => order.guest_email, "nome" => order.guest_first_name, "cognome" => order.guest_last_name,
        "mac" => compute_request_mac(cod_trans: cod_trans, divisa: divisa, importo: importo)
      }.compact
    end

    private

    def alias_code = @credentials[:alias]
    def mac_key = @credentials[:mac_key]
    def amount_in_cents(decimal_amount) = (decimal_amount * 100).round.to_s

    def compute_request_mac(cod_trans:, divisa:, importo:)
      Digest::SHA1.hexdigest("codTrans=#{cod_trans}divisa=#{divisa}importo=#{importo}#{mac_key}")
    end
  end
end
```

Due dettagli su cui vale la pena essere precisi:

- **Il `raise` sulle credenziali mancanti non è difensivo per il gusto di esserlo.** Senza, un deploy che ha dimenticato di impostare `nexi.production` costruirebbe silenziosamente una richiesta con valori `nil` e manderebbe un cliente su una pagina di pagamento rotta. Con quel `raise`, l'app fallisce rumorosamente al primissimo tentativo di checkout invece che in silenzio.
- **`amount_in_cents` conta perché il campo `importo` di Nexi è un intero senza separatore decimale** — `€42,50` deve diventare la stringa `"4250"`, non `"42.50"`. Sbagliare questo non solleva un errore; addebita semplicemente l'importo sbagliato.

`compute_request_mac` è il **MAC** (Message Authentication Code) — un hash SHA1 di alcuni campi concatenati senza separatori, più la chiave segreta, calcolato esattamente come lo calcoleranno in modo indipendente i server di Nexi per confermare che la richiesta non è stata manomessa in transito. Questa metà del gateway costruisce e firma la richiesta *in uscita*; l'episodio 2 aggiunge l'altra metà, che verifica quella *in entrata*.

## Route e `OrdersController`

```ruby
# config/routes.rb
resources :products, only: [ :index, :show ]
resources :orders, only: [ :new, :create ], param: :order_number

get "nexi/return", to: "nexi_callbacks#return", as: :nexi_return
post "nexi/notify", to: "nexi_callbacks#notify", as: :nexi_notify
```

`nexi_return` e `nexi_notify` devono esistere come route fin dal primo giorno — `OrdersController#create` genera URL per entrambe — anche se il controller dietro di esse, `NexiCallbacksController`, è ancora uno stub in questo episodio (`#notify` restituisce solo `head :ok`, `#return` reindirizza solo alla home). Nexi ha bisogno di un posto reale dove mandare il browser e la notifica server-to-server; cosa fanno davvero quegli endpoint con ciò che arriva è il problema del prossimo episodio.

```ruby
# app/controllers/orders_controller.rb
class OrdersController < ApplicationController
  before_action :set_product, only: [ :new, :create ]

  def new
    redirect_to(product_path(@product)) and return unless @product.purchasable?
    @order = Order.new
  end

  def create
    redirect_to(product_path(@product)) and return unless @product.purchasable?

    @order = Order.build_from_product!(
      product: @product, guest_attributes: order_params.to_h,
      quantity: order_params[:quantity].presence || 1
    )

    gateway = Nexi::Gateway.new
    @payment_form_url = gateway.payment_form_url
    @payment_params = gateway.build_payment_params(
      @order, return_url: nexi_return_url, cancel_url: nexi_return_url, notify_url: nexi_notify_url
    )
  rescue ActiveRecord::RecordInvalid => e
    @order = e.record
    render :new, status: :unprocessable_entity
  end

  private

  def set_product
    @product = Product.find(params[:product_id])
  end

  def order_params
    params.permit(:quantity, :guest_first_name, :guest_last_name, :guest_email, :guest_phone)
  end
end
```

`purchasable?` viene verificato sia in `#new` sia in `#create`, non solo una volta — tra il momento in cui un cliente apre il form e quello in cui lo invia, quel flag potrebbe essere disattivato. Il secondo controllo chiude quella finestra. E nota cosa `order_params` lascia volutamente fuori: nessun `:total_amount`, nessun `:unit_price`. Non è filtrato come una regola di sicurezza speciale — semplicemente non è mai nella lista dei permessi, quindi `params.permit` lo scarta silenziosamente prima che arrivi a `build_from_product!`. Un `total_amount: "1"` malevolo nel corpo del POST non ha dove andare.

## Due gotcha nelle viste che sono costati tempo di debug reale

Il form di checkout:

```erb
<%# app/views/orders/new.html.erb %>
<%= form_with url: orders_path, method: :post, data: { turbo: false } do |f| %>
  <%= f.hidden_field :product_id, value: @product.id %>
  <%= f.text_field :guest_first_name, required: true %>
  <%# ... %>
  <%= f.submit "Go to payment" %>
<% end %>
```

**Gotcha #1 — Turbo Drive inghiotte silenziosamente il submit.** Senza `data: { turbo: false }`, Rails logga la richiesta come `Processing ... as TURBO_STREAM`, e non succede nulla: nessun errore, nessun redirect, solo una pagina che sembra non aver fatto nulla al click. La correzione è una riga sola, ma trovarla significa davvero leggere `log/development.log` mentre si clicca invia — la riga `as TURBO_STREAM` è la traccia.

Il passaggio automatico verso Nexi:

```erb
<%# app/views/orders/create.html.erb %>
<%# form_with/form_tag forzano sempre accept-charset="UTF-8": Nexi richiede
    ISO-8859-1, quindi qui il tag <form> va scritto a mano. %>
<form id="nexi-payment-form" method="post" action="<%= @payment_form_url %>" accept-charset="ISO-8859-1">
  <% @payment_params.each do |name, value| %>
    <%= hidden_field_tag name, value, id: nil %>
  <% end %>
</form>
<script>document.getElementById("nexi-payment-form").submit();</script>
```

**Gotcha #2 — Rails impone sempre `accept-charset="UTF-8"` su ogni form.** Nexi ha bisogno di `ISO-8859-1` perché i nomi accentati passino correttamente, ma `form_with`/`form_tag` impostano `accept-charset` **incondizionatamente** dentro `action_view/helpers/form_tag_helper.rb` — qualsiasi valore passato in `html: {...}` viene sovrascritto subito dopo. L'unico modo per aggirarlo è quello sopra: saltare l'helper per il tag `<form>` stesso, e usare `hidden_field_tag` (che non passa dallo stesso wrapper) per i campi.

## Provarlo

```bash
git clone https://github.com/AntoninoScaffidi/nexi-xpay-with-rails.git
cd nexi-xpay-with-rails
git checkout episode-1
bundle install
bin/rails db:prepare
bin/rails db:seed
bin/rails test
```

La suite di test (11 test a questo punto) copre le due cose di cui vale più la pena fidarsi prima che sia coinvolto anche un solo euro: che `build_from_product!` usi sempre il prezzo dal database a prescindere da cosa manda il client, e che `Gateway#build_payment_params` converta l'importo in centesimi correttamente. Avvia il server, visita il prodotto seminato, e il form di checkout passa la mano alla vera pagina di pagamento sandbox di Nexi — il cliente semplicemente non può ancora completare un pagamento, perché da questo lato nulla è in ascolto della risposta.

## Cosa viene dopo

L'[episodio 2]({% post_url 2026-09-13-nexi-xpay-handling-the-outcome %}) copre i due canali che Nexi usa per comunicare cosa è successo — un redirect del browser (solo UX) e un webhook server-to-server (la vera fonte di verità) — la verifica del MAC al ritorno, e i bug veri incontrati costruendolo.
