---
layout: post
title: "Semantic SEO: Beyond the Keyword"
series: "semantic-seo"
episode: 1
lang: en
ref: semantic-seo-beyond-the-keyword
permalink: /semantic-seo-beyond-the-keyword/
canonical_url: https://antoninoscaffidi.github.io/semantic-seo-beyond-the-keyword/
image: /assets/images/semantic-seo-ep1-banner.png
date: 2026-09-12 09:00:00 +0200
---

This is the first episode of **Semantic SEO**, a new series on this blog — announced [a couple of weeks ago]({% post_url 2026-08-30-two-new-series-coming %}) alongside its technical counterpart, [Semantic Search in Ruby on Rails]({% post_url 2026-08-30-two-new-series-coming %}). Unlike most of what's here, this one isn't about Rails or code at all. It's for anyone who writes online, runs a website, or has ever wondered why some pages rank and others — seemingly stuffed with all the "right" words — don't.

## The old way: matching strings

For most of the web's history, ranking well in search meant one thing above all: getting the right words on the page, in the right places, the right number of times. If someone searched "local food tour in Florence," the winning strategy was to make sure that exact phrase — or close variants of it — appeared in your title, your headings, a few times in the body, maybe the image alt text. Search engines, at their core, were doing string matching: does this document contain these characters, in roughly this arrangement? The more precisely a page echoed the query back, the more it "deserved" to rank.

This produced some genuinely strange writing. Pages would repeat a phrase five or six times in three paragraphs, in ways no human would ever actually write, purely because the algorithm was counting occurrences. That era is mostly over, and it's worth understanding exactly what replaced it — not as SEO trivia, but because it changes what "writing well for search" actually means.

## What actually changed

**Hummingbird**, rolled out in 2013, was the first major shift: instead of parsing a query as a bag of separate keywords, Google started trying to understand the query *as a whole* — the relationship between the words, not just their presence. "What's a good place to eat like a local near the Duomo in Florence" isn't five independent keywords to match; it's one question with a specific intent, and Hummingbird was built to treat it that way.

**RankBrain**, added to the core algorithm in 2015, brought machine learning into ranking directly — specifically to handle the roughly 15% of daily searches Google had never seen before. For a genuinely novel query, there's no history of "what people clicked on for this exact phrase" to learn from. RankBrain's job was to map an unfamiliar query onto the *concepts* behind similar, familiar ones, and rank accordingly — an explicit move from matching text to modeling meaning.

**BERT**, in 2019, went further into the mechanics of language itself. Earlier systems mostly read a query as a loose collection of important words, tending to ignore small connective words like "to," "for," or "not" as noise. BERT reads words *in relation to the words around them*, in both directions at once — which is exactly why a preposition can flip the meaning of a whole sentence, and now flips the results too. "2019 brazil traveler to usa need a visa" means something different from "2019 usa traveler to brazil need a visa" — same words, different order, opposite intent — and that's precisely the kind of distinction earlier keyword matching wasn't built to catch.

More recently, **AI Overviews** (built on models in the MUM/Gemini family) go a step further still: for many queries, the search results page now includes an AI-generated summary synthesized *across multiple sources*, before a single blue link appears. The engine isn't just ranking pages that match a query anymore — it's reading several of them, extracting what's relevant to the specific question, and writing an answer. Ranking well increasingly means being one of the sources judged worth synthesizing from, not just one of the ten blue links.

## What "semantic" actually means, concretely

Put together, here's the practical shift: search engines increasingly represent both queries and content as **meaning**, not as strings of characters. Under the hood, this typically involves converting text into an **embedding** — a numeric representation positioned in a space where texts about similar things end up near each other, regardless of whether they share any of the same words. "Hidden gem food spots in Florence only locals know" and "where Florentines actually eat, not the tourist strip" can land close together in that space, because they're *about* the same thing, even with almost no vocabulary overlap. (This blog's own [ai-with-ruby series]({% post_url 2026-08-22-semantic-search-with-pgvector %}) builds exactly this kind of system, if you want to see the mechanism from the inside.)

That's also how a search engine recognizes **entities** — a specific person, place, product, or concept — and connects a page to what it's actually about, rather than to whichever words happen to appear on it. A page can rank for "authentic Florence food tour recommendations" without ever using that exact five-word phrase, as long as it's genuinely, thoroughly about that topic.

## Seeing the difference

Old-style, keyword-driven writing, aimed at "local food tour in Florence," might look like this:

> Looking for a local food tour in Florence? Our local food tour in Florence is the best local food tour in Florence for people who want a local food tour in Florence. Book our local food tour in Florence today!

Every sentence exists to repeat the phrase, not to say anything new. A semantically-written version covering the same topic reads like a person who actually knows the subject:

> Skip the tourist-trap trattorias lining Via dei Calzaiuoli. This walk starts at the Sant'Ambrogio market, still shopped daily by Florentine households, and works through five family-run stops: a lampredotto cart that's held the same corner since the 1980s, a wine bar pouring only Chianti from small producers within 20km, and a pastry counter where the ricciarelli come out of the oven each morning. Small groups, a Florence-born guide, no bus, no headset.

The second version never repeats "local food tour in Florence" verbatim, not even once — and it's the one a modern search engine is built to reward, because it actually demonstrates understanding of the topic: Sant'Ambrogio market, family-run, lampredotto, small producers, a local guide. Those aren't synonyms inserted to game a system; they're the vocabulary someone with real expertise on the subject would naturally use, and that concentration of related, accurate terminology is itself a strong signal of genuine topical depth — the thing modern search is actually trying to measure.

## What this means in practice

None of this means keywords stopped mattering entirely — a page about a Florence food tour should still, naturally, mention "Florence" and "food tour." What changed is the target: write to comprehensively and accurately cover a *topic*, in the vocabulary an expert on it would actually use, rather than to hit a specific phrase a fixed number of times. A few concrete habits that follow directly from everything above:

- **Answer the actual question, directly and early**, the way a knowledgeable person would answer it out loud — not after three paragraphs of preamble.
- **Cover the topic's natural surrounding vocabulary** (family-run, neighborhood market, small producer, local guide — not just "local food tour Florence" repeated), because that vocabulary is what demonstrates real depth to both a human reader and an embedding-based ranking system.
- **Structure content so each section answers one clear sub-question** — this maps naturally onto how a search engine (and increasingly, an AI Overview) extracts a specific relevant passage from a longer page, rather than needing the whole page to match a whole query.

## What's next

Episode 2 covers search intent — the four broad categories a query can fall into (informational, navigational, transactional, commercial) — and why matching a page to the *right* intent matters more than matching it to the *right* keyword.
