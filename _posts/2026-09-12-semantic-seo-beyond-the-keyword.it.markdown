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

Per gran parte della storia del web, posizionarsi bene nella ricerca ha significato soprattutto una cosa: mettere le parole giuste nella pagina, nei posti giusti, il numero giusto di volte. Se qualcuno cercava "migliori scarpe da running per piede piatto", la strategia vincente era assicurarsi che quella frase esatta — o varianti molto simili — comparisse nel titolo, nei sottotitoli, qualche volta nel corpo del testo, magari nel testo alternativo delle immagini. I motori di ricerca, nel loro nucleo, facevano string matching: questo documento contiene questi caratteri, più o meno in questo ordine? Più precisamente una pagina rifletteva la query, più "meritava" di posizionarsi.

Questo ha prodotto una scrittura genuinamente strana. Le pagine ripetevano una frase cinque o sei volte in tre paragrafi, in un modo che nessun essere umano scriverebbe mai davvero, puramente perché l'algoritmo contava le occorrenze. Quell'epoca è ormai quasi finita, e vale la pena capire esattamente cosa l'ha sostituita — non come curiosità SEO, ma perché cambia cosa significhi davvero "scrivere bene per la ricerca".

## Cosa è cambiato davvero

**Hummingbird**, lanciato nel 2013, è stato il primo grande cambiamento: invece di analizzare una query come un insieme di keyword separate, Google ha iniziato a cercare di capire la query *nel suo insieme* — la relazione tra le parole, non solo la loro presenza. "Qual è il posto più vicino per comprare scarpe da running vicino al Golden Gate Bridge" non sono cinque keyword indipendenti da far combaciare; è una domanda unica con un intento specifico, e Hummingbird è stato costruito per trattarla così.

**RankBrain**, aggiunto all'algoritmo principale nel 2015, ha portato il machine learning direttamente nel ranking — in particolare per gestire il circa 15% delle ricerche giornaliere che Google non aveva mai visto prima. Per una query genuinamente nuova, non esiste una cronologia di "cosa hanno cliccato le persone per questa frase esatta" da cui imparare. Il compito di RankBrain era mappare una query sconosciuta sui *concetti* dietro query simili e familiari, e posizionare di conseguenza — un passaggio esplicito dal far combaciare il testo al modellare il significato.

**BERT**, nel 2019, è andato ancora più a fondo nei meccanismi del linguaggio stesso. I sistemi precedenti leggevano perlopiù una query come un insieme approssimativo di parole importanti, tendendo a ignorare come rumore piccole parole connettive come "a", "per" o "non". BERT legge le parole *in relazione alle parole intorno a loro*, in entrambe le direzioni contemporaneamente — ed è esattamente per questo che una preposizione può ribaltare il significato di un'intera frase, e ora ribalta anche i risultati. "Cittadino brasiliano che viaggia negli USA nel 2019 ha bisogno del visto" significa qualcosa di diverso da "cittadino USA che viaggia in Brasile nel 2019 ha bisogno del visto" — stesse parole, ordine diverso, intento opposto — ed è esattamente il tipo di distinzione che il vecchio keyword matching non era costruito per cogliere.

Più di recente, gli **AI Overview** (costruiti su modelli della famiglia MUM/Gemini) vanno ancora oltre: per molte query, la pagina dei risultati ora include un riassunto generato dall'AI e sintetizzato *da più fonti insieme*, prima ancora che compaia un singolo link blu. Il motore non si limita più a posizionare pagine che corrispondono a una query — ne legge diverse, estrae ciò che è rilevante per la domanda specifica, e scrive una risposta. Posizionarsi bene significa sempre di più essere una delle fonti giudicate degne di essere sintetizzate, non solo uno dei dieci link blu.

## Cosa significa davvero "semantico", concretamente

