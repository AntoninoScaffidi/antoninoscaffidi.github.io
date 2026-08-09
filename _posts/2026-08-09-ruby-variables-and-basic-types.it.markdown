---
layout: post
title: "Ruby Deep Dive: variabili e tipi base"
series: "ruby-deep-dive"
episode: 1
lang: it
ref: ruby-variables-and-basic-types
permalink: /ruby-variables-and-basic-types/
canonical_url: https://antoninoscaffidi.github.io/it/ruby-variables-and-basic-types/
image: /assets/images/ruby-deep-dive-banner.png
date: 2026-08-09 09:00:00 +0200
---

Questo è il primo episodio di **Ruby Deep Dive**, una serie un po' diversa dalle altre due su questo blog. VicinoTe e AI with Ruby parlano entrambe di *costruire cose*. Questa parla di *capire il linguaggio in sé* — Ruby, da solo, niente Rails, niente framework. Non ha una cadenza fissa: viene scritta quando c'è tempo per affrontare un concetto per bene.

Ogni episodio ha esercizi in un repo di accompagnamento: [ruby-deep-dive](https://github.com/AntoninoScaffidi/ruby-deep-dive), taggato [`episode-1`](https://github.com/AntoninoScaffidi/ruby-deep-dive/tree/episode-1). Ognuno è uno stub di metodo da completare, verificato da un test. Leggi l'episodio, poi vai a far passare i test da solo — gli esercizi restano volutamente irrisolti nel repo, perché farli è il punto.

Iniziamo dal vero inizio: cos'è una variabile, e i tipi base di Ruby.

## Cos'è davvero una variabile

```ruby
language = "Ruby"
```

Questa riga fa due cose: crea un oggetto `String` che contiene il testo `"Ruby"`, e fa sì che il nome `language` punti a esso. Vale la pena essere precisi su questo, perché "variabile" può essere una parola fuorviante se vieni da un linguaggio dove le variabili sono scatole che contengono valori direttamente. In Ruby, una variabile è un'**etichetta**, e ciò che etichetta è un **oggetto** che vive da qualche parte in memoria. Due variabili possono puntare esattamente allo stesso oggetto:

```ruby
a = "Ruby"
b = a
b.upcase!
a #=> "RUBY"
```

Cambiare `b` ha cambiato anche `a`, perché non erano mai state due stringhe separate — erano due etichette sullo stesso oggetto. `upcase!` (con il `!`) muta la stringa sul posto, invece di restituirne una nuova. Questa distinzione — se una variabile sia un'etichetta o una scatola — è la radice di molta confusione più avanti con gli argomenti dei metodi e la mutazione, quindi vale la pena averla chiara fin dall'episodio 1.

Ruby non richiede di dichiarare il tipo di una variabile. `language` non è "una variabile stringa" — è solo un nome che al momento punta a una `String`. Assegnale qualcos'altro, e punterà a quello:

```ruby
language = "Ruby"
language = 42
```

Entrambe le righe sono perfettamente legali, una dopo l'altra. La variabile non ha cambiato tipo — ha solo iniziato a puntare altrove.

## Come si nominano le variabili locali

Un nome di variabile locale inizia con una lettera minuscola o un underscore, e per convenzione usa `snake_case`:

```ruby
favorite_language = "Ruby"
_unused = "inizia con un underscore, spesso usato per segnalare 'so che non la sto usando'"
```

Ruby è più rigido di molti linguaggi su questa convenzione: i nomi in `CamelCase` sono riservati a costanti e nomi di classi/moduli, quindi `FavoriteLanguage = "Ruby"` non crea una variabile locale — crea una *costante*, una cosa di tipo completamente diverso, e Ruby ti avviserà se provi a riassegnarla.

## I tipi base

### String

Testo, tra virgolette doppie o singole. La differenza conta: le stringhe tra virgolette doppie supportano l'**interpolazione** e le sequenze di escape, quelle a virgolette singole no.

```ruby
name = "Antonino"
"Hello, #{name}!"   #=> "Hello, Antonino!"
'Hello, #{name}!'   #=> "Hello, #{name}!" (letteralmente, nessuna interpolazione)
```

L'interpolazione — `#{...}` dentro una stringa a virgolette doppie — valuta qualsiasi espressione Ruby ci sia dentro le graffe e ne inserisce la forma testuale. È il modo idiomatico di costruire stringhe in Ruby; usala invece della concatenazione con `+`.

### Symbol

```ruby
:ruby
```

Un `Symbol` assomiglia a una stringa con i due punti davanti, ed è tentante pensarlo come "una stringa che non può cambiare". È vicino al vero, ma il modo più utile di pensarci è: un `Symbol` è un **nome**, usato come identificatore, non come dato da mostrare o manipolare. Lo stesso symbol scritto due volte è sempre esattamente lo stesso oggetto in memoria:

```ruby
:ruby.object_id == :ruby.object_id  #=> true
"ruby".object_id == "ruby".object_id #=> false
```

Ecco perché i symbol compaiono continuamente come chiavi di hash e nomi di metodo nel codice Ruby — sono economici da confrontare (Ruby controlla solo se è lo stesso oggetto, non carattere per carattere) ed economici da salvare (nessuna copia duplicata). La regola pratica: se il valore è qualcosa che un umano leggerà a schermo, probabilmente è una `String`. Se è un'etichetta interna che il programma usa per riferirsi a qualcosa, probabilmente è un `Symbol`.

### Integer e Float

```ruby
42        # Integer
3.14      # Float
```

Una cosa che sorprende chi viene da altri linguaggi: la divisione tra interi tronca, non arrotonda né solleva un errore.

```ruby
7 / 2     #=> 3, non 3.5
7 / 2.0   #=> 3.5
7.0 / 2   #=> 3.5
```

Se uno dei due operandi è un `Float`, il risultato è un `Float`. Se sono entrambi `Integer`, ottieni la divisione intera. È esattamente per questo che `add_as_float` negli esercizi di questo episodio chiede di convertire prima di sommare — `2 + 3` fa `5`, un `Integer`, in qualunque modo tu scriva il metodo.

### nil, true e false

`nil` rappresenta l'assenza di un valore — non zero, non una stringa vuota, davvero *niente*. `true` e `false` sono gli unici due valori di un tipo separato ciascuno, rispettivamente `TrueClass` e `FalseClass` (sì, `true` e `false` sono ciascuno l'unica istanza della propria classe — vedremo perché non è strano quanto sembra quando arriveremo a "tutto è un oggetto" nell'episodio 5).

Ecco il dettaglio che inciampa quasi tutti quelli che arrivano da un altro linguaggio: **in Ruby, solo `nil` e `false` sono falsy.** Tutto il resto è truthy — incluso `0`, inclusa `""` (stringa vuota), inclusi array e hash vuoti.

```ruby
if 0
  puts "questo viene eseguito"   # sì! 0 è truthy in Ruby
end

if ""
  puts "anche questo"  # eseguito anch'esso
end
```

Venendo da JavaScript, Python o PHP, dove `0` e `""` sono falsy, questa è la fonte di confusione più comune in assoluto del tipo "perché il mio `if` ha fatto questo". Tienilo a mente e smetterà presto di sorprenderti.

## Gli esercizi

Vai su [`01-variables-and-basic-types/exercises.rb`](https://github.com/AntoninoScaffidi/ruby-deep-dive/blob/main/01-variables-and-basic-types/exercises.rb) nel repo. Sette metodi, tutti vuoti, tutti riguardanti qualcosa di questo post: assegnazione, interpolazione, conversione da stringa a intero, symbol, controllo di `nil`, forzare un risultato `Float` e — volutamente — trovare "l'altro valore falsy" senza che sia nominato direttamente nell'esercizio.

```bash
git clone https://github.com/AntoninoScaffidi/ruby-deep-dive.git
cd ruby-deep-dive
bundle install
ruby 01-variables-and-basic-types/exercises_test.rb
```

Ogni test fallirà finché non completi il metodo corrispondente. È lo stato di partenza previsto — falli diventare verdi uno alla volta.

## Cosa viene dopo

L'episodio 2 copre il controllo di flusso: `if`/`unless`/`case`, i loop, e uno sguardo più da vicino a truthy/falsy ora che i tipi base sono a posto.
