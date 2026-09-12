---
layout: post
title: "SEO Semantica: Oltre le Keyword"
series: "semantic-seo"
episode: 1
lang: it
ref: semantic-seo-beyond-the-keyword
permalink: /semantic-seo-beyond-the-keyword/
canonical_url: https://antoninoscaffidi.github.io/it/semantic-seo-beyond-the-keyword/
image: /assets/images/semantic-seo-ep1-banner.png
date: 2026-09-12 09:00:00 +0200
---

Questo è il primo episodio di **SEO Semantica**, una nuova serie su questo blog — annunciata [un paio di settimane fa]({% post_url 2026-08-30-two-new-series-coming %}) insieme alla sua controparte tecnica, [Ricerca Semantica in Ruby on Rails]({% post_url 2026-08-30-two-new-series-coming %}). A differenza della maggior parte di quello che trovi qui, questa serie non parla affatto di Rails o di codice. È per chiunque scriva online, gestisca un sito, o si sia mai chiesto perché certe pagine si posizionano bene e altre — apparentemente piene delle parole "giuste" — no.

## Il vecchio modo: far combaciare le stringhe

Per gran parte della storia del web, posizionarsi bene nella ricerca ha significato soprattutto una cosa: mettere le parole giuste nella pagina, nei posti giusti, il numero giusto di volte. Se qualcuno cercava "dove mangiare il vero suino nero dei Nebrodi", la strategia vincente era assicurarsi che quella frase esatta — o varianti molto simili — comparisse nel titolo, nei sottotitoli, qualche volta nel corpo del testo, magari nel testo alternativo delle immagini. I motori di ricerca, nel loro nucleo, facevano string matching: questo documento contiene questi caratteri, più o meno in questo ordine? Più precisamente una pagina rifletteva la query, più "meritava" di posizionarsi.

Questo ha prodotto una scrittura genuinamente strana. Le pagine ripetevano una frase cinque o sei volte in tre paragrafi, in un modo che nessun essere umano scriverebbe mai davvero, puramente perché l'algoritmo contava le occorrenze. Quell'epoca è ormai quasi finita, e vale la pena capire esattamente cosa l'ha sostituita — non come curiosità SEO, ma perché cambia cosa significhi davvero "scrivere bene per la ricerca".

## Cosa è cambiato davvero

**Hummingbird**, lanciato nel 2013, è stato il primo grande cambiamento: invece di analizzare una query come un insieme di keyword separate, Google ha iniziato a cercare di capire la query *nel suo insieme* — la relazione tra le parole, non solo la loro presenza. "Dove posso assaggiare il suino nero dei Nebrodi vicino a Floresta" non sono cinque keyword indipendenti da far combaciare; è una domanda unica con un intento specifico, e Hummingbird è stato costruito per trattarla così.

**RankBrain**, aggiunto all'algoritmo principale nel 2015, ha portato il machine learning direttamente nel ranking — in particolare per gestire il circa 15% delle ricerche giornaliere che Google non aveva mai visto prima. Per una query genuinamente nuova, non esiste una cronologia di "cosa hanno cliccato le persone per questa frase esatta" da cui imparare. Il compito di RankBrain era mappare una query sconosciuta sui *concetti* dietro query simili e familiari, e posizionare di conseguenza — un passaggio esplicito dal far combaciare il testo al modellare il significato.

**BERT**, nel 2019, è andato ancora più a fondo nei meccanismi del linguaggio stesso. I sistemi precedenti leggevano perlopiù una query come un insieme approssimativo di parole importanti, tendendo a ignorare come rumore piccole parole connettive come "a", "per" o "non". BERT legge le parole *in relazione alle parole intorno a loro*, in entrambe le direzioni contemporaneamente — ed è esattamente per questo che una preposizione può ribaltare il significato di un'intera frase, e ora ribalta anche i risultati. "Cittadino brasiliano che viaggia negli USA nel 2019 ha bisogno del visto" significa qualcosa di diverso da "cittadino USA che viaggia in Brasile nel 2019 ha bisogno del visto" — stesse parole, ordine diverso, intento opposto — ed è esattamente il tipo di distinzione che il vecchio keyword matching non era costruito per cogliere.

Più di recente, gli **AI Overview**, e ora l'**AI Mode** — che Google ha confermato all'I/O 2026 non essere più sperimentale, in esecuzione sul modello Gemini 3.5 Flash a livello globale, con 2,5 miliardi di utenti mensili in oltre 200 paesi — vanno ancora oltre: per molte query, la pagina dei risultati ora include una risposta generata dall'AI e sintetizzata *da più fonti insieme*, prima ancora che compaia un singolo link blu. Il motore non si limita più a posizionare pagine che corrispondono a una query — ne legge diverse, estrae ciò che è rilevante per la domanda specifica, e scrive una risposta, completa di citazioni in linea verso le fonti da cui ha attinto. Posizionarsi bene significa sempre di più essere una delle fonti giudicate degne di essere sintetizzate, non solo uno dei dieci link blu.

## Cosa significa davvero "semantico", concretamente

