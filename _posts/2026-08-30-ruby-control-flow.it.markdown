---
layout: post
title: "Ruby Deep Dive: controllo di flusso"
series: "ruby-deep-dive"
episode: 2
lang: it
ref: ruby-control-flow
permalink: /ruby-control-flow/
canonical_url: https://antoninoscaffidi.github.io/it/ruby-control-flow/
image: /assets/images/ruby-deep-dive-ep2-banner.png
date: 2026-08-30 09:00:00 +0200
---

L'[episodio 1]({% post_url 2026-08-09-ruby-variables-and-basic-types %}) ha coperto le variabili e i tipi base, incluso la regola truthy/falsy di Ruby: solo `nil` e `false` sono falsy, tutto il resto — incluso `0` e `""` — è truthy. Quella regola è la base di tutto questo episodio, perché ogni ramo condizionale e ogni loop in Ruby si riduce, alla fine, a "questo è truthy o falsy?"

Il codice è nel repo [ruby-deep-dive](https://github.com/AntoninoScaffidi/ruby-deep-dive), taggato [`episode-2`](https://github.com/AntoninoScaffidi/ruby-deep-dive/tree/episode-2). Stesso formato dell'episodio 1: leggi il post, poi completa [`02-control-flow/exercises.rb`](https://github.com/AntoninoScaffidi/ruby-deep-dive/blob/main/02-control-flow/exercises.rb) finché [`02-control-flow/exercises_test.rb`](https://github.com/AntoninoScaffidi/ruby-deep-dive/blob/main/02-control-flow/exercises_test.rb) non diventa tutto verde.

## if / elsif / else

```ruby
def grade_for(score)
  if score >= 90
    "A"
  elsif score >= 80
    "B"
  else
    "F"
  end
end
```

Niente di sorprendente in superficie — ma c'è una cosa su cui vale la pena essere precisi: **`if` è un'espressione, non un'istruzione.** Restituisce il valore del ramo che è stato eseguito, e quel valore è ciò che il metodo restituisce (non c'è nessun `return` qui — l'ultima espressione valutata è il valore di ritorno del metodo, un'altra cosa vera in tutto Ruby, non specifica di `if`). Se nessun ramo corrisponde e non c'è `else`, l'intero `if` restituisce `nil`:

```ruby
result = if false
  "non succede mai"
end
result #=> nil
```

Questo è anche il motivo per cui Ruby ha una **forma modificatore** — una singola riga che si legge quasi come inglese, usata continuamente per le guard clause:

```ruby
puts "adulto" if age >= 18
return nil unless user
```

Usa la forma modificatore quando il corpo è una riga breve e non c'è `elsif`/`else`; usa la forma multi-riga nel momento in cui compare un secondo ramo o il corpo richiede più di un'istruzione.

## unless

`unless condizione` è esattamente `if !condizione`, niente di più:

```ruby
def bouncer_message(age)
  unless age < 18
    "Come on in!"
  else
    "Sorry, 18+ only."
  end
end
```

`unless ... else` è Ruby legale, ma la maggior parte delle style guide (e la maggior parte dei rubysti) lo scoraggia, per una ragione di leggibilità che vale la pena interiorizzare invece di accettare come regola fine a se stessa: con `unless`, il ramo `else` viene eseguito quando la condizione **è** vera, il che significa che ogni lettore deve negare la condizione nella propria testa prima che l'`else` abbia senso. `if cond ... else ...` si legge nell'ordine in cui il cervello lo elabora; `unless cond ... else ...` si legge al contrario. Riserva `unless` a ciò per cui è davvero indicato: una singola condizione di guardia senza alcun `else` — `unless` brilla esattamente dove brilla la forma modificatore, es. `return unless valid?`.

## L'operatore ternario

```ruby
def parity_word(n)
  n.even? ? "even" : "odd"
end
```

`condizione ? se_vero : se_falso` non è un costrutto di controllo di flusso a sé — è `if`/`else` come espressione, scritto su una riga. Usalo solo quando entrambi i rami sono brevi e sono esattamente due; un ternario con un altro ternario annidato dentro è un code smell, non uno stile compatto. Se serve un terzo ramo, quello è `case/when` o un `if/elsif/else` completo, non una catena di `?:`.

## case/when — e l'operatore che lo fa funzionare

```ruby
def parking_fee(hours)
  case hours
  when 0..1 then 0
  when 2..4 then 5
  when 5..8 then 10
  else 15
  end
end
```

