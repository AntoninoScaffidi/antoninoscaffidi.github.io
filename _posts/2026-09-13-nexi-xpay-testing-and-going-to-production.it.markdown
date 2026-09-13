---
layout: post
title: "Nexi XPay con Rails: test, verifica locale e produzione"
series: "nexi-xpay-with-rails"
episode: 3
lang: it
ref: nexi-xpay-testing-and-going-to-production
permalink: /nexi-xpay-testing-and-going-to-production/
canonical_url: https://antoninoscaffidi.github.io/it/nexi-xpay-testing-and-going-to-production/
image: /assets/images/nexi-xpay-ep3-banner.png
date: 2026-09-13 08:00:00 +0200
---

L'[episodio 1]({% post_url 2026-09-13-nexi-xpay-setup-and-initiating-a-payment %}) ha costruito il lato della richiesta, l'[episodio 2]({% post_url 2026-09-13-nexi-xpay-handling-the-outcome %}) il lato della risposta. 19 test automatici passano, e nessuno di essi tocca davvero la rete — che è esattamente il punto di una suite di test, ma significa anche che nessuno di essi dimostra davvero che i server reali di Nexi accettino ciò che questa app gli manda. Questo episodio chiude quel divario: un pagamento vero, contro la vera sandbox, con il webhook che arriva da solo.

