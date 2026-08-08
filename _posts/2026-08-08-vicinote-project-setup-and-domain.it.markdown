---
layout: post
title: "VicinoTe: preparare un marketplace Rails e progettarne il dominio"
series: "vicinote"
episode: 1
lang: it
ref: vicinote-project-setup-and-domain
permalink: /vicinote-project-setup-and-domain/
canonical_url: https://antoninoscaffidi.github.io/it/vicinote-project-setup-and-domain/
date: 2026-08-08 10:00:00 +0200
image: /assets/images/vicinote-banner.png
---

Questo è il primo episodio di **VicinoTe**, una serie in cui costruiamo un'applicazione Rails completa partendo da una cartella vuota, fino ad arrivare — negli episodi finali — a un modulo di AI basato su RubyLLM.

Il nome viene da *vicino a te*. VicinoTe è un marketplace di servizi locali: trovi qualcuno nelle vicinanze che ti ripara un rubinetto, ti insegna la chitarra, porta a spasso il cane o ti ristruttura il bagno — oppure sei tu a offrire le tue competenze a chi ti sta intorno.

In questo episodio prepariamo il progetto e, cosa più importante, decidiamo cosa stiamo effettivamente costruendo. Il codice è su GitHub, taggato [`episode-1`](https://github.com/AntoninoScaffidi/vicinote-tutorial/tree/episode-1) nel repo [vicinote-tutorial](https://github.com/AntoninoScaffidi/vicinote-tutorial), che crescerà a ogni post.

## Perché un marketplace

I tutorial costruiscono spesso un blog o una lista di cose da fare. Vanno bene per imparare la sintassi, ma sono troppo semplici per incontrare i problemi che rendono Rails interessante: lo stesso record che assume ruoli diversi a seconda del contesto, il denaro, le disponibilità, i permessi, una ricerca che deve funzionare davvero.

Un marketplace di servizi li tocca tutti, e lo fa in modo graduale: puoi avere qualcosa di funzionante dopo due episodi e avere ancora parecchio da costruire dopo dieci.

## Creare l'applicazione

```bash
rails new vicinote-tutorial -d postgresql --css tailwind
```

In quella riga ci sono tre decisioni, quindi rendiamole esplicite.

**`-d postgresql`.** Rails 8 usa SQLite di default, che oggi è davvero una buona scelta. Opto comunque per PostgreSQL per una ragione precisa: gli episodi sull'AI più avanti in questa serie hanno bisogno della **ricerca vettoriale** per la ricerca semantica sui servizi. Il modo standard di farlo in Postgres è l'estensione `pgvector`, e potervi accedere senza dover migrare il database a metà serie vale il piccolo lavoro in più adesso.

**`--css tailwind`.** Preferenza personale, e mantiene le view leggibili senza dover gestire un foglio di stile separato accanto al tutorial. Nulla nella serie dipende specificamente da Tailwind: se preferisci il CSS semplice, il markup resterà comunque comprensibile.

**I default di Rails 8 che teniamo.** Il generatore porta con sé anche Propshaft, Importmap, Turbo, Stimulus e il trio Solid (Solid Queue, Solid Cache, Solid Cable). Non ne configuriamo ancora nessuno, ma sono il motivo per cui in questo progetto non c'è Redis né un passaggio di build con Node.

Poi creiamo i database:

```bash
bin/rails db:create
```

Se il comando fallisce, PostgreSQL non è in esecuzione o non è raggiungibile: è l'unico pezzo di configurazione che devi sistemare sulla tua macchina prima di proseguire.

## Progettare il dominio

Questa è la parte su cui vale la pena rallentare. Impostare bene il modello adesso evita parecchie migrazioni dolorose più avanti.

Ecco la forma a cui puntiamo:

```
User            può offrire servizi E prenotarli
 ├─ Service     qualcosa che un utente offre (titolo, descrizione, prezzo)
 │   └─ Category
 └─ Booking     un utente prenota il servizio di un altro utente
     └─ Review  lasciata a prenotazione completata
```

### La decisione importante: un solo User, due ruoli

La domanda centrale in qualsiasi marketplace è come rappresentare i due lati della transazione. Ci sono tre risposte comuni.

**Opzione A — modelli separati.** Un modello `Provider` e un modello `Customer`, ciascuno con la propria tabella.

È la prima idea che viene in mente a quasi tutti, ed è quasi sempre sbagliata. Duplichi subito tutto ciò che non è specifico del ruolo: email, password, nome, avatar, telefono, indirizzo. Poi ti serve un'autenticazione che funzioni per entrambi, il che significa o due flussi di login o un groviglio polimorfico. E nel momento in cui una persona che offre lezioni di chitarra vuole prenotare un idraulico, le servono due account con due password — cosa assurda, ed esattamente la situazione in cui un marketplace di quartiere si imbatte di continuo.

**Opzione B — una colonna `role` su User.** Un'unica tabella `users` con `role: "provider"` oppure `role: "customer"`.

Meglio, ma codifica un presupposto falso: che essere fornitore o cliente sia una proprietà permanente di una persona. Non lo è. È una proprietà *di una specifica relazione*. La stessa persona è fornitore nella prenotazione in cui insegna chitarra e cliente in quella in cui chiama l'idraulico. Una singola colonna `role` non riesce a esprimerlo, e ti ritroverai a combatterci contro.

**Opzione C — il ruolo emerge dall'associazione.** Un solo modello `User`, nessuna colonna ruolo. Sei fornitore *dei servizi che hai creato* e cliente *delle prenotazioni che hai effettuato*.

È quella che useremo:

```ruby
class User < ApplicationRecord
  has_many :services                                        # ciò che offro
  has_many :bookings, foreign_key: :customer_id             # ciò che ho prenotato
  has_many :received_bookings, through: :services, source: :bookings
end
```

Nessuna duplicazione, un solo login, e un utente che fa entrambe le cose diventa il caso normale invece che l'eccezione. Quando più avanti ci servirà un comportamento riservato a chi offre servizi, sarà una domanda sui dati (`user.services.any?`), non su un flag da tenere sincronizzato.

Per onestà sul compromesso: questo approccio rende alcune query leggermente più articolate. "Mostrami tutto ciò che succede nel mio account" deve guardare due associazioni invece di una. In cambio, non dovremo mai rispondere alla domanda "cosa succede quando un cliente diventa fornitore" — perché non succede nulla. È un buon affare.

### Le prenotazioni contengono il prezzo

Una `Booking` non è solo un collegamento tra un utente e un servizio. È il registro di un accordo in un preciso momento: la data, il prezzo concordato, lo stato. Ha bisogno di una propria colonna prezzo invece di leggere `service.price`, perché il prezzo del servizio può cambiare domani e questo non deve riscrivere silenziosamente ciò che qualcuno aveva accettato di pagare la settimana scorsa.

È il genere di cosa facile da sbagliare e dolorosa da correggere quando ci sono dati reali, ed è il motivo per cui la decidiamo prima di scrivere una sola migrazione.

### Le recensioni appartengono alle prenotazioni, non ai servizi

Verrebbe naturale agganciare una `Review` direttamente a un `Service`. Agganciarla invece a una `Booking` ci dà gratis una cosa preziosa: solo chi ha effettivamente prenotato e completato un servizio può recensirlo. Il vincolo è incorporato nella forma dei dati, invece di essere affidato a una logica di validazione che dobbiamo ricordarci di scrivere.

## Una pagina di presentazione

Per chiudere con qualcosa di visibile, una home page statica:

```bash
bin/rails generate controller Pages home --skip-routes --no-helper --no-assets
```

```ruby
# config/routes.rb
root "pages#home"
```

Ho usato `--skip-routes` perché altrimenti il generatore aggiungerebbe `get "pages/home"`, mentre noi la vogliamo sulla radice. `--no-helper` e `--no-assets` servono solo a evitare la creazione di file che non useremo.

La view è semplice markup che descrive il progetto, con due pulsanti volutamente inerti: segnaposto per i flussi che costruiremo dopo.

```bash
bin/dev
```

Apri `http://127.0.0.1:3000` ed eccola: un'applicazione vuota che sa cosa vuole diventare.

## Cosa viene dopo

L'episodio 2 trasforma lo schizzo del dominio in codice vero: il modello `User`, l'autenticazione e le prime migrazioni. Da lì il marketplace inizia a prendere forma — servizi, categorie e i flussi che collegano i due lati di una prenotazione.
