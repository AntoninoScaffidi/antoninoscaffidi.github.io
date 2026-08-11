---
layout: post
title: "WhatsApp con Rails: setup del progetto e contatti"
series: "whatsapp-with-rails"
episode: 1
lang: it
ref: whatsapp-rails-setup-and-contacts
permalink: /whatsapp-rails-setup-and-contacts/
canonical_url: https://antoninoscaffidi.github.io/it/whatsapp-rails-setup-and-contacts/
image: /assets/images/whatsapp-with-rails-banner.png
date: 2026-08-11 09:00:00 +0200
---

Questo è il primo episodio di **WhatsApp with Rails**, una serie breve nata da un lavoro reale per un cliente: doveva integrare la messaggistica WhatsApp in un'app Rails, e prima di costruire il progetto completo volevo una piccola demo funzionante da mostrargli — e assicurarmi di aver capito davvero l'integrazione, non solo di aver seguito un tutorial.

Lo scope qui è volutamente ristretto. Niente CRM, niente campagne, niente template messaggi, niente statistiche — quello è il progetto vero, e non è di cosa parla questa serie. Questa serie parla di una cosa sola: far partire un vero messaggio WhatsApp da un'app Rails, capendo cosa succede davvero quando lo fai. Lo costruiremo in due modi — passando da Twilio (questo episodio e il prossimo), e poi, come episodio a parte, chiamando direttamente le API Meta WhatsApp Cloud, senza Twilio in mezzo, così i due approcci si possono confrontare.