Messo insieme, ecco il cambiamento pratico: i motori di ricerca rappresentano sempre di più sia le query che i contenuti come **significato**, non come sequenze di caratteri. Sotto il cofano, questo tipicamente comporta convertire il testo in un **embedding** — una rappresentazione numerica posizionata in uno spazio dove testi che parlano di cose simili finiscono vicini tra loro, indipendentemente dal fatto che condividano le stesse parole. "Scarpe da running economiche per piede largo" e "sneaker da corsa a basso costo per piede largo" possono finire vicine in quello spazio, perché *parlano* della stessa cosa, anche con quasi nessuna sovrapposizione di vocabolario. (La [serie ai-with-ruby]({% post_url 2026-08-22-semantic-search-with-pgvector %}) di questo blog costruisce esattamente questo tipo di sistema, se vuoi vedere il meccanismo dall'interno.)

È anche così che un motore di ricerca riconosce le **entità** — una persona, un luogo, un prodotto o un concetto specifico — e collega una pagina a ciò di cui parla davvero, invece che alle parole che per caso compaiono su di essa. Una pagina può posizionarsi per "consigli scarpe da running piede piatto" senza mai usare quella frase esatta, purché parli davvero, e in modo approfondito, di quell'argomento.

## Vedere la differenza

Una scrittura vecchio stile, guidata dalle keyword, mirata a "scarpe da running per piede piatto", potrebbe assomigliare a questo:

> Cerchi le migliori scarpe da running per piede piatto? Le nostre scarpe da running per piede piatto sono progettate per chi ha il piede piatto e cerca scarpe da running per piede piatto. Queste scarpe da running per piede piatto offrono un ottimo supporto.

Ogni frase esiste per ripetere la frase, non per dire qualcosa di nuovo. Una versione scritta in modo semantico sullo stesso argomento si legge come una persona che conosce davvero l'argomento:

> Il piede piatto — o un arco plantare completamente collassato — cambia come il peso atterra a ogni passo, ed è per questo che un'ammortizzazione generica spesso non basta. Cerca scarpe con stabilità, supporto rinforzato dell'arco plantare e una suola intermedia più rigida; i modelli a controllo del movimento aiutano ulteriormente se anche l'appoggio tende a ruotare verso l'interno (overpronazione).

La seconda versione non ripete mai "scarpe da running per piede piatto" alla lettera, nemmeno una volta — ed è quella che un motore di ricerca moderno è costruito per premiare, perché dimostra davvero comprensione dell'argomento: collasso dell'arco, scarpe con stabilità, supporto dell'arco, controllo del movimento, overpronazione. Non sono sinonimi inseriti per aggirare un sistema; sono il vocabolario che qualcuno con reale competenza sull'argomento userebbe naturalmente, e quella concentrazione di terminologia correlata e accurata è di per sé un segnale forte di profondità reale sull'argomento — la cosa che la ricerca moderna sta effettivamente cercando di misurare.

## Cosa significa in pratica

Niente di tutto questo significa che le keyword abbiano smesso del tutto di contare — una pagina sulle scarpe da running per piede piatto dovrebbe comunque, naturalmente, menzionare "piede piatto" e "scarpe da running". Ciò che è cambiato è l'obiettivo: scrivere per coprire in modo completo e accurato un *argomento*, nel vocabolario che un esperto userebbe davvero, invece che colpire una frase specifica un numero fisso di volte. Alcune abitudini concrete che derivano direttamente da tutto questo:

- **Rispondi alla domanda vera, direttamente e presto**, come farebbe a voce una persona competente — non dopo tre paragrafi di premessa.
- **Copri il vocabolario naturale che circonda l'argomento** (supporto dell'arco, scarpe con stabilità, overpronazione — non solo "scarpe running piede piatto" ripetuto), perché quel vocabolario è ciò che dimostra vera profondità sia a un lettore umano che a un sistema di ranking basato su embedding.
- **Struttura il contenuto in modo che ogni sezione risponda a una sotto-domanda chiara** — questo si mappa naturalmente su come un motore di ricerca (e sempre più, un AI Overview) estrae un passaggio specifico e rilevante da una pagina più lunga, invece di dover far corrispondere l'intera pagina all'intera query.

## Cosa viene dopo

L'episodio 2 copre l'intento di ricerca — le quattro grandi categorie in cui può rientrare una query (informazionale, navigazionale, transazionale, commerciale) — e perché far combaciare una pagina con l'intento *giusto* conti più che farla combaciare con la keyword *giusta*.