Il codice è taggato [`episode-3`](https://github.com/AntoninoScaffidi/nexi-xpay-with-rails/tree/episode-3) nel repo [nexi-xpay-with-rails](https://github.com/AntoninoScaffidi/nexi-xpay-with-rails) — questo episodio è più leggero sul nuovo codice applicativo e più pesante sul processo, dato che testare e verificare sono l'argomento vero e proprio.

## Perché la sola suite di test non basta

Ogni test scritto nei due episodi precedenti costruisce il proprio MAC usando la `mac_key` dell'app stessa e verifica che l'app sappia validare la propria firma — un controllo reale e significativo, ma un circuito chiuso. Dimostra che l'*algoritmo* è implementato correttamente; non dice nulla su se i server di Nexi calcolino per caso lo stesso identico algoritmo allo stesso modo per la *tua* specifica configurazione di terminale, se il tuo alias sandbox funzioni davvero, o se una notifica possa davvero raggiungere il tuo server dall'esterno. Queste cose si confermano solo con un'esecuzione end-to-end contro la cosa vera.

## Nexi non può raggiungere `localhost`

Questo è il blocco pratico: `/nexi/notify` deve essere raggiungibile dai server di Nexi stessi, e `localhost:3000` non è raggiungibile da nessun posto tranne la tua macchina. La soluzione è un tunnel pubblico:

```bash
brew install --cask ngrok
ngrok config add-authtoken <token>   # account gratuito, da ngrok.com
bin/dev                               # oppure: bin/rails server -p 3000
ngrok http 3000
```

`ngrok` stampa un URL `https://qualcosa.ngrok-free.app`. Due cose devono essere vere perché il resto funzioni:

**Devi navigare davvero da quell'URL, non da `localhost`.** `OrdersController#create` costruisce `notify_url` a partire dall'host da cui è arrivata la richiesta corrente — visita da `localhost` e Nexi si ritrova con un `notify_url` che non potrà mai raggiungere, per quanto correttamente sia costruito tutto il resto.

**Rails deve essere istruito a fidarsi dell'host ngrok**, o `ActionDispatch::HostAuthorization` rifiuta la richiesta prima ancora che il tuo controller la veda:

```ruby
# config/environments/development.rb
config.hosts << /.*\.ngrok-free\.(app|dev)/
```

Senza questa riga, visitare l'URL ngrok mostra la pagina di errore "Blocked hosts" di Rails — un gotcha reale, non ipotetico, e facile da cui perdere dieci minuti se non sai già che `config.hosts` è il meccanismo.

## Una vera carta di test, e un vero pagamento

Il backoffice sandbox di Nexi (la stessa pagina **Area test** a cui rimandava l'episodio 1 per le credenziali) fornisce numeri di carta di test:

- `4539 9700 0000 0006` — Visa, approvata
- `4539 9700 0000 0014` — Visa, rifiutata (utile per esercitare volutamente il ramo `KO` di `#notify`)

Qualunque data di scadenza futura, qualunque CVV a 3 cifre. L'autenticazione 3D Secure in sandbox accetta sempre l'OTP `123456`.

Eseguire l'intero flusso — checkout, redirect verso Nexi, inserimento carta, 3D Secure — e poi controllare cosa è davvero arrivato conferma che tutto funziona insieme, non solo ogni pezzo isolato. Ecco una notifica vera ricevuta su `/nexi/notify` durante proprio questo test, oscurata solo dove davvero non conta:

```
Started POST "/nexi/notify" for 185.198.117.20
Processing by NexiCallbacksController#notify as HTML
Parameters: {
  "codTrans"=>"19JWR3YNVOKC",
  "esito"=>"OK",
  "importo"=>"1200",               # 12,00 €
  "divisa"=>"EUR",
  "data"=>"20260912", "orario"=>"225428",
  "codAut"=>"MP7EL4",              # codice di autorizzazione reale
  "pan"=>"453997******0006",       # mascherato, mai in chiaro
  "brand"=>"VISA",
  "tipoTransazione"=>"VBV_FULL",   # 3D Secure completo
  "mac"=>"9fee38d052f87f5ac6548ce0a9ba20c2d9933e8e",
}
```

Nessun passaggio manuale in nessun punto di quel log — i server di Nexi hanno trovato da soli l'URL ngrok, ci hanno fatto un POST, e l'app ha segnato l'ordine come pagato interamente da sola. Questo è l'asticella vera per dire "funziona", ed è significativamente più alta di "i test sono verdi".

Una cosa che vale la pena sapere se vai a guardare il tuo log: **Nexi può mandare più campi di quanti ne elenchi qualunque tabella nella sua documentazione** — un campo configurato dall'esercente come `num_contratto`, o campi specifici del terminale come `check`/`aliasEffettivo`, a seconda di cosa è stato impostato nel tuo specifico profilo di backoffice. È esattamente per questo che `raw_notification_params` cattura l'intero payload senza distinzioni invece di scegliere un elenco fisso di chiavi attese — non c'è modo di sapere in anticipo l'insieme completo che un dato terminale manderà davvero.

## Dati di seed, per chi vuole provarlo in prima persona

```ruby
# db/seeds.rb
Product.find_or_create_by!(name: "Sample experience") do |product|
  product.price = 12.00
  product.purchasable = true
end
```

```bash
bin/rails db:seed
```

Prezzo basso di proposito — l'ambiente sandbox ha dei limiti, e non c'è motivo di testarli da vicino.

## Cosa manca davvero prima della produzione

Questa serie costruisce un flusso di pagamento completo, *testato*, *verificato* — volutamente non uno pronto per la produzione. Nello specifico, resta ancora aperto:

- **Credenziali `nexi.production`** — un alias di terminale e una MAC key reali, aggiunti a `credentials.yml.enc` accanto a `nexi.sandbox` una volta pronti ad accettare denaro vero. L'endpoint di produzione perde il prefisso `int-`: `https://ecommerce.nexi.it/...`.
- **Un'email di conferma**, inviata da `#notify` dopo `mark_paid!` — mai da `#return`, dato che quell'azione è solo UX e il cliente può saltarla del tutto.
- **Una vista admin su ordini e notifiche** — perlopiù in sola lettura, con un'azione esplicita `mark_refunded` se servono rimborsi, mai un `update` generico che potrebbe toccare campi finanziari.
- **Rivalidare esplicitamente `importo`/`divisa` della notifica contro `order.total_amount`/`order.currency`** prima di chiamare `mark_paid!`. Il MAC già garantisce che Nexi non abbia mandato qualcosa che una notifica manomessa non riuscirebbe a firmare correttamente — questo sarebbe una difesa in profondità sopra quello, non una correzione per un buco noto.
- **Attivare `purchasable`**, un prodotto reale alla volta, una volta che tutto quanto sopra è a posto.

Nessuna di queste cambia come funziona il flusso principale — sono ciò che separa "questo elabora correttamente un pagamento" da "questo è ciò che un'azienda vera gestisce senza supervisione".

## Per concludere

Tre episodi, un'app Rails piccola ma completa: un checkout che non si fida mai di un prezzo dal client, un gateway di pagamento che firma ciò che manda e verifica ciò che riceve, un webhook che sopravvive ai tentativi ripetuti, e una suite di test più un'esecuzione reale in sandbox a garantire tutto questo. Se stai integrando XPay tu stesso, il [repo completo](https://github.com/AntoninoScaffidi/nexi-xpay-with-rails) è lì da clonare, leggere e adattare.
