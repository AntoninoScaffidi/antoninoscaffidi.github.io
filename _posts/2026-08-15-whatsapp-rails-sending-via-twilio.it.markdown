---
layout: post
title: "WhatsApp con Rails: inviare un messaggio vero via Twilio"
series: "whatsapp-with-rails"
episode: 2
lang: it
ref: whatsapp-rails-sending-via-twilio
permalink: /whatsapp-rails-sending-via-twilio/
canonical_url: https://antoninoscaffidi.github.io/it/whatsapp-rails-sending-via-twilio/
image: /assets/images/whatsapp-with-rails-banner.png
date: 2026-08-15 09:00:00 +0200
---

L'[episodio 1]({% post_url 2026-08-11-whatsapp-rails-setup-and-contacts %}) ci ha dato un modello `Contact` e un form per aggiungere persone. Nulla parlava ancora con Twilio. Questo episodio chiude quel vuoto dall'inizio alla fine: un form per comporre un messaggio, una vera chiamata API, e un messaggio che arriva davvero su un vero telefono via WhatsApp — cosa che ho testato per davvero scrivendo questo post, incluso il modo in cui fallisce.

Il codice è taggato [`episode-2`](https://github.com/AntoninoScaffidi/whatsapp-with-rails/tree/episode-2) nel repo [whatsapp-with-rails](https://github.com/AntoninoScaffidi/whatsapp-with-rails). Questo è un post lungo — l'obiettivo è non lasciare nulla senza spiegazione: ogni gemma, ogni riga di ogni migrazione, ogni riga del modello e del controller, e gli errori esatti in cui incapperai e perché.

## Cos'è davvero la Sandbox WhatsApp, e perché serve

Mandare un messaggio WhatsApp attraverso la vera WhatsApp Business Platform richiede un numero di telefono registrato e approvato tramite Meta — un processo con passaggi di revisione e tempi di attesa. La **Sandbox** di Twilio esiste apposta per non dover passare da lì solo per scrivere e testare codice. È un numero Twilio condiviso e già approvato (`+14155238886` per chiunque usi la sandbox di Twilio) che può mandare e ricevere messaggi WhatsApp immediatamente, con una restrizione: parlerà solo con numeri di telefono che si sono **uniti** esplicitamente.

Unirsi significa mandare un messaggio specifico — `join <codice-di-due-parole>`, es. `join vowel-purpose`, un codice che Twilio genera per account — da WhatsApp, dal numero di telefono con cui vuoi fare i test, a quel numero sandbox. Puoi farlo a mano da WhatsApp, oppure aprendo il link/QR code che la console di Twilio ti mostra (Console → Messaging → Try it out → Send a WhatsApp message). Una volta che un numero si è unito, Twilio può mandargli messaggi via API; un numero che non si è mai unito li rifiuterà, sempre, con un errore che vedremo più avanti.

Due dettagli da sapere, perché prima o poi ci sbatterai contro entrambi:

- **L'adesione dura 3 giorni di inattività**, non per sempre. Se nessuno manda nulla alla sandbox da quel numero per 3 giorni, deve rifare il `join`.
- **Questo è strettamente un meccanismo di test.** In produzione mandi da un mittente abilitato per WhatsApp che possiedi (configurato comunque tramite Twilio, ma senza la restrizione "solo numeri già uniti") — la sandbox serve per lo sviluppo, non per parlare con clienti veri.

## Le due gemme

```ruby
# Gemfile
gem "twilio-ruby"
```

```ruby
group :development do
  gem "web-console"
  gem "dotenv-rails"
end
```

`twilio-ruby` è il client Ruby ufficiale di Twilio — un wrapper attorno all'API REST di Twilio, che ti dà `Twilio::REST::Client.new(...).messages.create(...)` invece di costruire richieste HTTP a mano e fare il parsing del JSON. `dotenv-rails`, come nella [serie ai-with-ruby]({% post_url 2026-08-07-wiring-rubyllm-into-rails %}), carica un file `.env` dentro `ENV` in development, così le credenziali vivono fuori dal codice.

```bash
bundle install
```

## Credenziali: `.env`, `.env.example` e `.gitignore`

Servono tre valori reali, tutti segreti: il tuo Account SID Twilio, il tuo Auth Token, e il numero sandbox WhatsApp. Il `.gitignore` di default di Rails 8 esclude già `.env*`, quindi un vero file `.env` non viene mai committato. Committiamo invece un modello:

```
# .env.example
TWILIO_ACCOUNT_SID=ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
TWILIO_AUTH_TOKEN=your-auth-token-here
TWILIO_WHATSAPP_NUMBER=+14155238886
```

Siccome il pattern `/.env*` del `.gitignore` è abbastanza ampio da intercettare anche `.env.example`, serve un'eccezione esplicita — esattamente la stessa trappola già incontrata nella serie ai-with-ruby:

```
# .gitignore
/.env*
!/.env.example
```

Per far girare davvero l'app, copia il modello e inserisci i valori reali:

```bash
cp .env.example .env
```

**Sull'Account SID e l'Auth Token in particolare**: sono le credenziali master del tuo account Twilio — chiunque le abbia può mandare messaggi (e farteli addebitare) a nome tuo. Si trovano sulla dashboard principale della Twilio Console appena fai login. Se mai ne incolli una per sbaglio da qualche parte pubblica, Twilio ti permette di rigenerare l'Auth Token dalla console; quello vecchio smette di funzionare immediatamente.

## Il client Twilio: un initializer, una riga

```ruby
# config/initializers/twilio.rb
Rails.application.config.x.twilio_client = Twilio::REST::Client.new(
  ENV.fetch("TWILIO_ACCOUNT_SID"),
  ENV.fetch("TWILIO_AUTH_TOKEN")
)
```

Stessa logica dell'initializer RubyLLM nella serie AI with Ruby: i file dentro `config/initializers/` vengono eseguiti una sola volta, all'avvio, prima di qualsiasi richiesta — il posto giusto per costruire qualcosa che parla con un'API di terze parti e passargli le tue credenziali.

Due cose su cui vale la pena essere precisi. Primo, `ENV.fetch("TWILIO_ACCOUNT_SID")` — nessun default dato come secondo argomento — **solleva un errore** se la variabile manca, invece di proseguire silenziosamente con `nil`. È voluto: un client Twilio mal configurato che non fa nulla in silenzio è molto più difficile da debuggare di un'app che si rifiuta di avviarsi con un chiaro `KeyError`. Secondo, `Rails.application.config.x` è il namespace integrato di Rails per la configurazione personalizzata dell'applicazione — la `x` sta per "custom", ed esiste apposta perché non si sia tentati di infilare configurazione specifica dell'app in costanti globali o direttamente in `Rails.application.config` (che Rails stesso usa). Ovunque nell'app, `Rails.application.config.x.twilio_client` restituisce lo stesso client configurato.

## Il modello Message, e la migrazione dietro di esso

```bash
bin/rails generate model Message contact:references body:text twilio_sid:string status:string
```

Il generator ha prodotto una migrazione che poi ho modificato prima di eseguirla — vale la pena guardare entrambe le versioni, perché la modifica è dove sta il ragionamento vero.

```ruby
# db/migrate/..._create_messages.rb — come generata
create_table :messages do |t|
  t.references :contact, null: false, foreign_key: true
  t.text :body
  t.string :twilio_sid
  t.string :status

  t.timestamps
end
```

```ruby
# db/migrate/..._create_messages.rb — come eseguita
create_table :messages do |t|
  t.references :contact, null: false, foreign_key: true
  t.text :body, null: false
  t.string :twilio_sid
  t.string :status, null: false, default: "queued"

  t.timestamps
end
```

Colonna per colonna:

- **`t.references :contact, null: false, foreign_key: true`** — è ciò che l'argomento `contact:references` passato al generator produce da solo: una colonna intera `contact_id`, un indice a livello di database su di essa, un vincolo di chiave esterna a livello di database (`foreign_key: true`) così il database stesso rifiuta di far puntare un `Message` a un `Contact` che non esiste, e `null: false` perché un messaggio senza contatto non ha senso.
- **`t.text :body, null: false`** — `text` invece di `string` perché il corpo di un messaggio non ha un limite di lunghezza naturale e breve come un nome; ho aggiunto `null: false` a mano perché il generator non aggiunge da solo vincoli di presenza, e un messaggio vuoto non è mai qualcosa che vogliamo inviare.
- **`t.string :twilio_sid`** — volutamente nullable. Viene valorizzato *dopo* che Twilio accetta il messaggio (approfondito sotto); prima di quel momento, esiste già una riga `Message` reale senza ancora un SID.
- **`t.string :status, null: false, default: "queued"`** — l'unica modifica sostanziale. Ho aggiunto `null: false, default: "queued"` così ogni `Message` ha uno status sensato nell'istante in cui viene creato, prima ancora di aver parlato con Twilio. Perché questa colonna esista affatto è la domanda più interessante, coperta nella prossima sezione.

```ruby
# app/models/message.rb
class Message < ApplicationRecord
  belongs_to :contact

  validates :body, presence: true

  def deliver!
    twilio_message = Rails.application.config.x.twilio_client.messages.create(
      from: "whatsapp:#{ENV.fetch('TWILIO_WHATSAPP_NUMBER')}",
      to: "whatsapp:#{contact.whatsapp_number}",
      body: body
    )

    update!(twilio_sid: twilio_message.sid, status: twilio_message.status)
  end
end
```

`belongs_to :contact` da solo basta a richiedere un contatto — Rails 5+ rende le associazioni `belongs_to` obbligatorie di default, quindi questa sola riga ci dà già gran parte di quello che `null: false` nella migrazione faceva a livello di database, ma a livello applicativo.

`validates :body, presence: true` rispecchia il `null: false` su `body` — uno è il database che rifiuta dati non validi indipendentemente da cosa li inserisce, l'altro è un errore di validazione amichevole prima ancora di provare, ed è ciò che permette a `MessagesController` di mostrare "can't be blank" nel form invece di un crash a livello di database.

`deliver!` è l'unico metodo che fa davvero qualcosa. Guarda bene `from:` e `to:` — i numeri WhatsApp nell'API di Twilio hanno sempre il prefisso letterale `whatsapp:`, es. `whatsapp:+14155238886`. È così che l'API di messaggistica unificata di Twilio distingue tra mandare lo stesso numero come messaggio WhatsApp o come semplice SMS — il prefisso, non un endpoint separato. Dimenticarlo su uno dei due lati fa fallire la chiamata o, silenziosamente, prova a mandare un normale SMS invece.

La convenzione di naming — `deliver!`, con il punto esclamativo — rispecchia la convenzione di Rails stesso per i metodi che fanno qualcosa con effetti collaterali reali e potenzialmente fallibili (`save!`, `create!`), a differenza di una semplice query. Mandare un messaggio WhatsApp è un effetto collaterale reale quanto un metodo può averne.

## Perché esistono `twilio_sid` e `status`: l'invio WhatsApp è asincrono

Vale la pena essere espliciti su questo, perché è facile dare per scontato che una chiamata API riuscita significhi che il messaggio è arrivato — non è così. Quando `messages.create` ritorna senza sollevare errori, significa solo: *Twilio ha accettato la richiesta e l'ha messa in coda per la consegna.* Lo `status` che Twilio restituisce a quel punto è tipicamente `"queued"`, non `"delivered"`. Cosa succede davvero al messaggio dopo — inviato, consegnato, letto, o fallito — avviene in modo asincrono, e per default la tua app Rails non ne sa più nulla a meno che non lo chieda.

Ecco tutto il motivo per cui `twilio_sid` e `status` sono colonne su `Message` invece di essere buttate via dopo la chiamata API: `twilio_sid` è l'identificatore che Twilio restituisce (`SM...`), ed è come recuperare quel messaggio più tardi per controllare cosa gli è successo davvero. È esattamente ciò che ho fatto testando questo episodio:

```ruby
fresh = Rails.application.config.x.twilio_client.messages(message.twilio_sid).fetch
fresh.status        # => "failed" — aggiornato in seguito, non dalla risposta originale
fresh.error_code     # => 63015
```

Il modo *giusto* per tenere `status` aggiornato senza fare polling a mano è un **webhook di status callback** — un URL che dai a Twilio, a cui fa una POST ogni volta che lo status di un messaggio cambia. È genuinamente più macchina (un endpoint pubblico, una rotta, la verifica della richiesta) di quanto stia in un episodio sul primo invio riuscito — quindi resta fuori scope qui, ma vale la pena sapere che `status` esiste su questo modello proprio perché quel futuro webhook avrà qualcosa da aggiornare.

## Collegarlo all'app: rotte, controller, elenco contatti

```ruby
# config/routes.rb
resources :contacts, only: [:index, :new, :create] do
  resources :messages, only: [:new, :create]
end
```

Le risorse annidate qui non sono decorazione — `messages` genuinamente non ha senso senza un `contact` di contesto; non esiste una schermata "componi un messaggio" che non riguardi già una persona specifica. Questo genera path come `new_contact_message_path(contact)` e `contact_messages_path(contact)`, entrambi con l'id del contatto.

```ruby
# app/controllers/messages_controller.rb
class MessagesController < ApplicationController
  before_action :set_contact

  def new
    @message = @contact.messages.new
  end

  def create
    @message = @contact.messages.new(message_params)

    if @message.save
      begin
        @message.deliver!
        redirect_to contacts_path, notice: "Message sent to #{@contact.name}."
      rescue Twilio::REST::RestError => e
        redirect_to contacts_path, alert: "Twilio couldn't send the message: #{e.error_message}"
      end
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  def set_contact
    @contact = Contact.find(params[:contact_id])
  end

  def message_params
    params.require(:message).permit(:body)
  end
end
```

`before_action :set_contact` gira prima di entrambe le azioni, caricando `@contact` da `params[:contact_id]` — il parametro della rotta annidata, non `:id` (che sarebbe l'id del messaggio stesso, che non esiste ancora per `new`/`create`).

`create` fa tre cose in ordine: costruisce il `Message` (non ancora salvato), lo salva nel database, *poi* tenta la consegna. Questo ordine conta — il messaggio esiste come riga di database, con `status: "queued"` dal default della colonna, ancora prima di sapere se Twilio lo accetterà. Se il processo si interrompesse tra il salvataggio e l'invio, resterebbe comunque una traccia di un messaggio previsto, non il silenzio.

## Il caso di fallimento sincrono: `Twilio::REST::RestError`

Ecco una cosa che ho scoperto solo rompendola davvero durante i test: `deliver!` può fallire in due modi completamente diversi, e solo uno dei due è visibile dove te lo aspetteresti.

Il **fallimento asincrono** — il numero non si è mai unito alla sandbox, il messaggio viene rifiutato più a valle — non solleva nulla in Rails. `messages.create` ritorna con successo con `status: "queued"`; il fallimento emerge solo più tardi se torni a controllare, esattamente come descritto sopra. Mi è capitato direttamente: mandare a un contatto il cui numero non aveva mai inviato `join <codice>` alla sandbox è tornato come una normale risposta `queued` che non solleva errori dal punto di vista dell'app, e solo dopo, recuperando il messaggio da Twilio, si è rivelato `status: "failed"`, `error_code: 63015`. **L'errore 63015, in particolare, significa "questo destinatario non si è unito a questa sandbox"** — la cosa più comune in assoluto in cui incapperai testando questa integrazione, e vale la pena riconoscerla a colpo d'occhio.

Il **fallimento sincrono** è diverso: credenziali sbagliate, una richiesta malformata, qualsiasi cosa che l'API di Twilio rifiuti immediatamente. Quello *solleva* davvero un errore, come `Twilio::REST::RestError` — l'ho confermato direttamente, costruendo deliberatamente un client con Account SID e Auth Token sbagliati e osservandolo sollevare l'errore sulla chiamata API:

```ruby
bad_client = Twilio::REST::Client.new("AC0000000000000000000000000000000", "wrongtoken")
bad_client.messages.create(from: "whatsapp:+14155238886", to: "whatsapp:+391234567890", body: "test")
# solleva Twilio::REST::RestError
# e.status_code    => 401
# e.error_message  => "Authentication Error - invalid username"
```

Prima di aggiungere il `rescue` che vedi nel controller sopra, questa classe di errore sarebbe risalita direttamente attraverso `deliver!`, attraverso `create`, in un `500` non gestito, in quella che dovrebbe essere una demo da mostrare a un cliente. `rescue Twilio::REST::RestError => e` lo cattura e reindirizza con un `alert` leggibile invece — `e.error_message` è la stringa comprensibile che Twilio stesso restituisce, quindi il messaggio mostrato è la spiegazione di Twilio, non una nostra supposizione.

Quello che questo episodio *non* fa è trasformare il caso asincrono del 63015 in un bel messaggio in UI — non è possibile dall'interno di `create` in alcun modo, dato che l'app ha già ricevuto una risposta "riuscita", `queued`, nel momento in cui avviene il fallimento vero. Gestirlo correttamente richiede il webhook di status callback menzionato sopra; lo segnalo qui perché sia chiaro che è un vuoto noto e voluto, non una svista.

## Provarlo

```bash
bin/dev
```

Aggiungi un contatto con un numero WhatsApp che si è davvero unito alla tua sandbox (vedi le istruzioni di join più sopra in questo post — un numero che non si è unito accetterà l'invio dal punto di vista dell'app e poi fallirà in modo invisibile, esattamente come descritto sopra). Clicca "Message" accanto a lui, scrivi qualcosa, invia.

Ecco il risultato reale dai test fatti per questo post — un messaggio vero, mandato da questo stesso codice, ricevuto su un telefono vero:

> **Twilio:** Test from whatsapp-with-rails episode 2 🎉

Se provi a mandare a un contatto il cui numero non si è mai unito alla sandbox, otterrai `queued` nell'app e nulla sul telefono — è l'errore 63015 che aspetta di essere scoperto se controlli lo status del messaggio in seguito, non un bug in questo codice.

## Cosa viene dopo

L'episodio 3 fa lo stesso lavoro in un modo diverso: chiamando direttamente l'API Meta WhatsApp Cloud, senza Twilio in mezzo, così il compromesso tra i due approcci — la semplicità di Twilio contro un servizio in meno nel mezzo — si vede nel codice vero, non solo in astratto.
