---
layout: post
title: "Ruby Deep Dive: metodi"
series: "ruby-deep-dive"
episode: 3
lang: it
ref: ruby-methods
permalink: /ruby-methods/
canonical_url: https://antoninoscaffidi.github.io/it/ruby-methods/
image: /assets/images/ruby-deep-dive-ep3-banner.png
date: 2026-09-01 09:00:00 +0200
---

L'[episodio 2]({% post_url 2026-08-30-ruby-control-flow %}) ha coperto `if`, `case` e i loop — i costrutti che decidono *cosa* viene eseguito. Questo episodio parla di ciò che viene effettivamente deciso di eseguire: i metodi. Non ancora le classi (quello è territorio dell'episodio 4) — solo un metodo standalone, i suoi argomenti, e cosa restituisce a chi lo ha chiamato.

Il codice è nel repo [ruby-deep-dive](https://github.com/AntoninoScaffidi/ruby-deep-dive), taggato [`episode-3`](https://github.com/AntoninoScaffidi/ruby-deep-dive/tree/episode-3). Stesso formato di sempre: leggi il post, poi completa [`03-methods/exercises.rb`](https://github.com/AntoninoScaffidi/ruby-deep-dive/blob/main/03-methods/exercises.rb) finché [`03-methods/exercises_test.rb`](https://github.com/AntoninoScaffidi/ruby-deep-dive/blob/main/03-methods/exercises_test.rb) non diventa tutto verde.

## Definire un metodo

```ruby
def add(a, b)
  a + b
end
```

`def`, un nome, parentesi intorno ai parametri (facoltative, ma vale la pena scriverle sempre), un corpo, `end`. Nessuna dichiarazione di tipo da nessuna parte — non su `a`, non su `b`, non su cosa `add` restituisce. È la stessa idea dell'episodio 1 — "un nome è solo un'etichetta, non una scatola con un tipo dichiarato" — applicata ai parametri di metodo: `a` e `b` sono solo nomi che vengono legati a qualunque oggetto sia stato passato, di qualunque tipo esso sia.

## Il valore di ritorno: implicito ed esplicito

`add` non ha la keyword `return`, eppure restituisce comunque `a + b`. **Un metodo Ruby restituisce sempre il valore dell'ultima espressione valutata nel suo corpo** — è la stessa regola stabilita nell'episodio 2 per `if`, estesa a tutto il metodo. Non c'è nulla di speciale in `return` che fa "restituire qualcosa" a un metodo — ogni metodo restituisce sempre qualcosa, che tu scriva la keyword o no.

`return` serve per il caso in cui devi restituire un valore *prima* di arrivare alla fine del metodo — un'uscita anticipata:

```ruby
def safe_divide(a, b)
  return nil if b.zero?
  a / b.to_f
end
```

Leggilo così: "se `b` è zero, fermati qui e restituisci `nil`; altrimenti continua, e qualunque cosa valuti l'ultima riga è la risposta." Il `return` sulla prima riga è esplicito e necessario — senza, `nil` sarebbe solo un valore in mezzo al metodo, scartato. La `a / b.to_f` sull'ultima riga non ha bisogno di `return` — è già esattamente dove la regola del ritorno implicito va a cercare un valore. Il Ruby idiomatico si appoggia continuamente a questo: usa `return` esplicito per le uscite anticipate e le guard clause, e lascia che l'ultima riga faccia il resto ovunque altro. Un `return` sull'ultimissima riga di un metodo è legale ma ridondante — lo vedrai raramente nel codice Ruby.

## Argomenti posizionali e con default

`add(a, b)` e `greet(name)` prendono entrambi argomenti **posizionali** — il primo valore del chiamante riempie il primo parametro, il secondo il secondo, abbinati puramente per posizione, non per nome. Dai a un parametro un valore di default, e il chiamante può ometterlo:

```ruby
def greet(name, greeting = "Hello")
  "#{greeting}, #{name}!"
end

greet("Ada")        #=> "Hello, Ada!"
greet("Ada", "Hi")  #=> "Hi, Ada!"
```

`greeting = "Hello"` non viene eseguito al momento della definizione del metodo — viene eseguito *ogni volta che il metodo è chiamato senza quell'argomento*, quindi l'espressione di default può persino fare riferimento a parametri precedenti o chiamare altri metodi; viene valutata di nuovo a ogni chiamata, non calcolata una volta sola e riusata.

## Argomenti keyword

Gli argomenti posizionali hanno un vero punto debole appena un metodo ne prende più di due o tre: al punto di chiamata, `describe_person("Ada", 36)` non ti dice nulla su quale numero sia cosa senza andare a controllare la definizione del metodo. Gli argomenti keyword risolvono il problema nominando il valore proprio dove viene passato:

```ruby
def describe_person(name:, age: 0)
  "#{name} (#{age})"
end

describe_person(name: "Ada")            #=> "Ada (0)"
describe_person(name: "Ada", age: 36)   #=> "Ada (36)"
describe_person(age: 36, name: "Ada")   #=> "Ada (36)" -- l'ordine non conta
```

`name:` senza nulla dopo i due punti nella lista dei parametri significa "obbligatorio — chiamare questo metodo senza un argomento `name:` è un `ArgumentError`, non un `nil`." `age: 0` significa "facoltativo, con default `0`" — esattamente la stessa idea di valore di default degli argomenti posizionali, solo scritta con una keyword. Due scelte indipendenti — posizionale vs. keyword, e obbligatorio vs. con default — che si combinano liberamente. Usa argomenti keyword nel momento in cui la lista dei parametri di un metodo supera uno o due valori, o quando due parametri sono dello stesso tipo e potrebbero essere scambiati silenziosamente da chi chiama (due interi, due stringhe) — la keyword rende quell'errore impossibile da scrivere per sbaglio.

## Splat: catturare il resto

A volte un metodo genuinamente non sa in anticipo quanti argomenti riceverà. `*` in una lista di parametri — lo **splat** — raccoglie ogni argomento posizionale rimanente in un `Array`:

```ruby
def sum_all(*numbers)
  numbers.sum
end

sum_all(1, 2, 3)  #=> 6
sum_all()         #=> 0
sum_all(5)        #=> 5
```

Dentro il metodo, `numbers` è semplicemente un `Array` normale, che il chiamante abbia passato zero, uno o cinquanta argomenti — non c'è nessun tipo speciale "argomenti variadici" da imparare, lo splat li impacchetta semplicemente nello stesso `Array` che già conosci dal materiale Tier-1 dell'episodio 1. Il doppio splat, `**`, fa l'equivalente per gli argomenti keyword — raccoglie ogni argomento keyword passato dal chiamante in un `Hash`:

```ruby
def build_profile(**attributes)
  attributes
end

build_profile(name: "Ada", age: 36)  #=> { name: "Ada", age: 36 }
```

`*` e `**` non servono solo a definire metodi flessibili — funzionano anche al contrario, al *punto di chiamata*, per espandere una collezione che già hai in argomenti separati: `sum_all(*[1, 2, 3])` è esattamente la stessa chiamata di `sum_all(1, 2, 3)`.

## Splat a sinistra: assegnazione destrutturante

Lo stesso `*` compare in un posto che non è affatto una definizione di metodo — la semplice assegnazione:

```ruby
first, *rest = [1, 2, 3, 4]
first  #=> 1
rest   #=> [2, 3, 4]
```

Questa si chiama assegnazione destrutturante (o "multipla"): gli elementi del lato destro vengono distribuiti tra i nomi a sinistra, e `*rest` assorbe tutto ciò che avanza in un `Array` — vuoto se non resta nulla, mai `nil`:

```ruby
def first_and_rest(array)
  first, *rest = array
  [first, rest]
end

first_and_rest([1, 2, 3])  #=> [1, [2, 3]]
first_and_rest([5])        #=> [5, []]
first_and_rest([])         #=> [nil, []]
```

`first_and_rest([])` restituisce `[nil, []]`, non un errore — destrutturare un array vuoto lascia semplicemente `first` non assegnato (`nil`), esattamente come qualunque variabile locale è `nil` prima di essere mai impostata.

## Convenzioni di naming: `?` e `!`

Due suffissi sono convenzioni che Ruby non impone da nessuna parte nel linguaggio stesso, ma che ogni rubysta si aspetta vengano rispettate:

```ruby
def palindrome?(str)
  cleaned = str.downcase
  cleaned == cleaned.reverse
end
```

Un metodo che finisce in `?` è una promessa, non una regola che l'interprete controlla: **questo metodo restituisce un booleano, e non cambia nulla.** `empty?`, `nil?`, `even?` — tutti metodi predefiniti che seguono la stessa convenzione. Niente ti impedisce di scrivere `def valid?; @errors = []; true; end`, ma farlo viola una convenzione su cui ogni lettore del tuo codice fa affidamento, silenziosamente, dando per scontato che sia vera.

```ruby
def reverse_in_place!(array)
  array.reverse!
end
```

Un metodo che finisce in `!` è l'altra promessa: **questo metodo muta qualcosa a cui il chiamante aveva già un riferimento**, invece di costruire e restituire qualcosa di nuovo. Questo si ricollega direttamente all'idea dell'episodio 1 — "una variabile è un'etichetta, non una scatola" — `array.reverse!` non crea un nuovo array invertito e lo restituisce; inverte lo *stesso* oggetto `Array` sul posto. Anche la variabile originale del chiamante vede il cambiamento, perché ha sempre puntato a quell'unico oggetto:

```ruby
array = [1, 2, 3]
result = reverse_in_place!(array)
result.equal?(array)  #=> true -- letteralmente lo stesso oggetto, non solo valori uguali
```

`!` non significa tecnicamente "pericoloso" o "muta" secondo nessuna regola imposta dal linguaggio (`Kernel#exit!`, per esempio, non muta nulla) — la versione più precisa in una riga è "questo metodo fa qualcosa su cui dovresti riflettere due volte prima di chiamarlo," e "muta il suo argomento o receiver sul posto" è di gran lunga la ragione più comune per cui succede. La regola pratica che regge davvero: quando un metodo ha sia una versione normale che una versione `!` (`sort`/`sort!`, `reverse`/`reverse!`, `upcase`/`upcase!`), quella normale restituisce un nuovo oggetto e lascia stare l'originale, e quella `!` cambia direttamente l'oggetto originale.

## Gli esercizi

Nove metodi in [`03-methods/exercises.rb`](https://github.com/AntoninoScaffidi/ruby-deep-dive/blob/main/03-methods/exercises.rb): ritorno implicito, `return` esplicito per l'uscita anticipata, un argomento con default, argomenti keyword obbligatori e con default, uno splat che raccoglie argomenti posizionali, un doppio splat che raccoglie argomenti keyword in un `Hash`, un metodo predicato `?`, un metodo mutante `!`, e assegnazione destrutturante con uno splat.

```bash
git clone https://github.com/AntoninoScaffidi/ruby-deep-dive.git
cd ruby-deep-dive
bundle install
ruby 03-methods/exercises_test.rb
```

## Cosa viene dopo

L'episodio 4 passa dai metodi standalone alle classi e agli oggetti: `initialize`, variabili di istanza, e `attr_accessor` — il punto in cui "un metodo" comincia a diventare "un metodo che appartiene a qualcosa."