Sembra uno `switch` di altri linguaggi, ma sotto il cofano sta facendo qualcosa di significativamente diverso, e capirlo cambia ciò per cui useresti `case/when`. Ogni clausola `when` non testa `hours == 0..1` — chiama **`(0..1) === hours`**, l'operatore di *uguaglianza case*, con il valore di `when` a sinistra. `Range#===` è definito come "questo range copre il valore" (`include?`), ed è esattamente per questo che i range funzionano come clausole `when`. Non è sintassi speciale per i range — è `===` che si comporta diversamente a seconda di quale classe lo definisce:

```ruby
(0..1) === 1        #=> true   (Range#===: appartenenza)
String === "hi"      #=> true   (Module#===: is_a?)
/^h/ === "hi"         #=> true  (Regexp#===: corrispondenza)
42 === 42             #=> true  (Object#===: ricade su ==)
```

Questo significa che `case/when` non è limitato a valori e range — un ramo `when String` corrisponde a qualunque cosa sia una `String`, e un ramo `when /^h/` corrisponde a qualunque cosa la regex faccia match. È uno strumento di dispatch molto più generale di "switch su un valore", una volta capito cosa viene davvero chiamato.

L'altro dettaglio da sapere: una clausola `when` può elencare più candidati separati da virgola, verificati in OR:

```ruby
def day_type(day)
  case day
  when :saturday, :sunday
    "Weekend"
  when :monday, :tuesday, :wednesday, :thursday, :friday
    "Weekday"
  else
    "Not a day"
  end
end
```

`when :saturday, :sunday` esegue il ramo se `:saturday === day` **oppure** `:sunday === day` — un ramo solo, più valori accettati, senza dover ripetere il corpo per ogni giorno.

E un `case` non ha bisogno di un soggetto. Scritto senza, ogni `when` è una condizione booleana completa, valutata dall'alto in basso come una catena `if/elsif` — utile davvero quando le condizioni non testano tutte la stessa variabile:

```ruby
case
when age < 13 then "bambino"
when age < 20 then "teenager"
else "adulto"
end
```

## while e until

```ruby
def sum_up_to(n)
  total = 0
  i = 1
  while i <= n
    total += i
    i += 1
  end
  total
end
```

`while condizione ... end` esegue il corpo finché la condizione resta truthy, controllandola *prima* di ogni iterazione — inclusa la primissima, quindi un `while` la cui condizione parte falsa non gira mai nemmeno una volta (è esattamente per questo che `sum_up_to(0)` restituisce `0`: `i` parte da `1`, `1 <= 0` è falso subito, il corpo del loop non viene mai eseguito). `until condizione` è `while !condizione` — stesso loop, controllo invertito:

```ruby
def countdown(n)
  result = []
  i = n
  until i < 0
    result << i
    i -= 1
  end
  result
end
```

Anche entrambi hanno una forma modificatore, stessa idea di `if`/`unless`:

```ruby
i += 1 while i < 10
```

Fai attenzione con la forma modificatore in particolare intorno a `begin...end while` — ha una particolarità (il corpo viene eseguito una volta *prima* del primo controllo, a differenza di ogni altro `while`) abbastanza rara nella pratica da non valere la pena memorizzarla adesso, ma vale la pena sapere che esiste così da non sorprenderti più avanti.

## loop e break con un valore

`while`/`until` hanno bisogno di una condizione all'inizio. A volte il punto di uscita naturale è nel *mezzo* del corpo, non qualcosa esprimibile come condizione pre-loop. È a questo che serve `loop`:

```ruby
def first_power_of_two_above(limit)
  power = 1
  loop do
    break power if power > limit
    power *= 2
  end
end
```

`loop` è un metodo normale (`Kernel#loop`) che prende un blocco e lo esegue per sempre, finché qualcosa al suo interno non lo ferma. `break` ferma il loop — ed ecco il dettaglio facile da perdere venendo da altri linguaggi: **`break valore` non si limita a uscire dal loop, fa sì che l'intera espressione `loop do...end` restituisca `valore`.** Un loop `while` restituisce sempre `nil`, comunque finisca; `loop` restituisce qualunque cosa `break` gli abbia passato (o `nil` se non fa mai `break` con un valore). Ecco perché `first_power_of_two_above` non ha bisogno di una variabile dichiarata fuori dal loop per "ricordare" la risposta — il loop stesso *è* la risposta, una volta che ci fai `break` con essa.

## next

Dentro qualunque loop, `next` salta direttamente all'iterazione successiva senza eseguire il resto del corpo:

