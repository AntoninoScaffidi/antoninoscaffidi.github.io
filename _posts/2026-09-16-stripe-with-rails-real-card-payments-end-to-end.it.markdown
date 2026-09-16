---
layout: post
title: "Stripe con Rails: pagamenti reali con carta, dall'inizio alla fine"
lang: it
ref: stripe-with-rails-real-card-payments-end-to-end
permalink: /stripe-with-rails/
canonical_url: https://antoninoscaffidi.github.io/it/stripe-with-rails/
image: /assets/images/stripe-banner.png
date: 2026-09-16 20:38:00 +0200
---

È appena partita una nuova serie: **Stripe con Rails** — integrare [Stripe Checkout](https://stripe.com) in un'app Ruby on Rails, da un vero `rails new` fino a un flusso di pagamento funzionante e testato. Riusa volutamente lo stesso dominio `Product`/`Order` di [Nexi XPay con Rails]({% post_url 2026-09-13-nexi-xpay-setup-and-initiating-a-payment %}), così ciò che cambia davvero tra un gateway europeo "in casa" e uno americano moderno, developer-first, emerge nel codice stesso, non solo a parole.

![Stripe con Rails: pagamenti reali con carta, dall'inizio alla fine — una card di codice con Stripe::Checkout::Session.create, e il flusso checkout → pagina hosted → webhook verificato.](/assets/images/stripe-banner.png)

## Il modello mentale in una riga

Il "Pagamento Semplice" di Nexi è un **POST di un form firmato, inviato dal browser**: la tua app Rails costruisce un form con campi nascosti e un MAC calcolato a mano, e il *browser* lo invia direttamente ai server di Nexi. Il tuo backend non parla mai direttamente con Nexi per avviare un pagamento.

Stripe Checkout inverte questo schema. Il tuo server chiama prima Stripe:

```ruby
session = Stripe::Checkout::Session.create(
  mode: "payment",
  line_items: [{
    price_data: {
      currency: "eur",
      product_data: { name: "Corso di cucina siciliana" },
      unit_amount: 3500 # centesimi, per unità — mai il totale
    },
    quantity: 1
  }],
  client_reference_id: order.order_number,
  success_url: stripe_success_url,
  cancel_url: stripe_cancel_url
)

redirect_to session.url, allow_other_host: true
```

Stripe restituisce un URL, e *solo dopo* mandi lì il browser. Nessun MAC da calcolare a mano — la richiesta stessa è autenticata da un header `Bearer sk_test_...` che il gem ufficiale allega da solo.

## Stessa conferma a due canali, firma diversa

Entrambi i gateway risolvono lo stesso problema nello stesso modo: un redirect verso il tuo sito è solo UX e può semplicemente non avvenire mai (scheda chiusa, browser crashato), quindi non può mai essere ciò che marca un ordine come pagato. L'unica fonte di verità è un messaggio che il gateway manda **direttamente al tuo server**:

| | Nexi | Stripe |
|---|---|---|
| Canale asincrono | Notifica S2S, firmata con MAC | Evento webhook, firmato HMAC |
| Canale sincrono (solo UX) | Redirect browser + campo `esito` | Redirect browser a `success_url`/`cancel_url` |
| Il tuo riferimento ordine | `codTrans` | `client_reference_id` |

Questa è esattamente la forma dell'episodio 2: verificare la firma di un webhook Stripe con `Stripe::Webhook.construct_event`, messa a confronto diretto con la verifica MAC SHA1 scritta a mano nella serie Nexi.

## Cosa ha costruito davvero l'episodio 1

Non uno schizzo — una vera app Rails 8.1, Postgres, Tailwind, taggata git per episodio, con la propria suite Minitest:

```ruby
test "create_session sends the price in cents, not decimal euros" do
  order = orders(:pending_order)
  captured_params = nil
  fake_session = OpenStruct.new(id: "cs_test_123", url: "https://checkout.stripe.com/pay/cs_test_123")

  Stripe::Checkout::Session.stub(:create, ->(params) { captured_params = params; fake_session }) do
    StripeCheckoutService.create_session(
      order: order, success_url: "https://example.com/success", cancel_url: "https://example.com/cancel"
    )
  end

  line_item = captured_params[:line_items].first
  assert_equal 3500, line_item[:price_data][:unit_amount]
  assert_equal "eur", line_item[:price_data][:currency]
end
```

11 test, nessuna chiamata Stripe reale, e lungo il percorso due bug veri causati da gem pubblicate proprio quella sera (`json` 3.0.2 che rompe `ActiveSupport::JSON.decode`, `minitest` 6.0.6 che rimuove `Minitest::Mock` dal gem core) — più un bug di Turbo Drive genuinamente sottile: un redirect cross-origin ingoiato in silenzio da un submit di form basato su `fetch()`, diagnosticato dal vivo nel browser prima della correzione.

## Leggi l'episodio 1

[Configurazione e avvio di un pagamento]({% post_url 2026-09-16-stripe-setup-and-initiating-a-payment %}) copre tutto per intero — il meccanismo di Stripe Checkout, il modello dati, lo `StripeCheckoutService`, una spiegazione da zero di cos'è davvero un webhook e perché Checkout ne ha bisogno, ed entrambi i bug nel dettaglio.

## Cosa viene dopo

- **Episodio 2** — gestire l'esito: verificare la firma del webhook, mappare `checkout.session.completed` su `Order#mark_paid!`, testare in locale con la Stripe CLI invece di ngrok.
- **Episodio 3** — verso la produzione: una vera traccia di audit per gli eventi in arrivo, gestione di `checkout.session.expired` e `payment_intent.payment_failed`, non solo il percorso felice.

Codice companion: [github.com/AntoninoScaffidi/stripe-with-rails](https://github.com/AntoninoScaffidi/stripe-with-rails), taggato `episode-1`.
