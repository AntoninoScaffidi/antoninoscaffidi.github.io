---
layout: post
title: "Ruby Deep Dive: Methods"
series: "ruby-deep-dive"
episode: 3
lang: en
ref: ruby-methods
permalink: /ruby-methods/
canonical_url: https://antoninoscaffidi.github.io/ruby-methods/
image: /assets/images/ruby-deep-dive-ep3-banner.png
date: 2026-09-01 09:00:00 +0200
---

[Episode 2]({% post_url 2026-08-30-ruby-control-flow %}) covered `if`, `case`, and loops — the constructs that decide *what* runs. This episode is about the thing that actually gets decided to run: methods. Not classes yet (that's episode 4 territory) — just a standalone method, its arguments, and what it hands back to whoever called it.

Code is in the [ruby-deep-dive](https://github.com/AntoninoScaffidi/ruby-deep-dive) repo, tagged [`episode-3`](https://github.com/AntoninoScaffidi/ruby-deep-dive/tree/episode-3). Same format as always: read the post, then fill in [`03-methods/exercises.rb`](https://github.com/AntoninoScaffidi/ruby-deep-dive/blob/main/03-methods/exercises.rb) until [`03-methods/exercises_test.rb`](https://github.com/AntoninoScaffidi/ruby-deep-dive/blob/main/03-methods/exercises_test.rb) is all green.

## Defining a method

```ruby
def add(a, b)
  a + b
end
```

`def`, a name, parentheses around the parameters (optional, but worth always writing), a body, `end`. No type declarations anywhere — not on `a`, not on `b`, not on what `add` gives back. That's the same "a name is just a label, not a box with a declared type" idea from episode 1, applied to method parameters: `a` and `b` are just names that get bound to whatever objects were passed in, of whatever type they happen to be.

## The return value: implicit and explicit

`add` has no `return` keyword, and it still returns `a + b`. **A Ruby method always returns the value of the last expression evaluated in its body** — this is the same rule episode 2 established for `if`, extended to the whole method. There's nothing special about `return` making a method "return something" — every method returns something, whether you write the keyword or not.

`return` is for the case where you need to hand a value back *before* reaching the end of the method — an early exit:

```ruby
def safe_divide(a, b)
  return nil if b.zero?
  a / b.to_f
end
```

Read this as: "if `b` is zero, stop right here and return `nil`; otherwise, keep going, and whatever the last line evaluates to is the answer." The `return` on line one is explicit and necessary — without it, `nil` would just be a value sitting in the middle of the method, discarded. The `a / b.to_f` on the last line doesn't need `return` at all; it's already exactly where the implicit return rule looks for a value. Idiomatic Ruby leans on this constantly: reach for explicit `return` for early exits and guard clauses, and let the last line do the returning everywhere else. A `return` on the very last line of a method is legal but redundant — you'll rarely see it in Ruby code.

## Positional and default arguments

`add(a, b)` and `greet(name)` both take **positional** arguments — the caller's first value fills the first parameter, second value fills the second, matched purely by position, not by name. Give a parameter a default, and the caller can leave it out:

```ruby
def greet(name, greeting = "Hello")
  "#{greeting}, #{name}!"
end

greet("Ada")        #=> "Hello, Ada!"
greet("Ada", "Hi")  #=> "Hi, Ada!"
```

`greeting = "Hello"` doesn't run at method-definition time — it runs *each time the method is called without that argument*, so the default expression can even reference earlier parameters or call other methods; it's evaluated fresh per call, not computed once and reused.

## Keyword arguments

Positional arguments have a real weakness once a method takes more than two or three of them: at the call site, `describe_person("Ada", 36)` tells you nothing about which number is which without going and checking the method definition. Keyword arguments fix that by naming the value right where it's passed:

```ruby
def describe_person(name:, age: 0)
  "#{name} (#{age})"
end

describe_person(name: "Ada")            #=> "Ada (0)"
describe_person(name: "Ada", age: 36)   #=> "Ada (36)"
describe_person(age: 36, name: "Ada")   #=> "Ada (36)" -- order doesn't matter
```

`name:` with nothing after the colon in the parameter list means "required — calling this method without a `name:` argument is an `ArgumentError`, not a `nil`." `age: 0` means "optional, defaulting to `0`" — exactly the same default-value idea as positional arguments, just spelled with a keyword. Two independent choices — positional vs. keyword, and required vs. defaulted — that combine freely. Reach for keyword arguments the moment a method's parameter list is more than one or two values, or when two parameters are the same type and could be silently swapped by a caller (two integers, two strings) — the keyword makes that mistake impossible to write by accident.

## Splat: catching the rest

Sometimes a method genuinely doesn't know in advance how many arguments it'll get. `*` in a parameter list — the **splat** — collects every remaining positional argument into an `Array`:

```ruby
def sum_all(*numbers)
  numbers.sum
end

sum_all(1, 2, 3)  #=> 6
sum_all()         #=> 0
sum_all(5)        #=> 5
```

Inside the method, `numbers` is just a plain `Array`, whether the caller passed zero, one, or fifty arguments — there's no special "variadic arguments" type to learn, splat just packs them into the same `Array` you already know from episode 1's Tier-1 material. The double-splat, `**`, does the equivalent thing for keyword arguments — it collects every keyword argument the caller passed into a `Hash`:

```ruby
def build_profile(**attributes)
  attributes
end

build_profile(name: "Ada", age: 36)  #=> { name: "Ada", age: 36 }
```

`*` and `**` aren't just for defining flexible methods — they work in reverse too, at the *call site*, to expand a collection you already have into separate arguments: `sum_all(*[1, 2, 3])` is exactly the same call as `sum_all(1, 2, 3)`.

## Splat on the left: destructuring assignment

The same `*` shows up somewhere that isn't a method definition at all — plain assignment:

```ruby
first, *rest = [1, 2, 3, 4]
first  #=> 1
rest   #=> [2, 3, 4]
```

This is called destructuring (or "multiple") assignment: the right-hand side's elements get distributed across the names on the left, and `*rest` soaks up everything that's left over into an `Array` — empty if there's nothing left, never `nil`:

```ruby
def first_and_rest(array)
  first, *rest = array
  [first, rest]
end

first_and_rest([1, 2, 3])  #=> [1, [2, 3]]
first_and_rest([5])        #=> [5, []]
first_and_rest([])         #=> [nil, []]
```

`first_and_rest([])` gives `[nil, []]`, not an error — destructuring an empty array just leaves `first` unassigned (`nil`), the same way any local variable is `nil` before it's ever set.

## Naming conventions: `?` and `!`

Two suffixes are conventions Ruby enforces nowhere in the language itself, but that every Rubyist relies on being followed:

```ruby
def palindrome?(str)
  cleaned = str.downcase
  cleaned == cleaned.reverse
end
```

A method ending in `?` is a promise, not a rule the interpreter checks: **this method returns a boolean, and doesn't change anything.** `empty?`, `nil?`, `even?` — all built-in methods that follow the same convention. Nothing stops you from writing `def valid?; @errors = []; true; end`, but doing that violates a convention every reader of your code is relying on, silently, to be true.

```ruby
def reverse_in_place!(array)
  array.reverse!
end
```

A method ending in `!` is the other promise: **this method mutates something the caller already had a reference to**, rather than building and returning something new. This connects directly back to episode 1's "variable is a label, not a box" idea — `array.reverse!` doesn't create a new, reversed array and hand it back; it reverses the *same* `Array` object in place. The caller's original variable sees the change too, because it was always pointing at that one object:

```ruby
array = [1, 2, 3]
result = reverse_in_place!(array)
result.equal?(array)  #=> true -- literally the same object, not just equal values
```

`!` doesn't technically mean "dangerous" or "mutates" by any rule the language enforces (`Kernel#exit!` doesn't mutate anything, for instance) — the more accurate one-line version is "this method does something you should look twice at before calling," and "mutates its argument or receiver in place" is by far the most common reason for that. The rule of thumb that actually holds: when a method has both a plain and a `!` version (`sort`/`sort!`, `reverse`/`reverse!`, `upcase`/`upcase!`), the plain one returns a new object and leaves the original alone, and the `!` one changes the original object directly.

## The exercises

Nine methods in [`03-methods/exercises.rb`](https://github.com/AntoninoScaffidi/ruby-deep-dive/blob/main/03-methods/exercises.rb): implicit return, explicit early `return`, a default argument, required and defaulted keyword arguments, a splat that collects positional arguments, a double-splat that collects keyword arguments into a `Hash`, a `?` predicate method, a `!` mutating method, and destructuring assignment with a splat.

```bash
git clone https://github.com/AntoninoScaffidi/ruby-deep-dive.git
cd ruby-deep-dive
bundle install
ruby 03-methods/exercises_test.rb
```

## What's next

Episode 4 moves from standalone methods to classes and objects: `initialize`, instance variables, and `attr_accessor` — the point where "a method" starts becoming "a method that belongs to something."