```ruby
def sum_of_evens(numbers)
  total = 0
  numbers.each do |n|
    next if n.odd?
    total += n
  end
  total
end
```

`next if n.odd?` si legge come una guard clause in testa al blocco: "se questo non è idoneo, passa oltre" — la stessa forma di un `return` anticipato dentro un metodo, solo per una singola iterazione del loop invece che per l'intera chiamata. (`numbers.each` è tecnicamente un metodo `Enumerable` più che una parola chiave di controllo di flusso come `while`/`until`/`loop` — `each` e i suoi parenti avranno un intero episodio tutto loro più avanti in questa serie — ma `next`/`break` funzionano dentro il suo blocco esattamente come dentro `while`, `until` o `loop`.)

## for — e perché lo vedrai raramente

Ruby ha davvero un loop `for`:

```ruby
for i in 1..3
  puts i
end
```

Viene menzionato qui soprattutto perché tu lo riconosca se ci incappi, non perché tu debba usarlo. Il motivo per cui il Ruby idiomatico evita `for` a favore di `each`/`times`/ecc. è una differenza di scoping, e vale la pena vederla una volta direttamente:

```ruby
for i in 1..3
end
i #=> 3, ancora accessibile qui

[1, 2, 3].each do |i|
end
i #=> NameError: undefined local variable or method 'i'
```

`for` non introduce un nuovo scope — la variabile del loop finisce nello scope circostante, esattamente come farebbe il contatore di un normale loop `while`. Un blocco passato a `each` *introduce* invece un proprio scope, quindi `i` dentro il blocco è una variabile locale nuova che smette di esistere nel momento in cui il blocco finisce. Quel contenimento è esattamente il comportamento che vuoi quasi sempre, ed è la vera ragione per cui `each` ha vinto su `for` come idioma Ruby — non "for è deprecato", solo "lo scoping di each è più sicuro di default".

## Truthy e falsy, un livello più a fondo

L'episodio 1 ha stabilito la regola: solo `nil` e `false` sono falsy. Ora che `if`/`while`/`unless` sono tutti a posto, ecco l'idioma che quella regola rende possibile, e che compare continuamente nel codice Ruby reale:

```ruby
name = nil
display_name = name || "Guest"
```

`||` non produce solo `true`/`false` — valuta il lato sinistro, e se è truthy, **restituisce proprio quel valore**; solo se il lato sinistro è falsy valuta e restituisce il lato destro. `&&` funziona allo stesso modo ma al contrario: restituisce il lato sinistro se è falsy (interrompendo la valutazione prima ancora di toccare il lato destro), altrimenti restituisce il lato destro. Questo è il vero meccanismo dietro il pattern `x || default`, estremamente comune — non è una sintassi speciale per "valore predefinito", è `||` che fa esattamente ciò che fa sempre, applicato a un caso in cui quel comportamento è proprio quello che serve.

Un'altra cosa che vale la pena segnalare ora piuttosto che dopo che ti abbia morso: Ruby ha anche `and`/`or`/`not` come equivalenti in parole inglesi di `&&`/`||`/`!`. **Non** sono intercambiabili con gli operatori simbolici, perché legano a una precedenza molto più bassa — più bassa di `=`:

```ruby
x = false or true
x #=> false
```

Sembra che dovrebbe impostare `x` a `true`, ma `=` lega più stretto di `or`, quindi viene in realtà interpretato come `(x = false) or true` — `x` prende `false`, e l'`or true` non fa nulla di utile. Questa è una trappola ben nota, non un rischio stilistico improbabile: attieniti a `&&`/`||`/`!` per la logica vera e propria, e considera `and`/`or`/`not` di fatto inutilizzati nel Ruby idiomatico.

## Gli esercizi

Dieci metodi in [`02-control-flow/exercises.rb`](https://github.com/AntoninoScaffidi/ruby-deep-dive/blob/main/02-control-flow/exercises.rb), uno per ogni concetto trattato sopra: `if`/`elsif`/`else`, `unless`, l'operatore ternario, `case/when` con range, `case/when` con valori separati da virgola, `while`, `until`, `loop`/`break` con un valore, `next`, e la truthiness nativa di Ruby via `!!`.

```bash
git clone https://github.com/AntoninoScaffidi/ruby-deep-dive.git
cd ruby-deep-dive
bundle install
ruby 02-control-flow/exercises_test.rb
```

## Cosa viene dopo

L'episodio 3 copre i metodi: definizioni, argomenti (posizionali, keyword, con default, splat), e cosa restituisce davvero un metodo.