Messo insieme, ecco il cambiamento pratico: i motori di ricerca rappresentano sempre di più sia le query che i contenuti come **significato**, non come sequenze di caratteri. Sotto il cofano, questo tipicamente comporta convertire il testo in un **embedding** — una rappresentazione numerica posizionata in uno spazio dove testi che parlano di cose simili finiscono vicini tra loro, indipendentemente dal fatto che condividano le stesse parole. "Vero maiale allo stato semibrado dei boschi dei Nebrodi" e "dove prendono davvero il suino nero i locali, non la versione da souvenir" possono finire vicine in quello spazio, perché *parlano* della stessa cosa, anche con quasi nessuna sovrapposizione di vocabolario. (La [serie ai-with-ruby]({% post_url 2026-08-22-semantic-search-with-pgvector %}) di questo blog costruisce esattamente questo tipo di sistema, se vuoi vedere il meccanismo dall'interno.)

È anche così che un motore di ricerca riconosce le **entità** — una persona, un luogo, un prodotto o un concetto specifico — e collega una pagina a ciò di cui parla davvero, invece che alle parole che per caso compaiono su di essa. Una pagina può posizionarsi per "consigli suino nero dei Nebrodi autentico" senza mai usare quella frase esatta, purché parli davvero, e in modo approfondito, di quell'argomento.

## Vedere la differenza

Una scrittura vecchio stile, guidata dalle keyword, mirata a "dove mangiare il vero suino nero dei Nebrodi", potrebbe assomigliare a questo:

> Cerchi dove mangiare il vero suino nero dei Nebrodi? Il nostro ristorante serve il miglior suino nero dei Nebrodi per chi cerca il vero suino nero dei Nebrodi. Prenota oggi per assaggiare il nostro suino nero dei Nebrodi!

Ogni frase esiste per ripetere la frase, non per dire qualcosa di nuovo. Una versione scritta in modo semantico sullo stesso argomento si legge come una persona che conosce davvero l'argomento:

> Il suino nero dei Nebrodi non è un maiale da allevamento intensivo: è una razza autoctona che pascola libera nei boschi di faggio e quercia del parco dei Nebrodi, nutrendosi soprattutto di ghiande e castagne — è proprio per questo un Presidio Slow Food, con capi tenuti volutamente pochi. Le trattorie di paese che lo cucinano davvero lo propongono come salsiccia fresca, capocollo, o il salame che qui chiamano *fellata* — mai come una generica "bistecca di maiale" sul menù plastificato. Chiedere se il suino è certificato Presidio è l'unico modo per sapere che non si sta mangiando un ibrido allevato altrove con lo stesso nome.

La seconda versione non ripete mai "suino nero dei Nebrodi" alla lettera più di una volta, ed è quella che un motore di ricerca moderno è costruito per premiare, perché dimostra davvero comprensione dell'argomento: Presidio Slow Food, semibrado, capocollo, fellata, certificato Presidio. Non sono sinonimi inseriti per aggirare un sistema; sono il vocabolario che qualcuno con reale competenza sull'argomento userebbe naturalmente, e quella concentrazione di terminologia correlata e accurata è di per sé un segnale forte di profondità reale sull'argomento — la cosa che la ricerca moderna sta effettivamente cercando di misurare. (L'allevamento semibrado della razza, il suo status di Presidio Slow Food, e capocollo/fellata come suoi salumi tradizionali sono tutti fatti reali e documentati sui Nebrodi, non colore inventato per questo esempio — vedi le Fonti in fondo.)

## Cosa significa in pratica

Niente di tutto questo significa che le keyword abbiano smesso del tutto di contare — una pagina sul suino nero dei Nebrodi dovrebbe comunque, naturalmente, menzionare "Nebrodi" e "suino nero". Ciò che è cambiato è l'obiettivo: scrivere per coprire in modo completo e accurato un *argomento*, nel vocabolario che un esperto userebbe davvero, invece che colpire una frase specifica un numero fisso di volte. Alcune abitudini concrete che derivano direttamente da tutto questo:

- **Rispondi alla domanda vera, direttamente e presto**, come farebbe a voce una persona competente — non dopo tre paragrafi di premessa.
- **Copri il vocabolario naturale che circonda l'argomento** (Presidio Slow Food, semibrado, capocollo, certificato Presidio — non solo "suino nero Nebrodi" ripetuto), perché quel vocabolario è ciò che dimostra vera profondità sia a un lettore umano che a un sistema di ranking basato su embedding.
- **Struttura il contenuto in modo che ogni sezione risponda a una sotto-domanda chiara** — questo si mappa naturalmente su come un motore di ricerca (e sempre più, un AI Overview) estrae un passaggio specifico e rilevante da una pagina più lunga, invece di dover far corrispondere l'intera pagina all'intera query.

## Fonti

- [A Guide to Google Search Ranking Systems](https://developers.google.com/search/docs/appearance/ranking-systems-guide) — la documentazione ufficiale di Google su RankBrain, BERT, Neural matching e gli altri sistemi citati sopra.
- [Understanding searches better than ever before](https://blog.google/products-and-platforms/products/search/search-language-understanding-bert/) — l'annuncio ufficiale di Google su BERT, 25 ottobre 2019.
- [Google Search's I/O 2026 updates: AI agents and more](https://blog.google/products-and-platforms/products/search/search-io-2026/) — l'annuncio ufficiale di Google che conferma il rollout globale e non più sperimentale dell'AI Mode su Gemini 3.5 Flash.
- [Google Search: A timeline of the 25 biggest moments](https://blog.google/products-and-platforms/products/search/25-biggest-google-search-updates/) — la retrospettiva ufficiale di Google su Hummingbird (2013) e RankBrain (2015).
- [Suino nero dei Nebrodi — Presìdi Slow Food](https://www.fondazioneslowfood.com/it/presidi-slow-food/suino-nero-dei-nebrodi/) — l'allevamento semibrado del suino nero dei Nebrodi, il suo status di Presidio Slow Food, e i salumi tradizionali (capocollo, salsiccia fresca e secca, il salame fellata) usati nell'esempio di scrittura di questo episodio.

## Cosa viene dopo

L'episodio 2 copre l'intento di ricerca — le quattro grandi categorie in cui può rientrare una query (informazionale, navigazionale, transazionale, commerciale) — e perché far combaciare una pagina con l'intento *giusto* conti più che farla combaciare con la keyword *giusta*.
