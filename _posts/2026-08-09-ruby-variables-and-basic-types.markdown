---
layout: post
title: "Ruby Deep Dive: Variables and Basic Types"
series: "ruby-deep-dive"
episode: 1
lang: en
ref: ruby-variables-and-basic-types
permalink: /ruby-variables-and-basic-types/
canonical_url: https://antoninoscaffidi.github.io/ruby-variables-and-basic-types/
image: /assets/images/ruby-deep-dive-banner.png
date: 2026-08-09 09:00:00 +0200
---

This is the first episode of **Ruby Deep Dive**, a series that's a bit different from the other two on this blog. VicinoTe and AI with Ruby are both about *building things*. This one is about *understanding the language itself* — Ruby, on its own, no Rails, no framework. It doesn't run on a schedule; it gets written whenever there's time to go through a concept properly.

Every episode comes with exercises in a companion repo: [ruby-deep-dive](https://github.com/AntoninoScaffidi/ruby-deep-dive), tagged [`episode-1`](https://github.com/AntoninoScaffidi/ruby-deep-dive/tree/episode-1). Each one is a method stub to fill in, checked by a test. Read the episode, then go make the tests pass yourself — the exercises are deliberately left unsolved in the repo, because doing them is the point.

We're starting at the real beginning: what a variable is, and Ruby's basic types.

## What a variable actually is

```ruby
language = "Ruby"
```

This line does two things: it creates a `String` object holding the text `"Ruby"`, and it makes the name `language` point to it. That's worth being precise about, because "variable" can be a misleading word if you're coming from a language where variables are boxes that hold values directly. In Ruby, a variable is a **label**, and what it labels is an **object** living somewhere in memory. Two variables can point to the very same object:

```ruby
a = "Ruby"
b = a
b.upcase!
a #=> "RUBY"
```

Changing `b` changed `a` too, because they were never two separate strings — they were two labels on the same one. `upcase!` (with the `!`) mutates the string in place, rather than returning a new one. This distinction — is a variable a label or a box — is the root of a lot of confusion later on with method arguments and mutation, so it's worth having clearly in mind from episode 1.

Ruby doesn't require you to declare a variable's type. `language` isn't "a string variable" — it's just a name currently pointing at a `String`. Assign it something else, and it points at that instead:

```ruby
language = "Ruby"
language = 42
```

Both lines are completely legal, one after the other. The variable didn't change type — it just points somewhere new.

## Local variable naming

A local variable name starts with a lowercase letter or an underscore, and by convention uses `snake_case`:

```ruby
favorite_language = "Ruby"
_unused = "starts with an underscore, often used to mark 'I know I'm not using this'"
```

Ruby cares about this convention more than most languages: `CamelCase` names are reserved for constants and class/module names, so `FavoriteLanguage = "Ruby"` doesn't create a local variable — it creates a *constant*, a different kind of thing entirely, one Ruby will warn you about if you reassign it.

## The basic types

### String

Text, in double or single quotes. The difference matters: double-quoted strings support **interpolation** and escape sequences, single-quoted ones don't.

```ruby
name = "Antonino"
"Hello, #{name}!"   #=> "Hello, Antonino!"
'Hello, #{name}!'   #=> "Hello, #{name}!" (literally, no interpolation)
```

Interpolation — `#{...}` inside a double-quoted string — evaluates whatever Ruby expression is inside the braces and inserts its string form. It's the idiomatic way to build strings in Ruby; reach for it instead of `+` concatenation.

### Symbol

```ruby
:ruby
```

A `Symbol` looks like a string with a colon in front, and it's tempting to think of it as "a string that can't change." That's close, but the more useful way to think about it: a `Symbol` is a **name**, used as an identifier, not as data to display or manipulate. The same symbol written twice is always the exact same object in memory:

```ruby
:ruby.object_id == :ruby.object_id  #=> true
"ruby".object_id == "ruby".object_id #=> false
```

That's why symbols show up constantly as hash keys and method names in Ruby code — they're cheap to compare (Ruby just checks if it's the same object, not character-by-character) and cheap to store (no duplicate copies). The rule of thumb: if the value is something a human will read on screen, it's probably a `String`. If it's an internal label the program uses to refer to something, it's probably a `Symbol`.

### Integer and Float

```ruby
42        # Integer
3.14      # Float
```

One thing that surprises people coming from other languages: integer division truncates, it doesn't round or raise an error.

```ruby
7 / 2     #=> 3, not 3.5
7 / 2.0   #=> 3.5
7.0 / 2   #=> 3.5
```

If either operand is a `Float`, the result is a `Float`. If both are `Integer`, you get integer division. This is exactly why `add_as_float` in this episode's exercises asks you to convert before adding — `2 + 3` is `5`, an `Integer`, no matter how you write the method.

### nil, true, and false

`nil` represents the absence of a value — not zero, not an empty string, genuinely *nothing*. `true` and `false` are the only two values of a separate type, `TrueClass` and `FalseClass` respectively (yes, `true` and `false` are each the sole instance of their own class — more on why that's not as strange as it sounds when we get to "everything is an object" in episode 5).

Here's the detail that trips up almost everyone arriving from another language: **in Ruby, only `nil` and `false` are falsy.** Everything else is truthy — including `0`, including `""` (empty string), including empty arrays and hashes.

```ruby
if 0
  puts "this runs"   # it does! 0 is truthy in Ruby
end

if ""
  puts "so does this"  # also runs
end
```

Coming from JavaScript, Python, or PHP, where `0` and `""` are falsy, this is the single most common source of "why did my `if` do that" confusion. Keep it in mind and it'll stop surprising you fast.

## The exercises

Head to [`01-variables-and-basic-types/exercises.rb`](https://github.com/AntoninoScaffidi/ruby-deep-dive/blob/main/01-variables-and-basic-types/exercises.rb) in the repo. Seven methods, all empty, all covering something from this post: assignment, interpolation, string-to-integer conversion, symbols, `nil` checking, forcing a `Float` result, and — deliberately — finding "the other falsy value" without it being named for you directly in the exercise.

```bash
git clone https://github.com/AntoninoScaffidi/ruby-deep-dive.git
cd ruby-deep-dive
bundle install
ruby 01-variables-and-basic-types/exercises_test.rb
```

Every test will fail until you fill in the corresponding method. That's the intended starting state — turn them green one at a time.

## What's next

Episode 2 covers control flow: `if`/`unless`/`case`, loops, and a closer look at truthy/falsy now that the basic types are in place.