Il codice è su GitHub, taggato [`episode-1`](https://github.com/AntoninoScaffidi/whatsapp-with-rails/tree/episode-1), nel repo [whatsapp-with-rails](https://github.com/AntoninoScaffidi/whatsapp-with-rails).

## Cosa costruiamo, e perché prima Twilio

Prima di scrivere codice, vale la pena capire cos'è davvero Twilio qui, perché "API WhatsApp" è una frase un po' ambigua — ci sono due modi per arrivarci.

**Meta possiede WhatsApp**, e Meta offre davvero un'API diretta (la WhatsApp Business Cloud API) per inviare e ricevere messaggi in modo programmatico. Puoi integrarti direttamente con quella. Ma l'API di Meta richiede un account Meta Business, una review dell'app, la registrazione del numero attraverso il sistema di Meta stesso, e la sua autenticazione e configurazione dei webhook sono specifiche di Meta.

**Twilio si appoggia sopra a tutto questo.** È una piattaforma di comunicazione che ha già fatto l'integrazione con Meta, avvolta in un'API REST più semplice e ben documentata (con tanto di gemma Ruby) che ha lo stesso aspetto sia che tu stia mandando WhatsApp, SMS o una chiamata vocale. Ti serve comunque una registrazione del mittente approvata per WhatsApp in entrambi i casi, ma la modalità sandbox di Twilio ti fa mandare messaggi di test in pochi minuti, senza aspettare il processo di approvazione di Meta.

Per questo l'episodio 1 e 2 usano Twilio: è la strada più veloce per far arrivare davvero un messaggio su un telefono, ed è una scelta legittima anche in produzione, non solo una scorciatoia — parecchi prodotti reali fanno passare la loro messaggistica WhatsApp da Twilio in modo permanente. L'episodio 3 poi fa lo stesso lavoro passando direttamente dall'API di Meta, così il compromesso (semplicità e velocità contro un servizio in meno nel mezzo) si vede nel codice vero, non solo in astratto.

## Creare l'app

```bash
rails new whatsapp-with-rails -d postgresql --css tailwind
```

Stessa logica delle altre serie su questo blog: PostgreSQL perché è un default ragionevole per qualcosa che potrebbe crescere, Tailwind per mantenere le view leggibili senza un foglio di stile separato. Qui non c'è ancora nulla di specifico di Twilio — questo comando è identico a come inizieresti qualsiasi piccola app Rails.

```bash
bin/rails db:create
```

## Il modello Contact

Il progetto vero ha un CRM completo — segmenti, tag, tracciamento del consenso GDPR, import/export. Qui ci servono esattamente due campi: chi, e quale numero contattare.

```bash
bin/rails generate model Contact name:string whatsapp_number:string
```

Due cose da fare prima di migrare: rendere entrambi i campi obbligatori, e validare per bene il formato del numero — perché la messaggistica WhatsApp **richiede** un formato specifico, ed è molto meglio intercettare un numero malformato in una validazione del form che in una chiamata API fallita più tardi.

```ruby
# db/migrate/..._create_contacts.rb
create_table :contacts do |t|
  t.string :name, null: false
  t.string :whatsapp_number, null: false

  t.timestamps
end
```

```ruby
# app/models/contact.rb
class Contact < ApplicationRecord
  validates :name, presence: true
  validates :whatsapp_number, presence: true, format: {
    with: /\A\+[1-9]\d{6,14}\z/,
    message: "must be in E.164 format, e.g. +391234567890"
  }
end
```

### Perché proprio E.164

**E.164** è lo standard internazionale dell'ITU per la formattazione dei numeri di telefono — il formato che garantisce che un numero sia univoco in tutto il mondo. La forma è `+` seguito dal prefisso internazionale del paese, seguito dal numero dell'abbonato, senza spazi, trattini, parentesi, senza lo zero iniziale sulla parte locale dove il piano di numerazione del paese normalmente ne prevederebbe uno.

Un esempio concreto: un cellulare italiano che normalmente scriveresti come `333 1234567` diventa in E.164 `+393331234567` — prefisso `39`, poi il numero così com'è (i cellulari italiani non portano uno zero di trunk iniziale già di loro).

Perché conta qui in particolare: l'API WhatsApp di Twilio — e il protocollo WhatsApp stesso alla base — richiede E.164. Senza un unico formato univoco, `333-1234567` non ha senso fuori contesto: manca il prefisso del paese? Va tolto uno zero iniziale? E.164 elimina ognuno di questi dubbi.

La regex rispecchia direttamente lo standard: `+`, poi una cifra da `1` a `9` (i prefissi paese non iniziano mai con `0`), poi altre 6-14 cifre — che corrisponde esattamente al limite reale di E.164 di 15 cifre totali dopo il `+`.

```bash
bin/rails db:migrate
```

## Rotte, controller, elenco contatti

Solo il minimo REST per elencare i contatti e aggiungerne uno — niente modifica, niente cancellazione, non servono per questa demo:

```ruby
# config/routes.rb
resources :contacts, only: [:index, :new, :create]
root "contacts#index"
```

```ruby
# app/controllers/contacts_controller.rb
class ContactsController < ApplicationController
  def index
    @contacts = Contact.order(:name)
  end

  def new
    @contact = Contact.new
  end

  def create
    @contact = Contact.new(contact_params)

    if @contact.save
      redirect_to contacts_path, notice: "Contact added."
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  def contact_params
    params.require(:contact).permit(:name, :whatsapp_number)
  end
end
```

Due cose da segnalare, entrambe lezioni riprese dalle altre serie di questo blog: `create` **reindirizza** dopo un salvataggio riuscito (il pattern Post/Redirect/Get — vedi l'[episodio 3 di AI with Ruby]({% post_url 2026-08-09-persisting-conversations-with-activerecord %}) per capire perché conta in particolare per Turbo Drive), e un salvataggio fallito **renderizza con `status: :unprocessable_entity`** invece del `200 OK` di default — lo status HTTP corretto per "la richiesta è stata capita ma i dati non erano validi", e qualcosa che Turbo stesso controlla per decidere se trattare una risposta di form come un errore.

Le view sono un elenco semplice e un form semplice — niente di nuovo qui, quindi non le ripeto per intero; sono nel repo se vuoi vedere il markup Tailwind.

## Provarlo

```bash
bin/dev
```

Apri `http://127.0.0.1:3000`: un elenco contatti vuoto, un pulsante "Add contact", un form. Prova a inviare un numero senza il `+` — la validazione lo intercetta, con il messaggio d'errore esatto che spiega cosa ci si aspetta.

Qui nulla parla ancora con Twilio. È voluto — l'episodio 2 collega la gemma `twilio-ruby` e manda davvero un messaggio a uno di questi contatti.

## Cosa viene dopo

L'episodio 2 aggiunge la gemma `twilio-ruby`, un form per comporre un messaggio, e la chiamata API che lo invia — più cos'è davvero una sandbox WhatsApp di Twilio e perché ti serve prima che Meta approvi il tuo mittente.
