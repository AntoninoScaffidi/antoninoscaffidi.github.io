---
layout: post
title: "Ruby Deep Dive: Control Flow"
series: "ruby-deep-dive"
episode: 2
lang: en
ref: ruby-control-flow
permalink: /ruby-control-flow/
canonical_url: https://antoninoscaffidi.github.io/ruby-control-flow/
image: /assets/images/ruby-deep-dive-banner.png
date: 2026-08-30 09:00:00 +0200
---

[Episode 1]({% post_url 2026-08-09-ruby-variables-and-basic-types %}) covered variables and the basic types, including Ruby's truthy/falsy rule: only `nil` and `false` are falsy, everything else — including `0` and `""` — is truthy. That rule is the foundation for everything in this episode, because every branch and every loop in Ruby ultimately comes down to "is this truthy or falsy?"

Code is in the [ruby-deep-dive](https://github.com/AntoninoScaffidi/ruby-deep-dive) repo, tagged [`episode-2`](https://github.com/AntoninoScaffidi/ruby-deep-dive/tree/episode-2). Same format as episode 1: read the post, then fill in [`02-control-flow/exercises.rb`](https://github.com/AntoninoScaffidi/ruby-deep-dive/blob/main/02-control-flow/exercises.rb) until [`02-control-flow/exercises_test.rb`](https://github.com/AntoninoScaffidi/ruby-deep-dive/blob/main/02-control-flow/exercises_test.rb) is all green.

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

Nothing surprising here on the surface — but there's one thing worth being precise about: **`if` is an expression, not a statement.** It evaluates to the value of whichever branch ran, and that value is what the method returns (there's no `return` here — the last expression evaluated is the method's return value, another thing that's true throughout Ruby, not specific to `if`). If no branch matches and there's no `else`, the whole `if` evaluates to `nil`:

```ruby
result = if false
  "never happens"
end
result #=> nil
```

This is also why Ruby has a **modifier form** — a one-liner that reads almost like English, used constantly for guard clauses:

```ruby
puts "adult" if age >= 18
return nil unless user
```

Reach for the modifier form when the body is one short line and there's no `elsif`/`else`; reach for the multi-line form the moment there's a second branch or the body needs more than one statement.

## unless

`unless condition` is exactly `if !condition`, nothing more:

```ruby
def bouncer_message(age)
  unless age < 18
    "Come on in!"
  else
    "Sorry, 18+ only."
  end
end
```

`unless ... else` is legal Ruby, but most style guides (and most Rubyists) discourage it, for a readability reason worth internalizing rather than just accepting as a rule: with `unless`, the `else` branch runs when the condition **is** true, which means every reader has to negate the condition in their head before the `else` makes sense. `if cond ... else ...` reads in the order your brain processes it; `unless cond ... else ...` reads backwards. Save `unless` for the case it's actually good at: a single guard condition with no `else` at all — `unless` shines exactly where the modifier form does, e.g. `return unless valid?`.

## The ternary operator

```ruby
def parity_word(n)
  n.even? ? "even" : "odd"
end
```

`condition ? if_true : if_false` is not a separate control-flow construct — it's `if`/`else` as an expression, written on one line. Use it only when both branches are short and there's exactly two of them; a ternary with a nested ternary inside it is a code smell, not a compact style choice. If you need a third branch, that's `case/when` or a full `if/elsif/else`, not a chain of `?:`.

## case/when — and the operator that makes it work

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

This looks like a `switch` statement from other languages, but it's doing something meaningfully different under the hood, and understanding it changes what you reach for `case/when` to do. Each `when` clause doesn't test `hours == 0..1` — it calls **`(0..1) === hours`**, the *case equality* operator, with the `when` value on the left. `Range#===` is defined as "does this range cover the value" (`include?`), which is exactly why ranges work as `when` clauses at all. It's not special-cased syntax for ranges — it's `===` behaving differently depending on what class defines it:

```ruby
(0..1) === 1        #=> true   (Range#===: membership)
String === "hi"      #=> true   (Module#===: is_a?)
/^h/ === "hi"         #=> true  (Regexp#===: match)
42 === 42             #=> true  (Object#===: falls back to ==)
```

That means `case/when` isn't limited to values and ranges — a `when String` branch matches anything that's a `String`, and a `when /^h/` branch matches anything a regex matches. It's a much more general dispatch tool than "switch on a value," once you know what's actually being called.

The other detail worth knowing: a `when` clause can list several candidates separated by commas, matched with OR:

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

`when :saturday, :sunday` runs the branch if `:saturday === day` **or** `:sunday === day` — one branch, several accepted values, no need to repeat the body per day.

And a `case` doesn't need a subject at all. Written without one, each `when` is a full boolean condition, and it's evaluated top to bottom like an `if/elsif` chain — genuinely useful when the conditions aren't all testing the same variable:

```ruby
case
when age < 13 then "kid"
when age < 20 then "teen"
else "adult"
end
```

## while and until

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

`while condition ... end` runs the body for as long as the condition stays truthy, checking it *before* each iteration — including the very first one, so a `while` whose condition starts out false never runs at all (that's exactly why `sum_up_to(0)` returns `0`: `i` starts at `1`, `1 <= 0` is false immediately, the loop body never executes). `until condition` is `while !condition` — same loop, inverted check:

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

Both have a modifier form too, same idea as `if`/`unless`:

```ruby
i += 1 while i < 10
```

Be careful with the modifier form specifically around `begin...end while` — it has a quirk (the body runs once *before* the first check, unlike every other `while`) that's rare enough in practice not to be worth memorizing right now, just worth knowing it exists so it doesn't surprise you later.

## loop and break with a value

`while`/`until` need a condition up front. Sometimes the natural exit point is in the *middle* of the body, not somewhere you can express as a pre-loop condition. That's what `loop` is for:

```ruby
def first_power_of_two_above(limit)
  power = 1
  loop do
    break power if power > limit
    power *= 2
  end
end
```

`loop` is a plain method (`Kernel#loop`) that takes a block and runs it forever, until something inside stops it. `break` stops the loop — and here's the detail that's easy to miss coming from other languages: **`break value` doesn't just exit the loop, it makes the whole `loop do...end` expression evaluate to `value`.** A `while` loop always evaluates to `nil`, no matter how it ends; `loop` evaluates to whatever `break` handed it (or `nil` if it never breaks with a value). That's why `first_power_of_two_above` doesn't need a variable declared outside the loop to "remember" the answer — the loop itself *is* the answer, once you `break` with it.

## next

Inside any loop, `next` skips straight to the next iteration without running the rest of the body:

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

`next if n.odd?` reads as a guard clause at the top of the block: "if this one doesn't qualify, move on" — the same shape as an early `return` inside a method, just for one loop iteration instead of the whole call. (`numbers.each` is technically an `Enumerable` method rather than a control-flow keyword like `while`/`until`/`loop` — `each` and its relatives get a full episode of their own later in this series — but `next`/`break` work inside its block exactly the same way they do inside `while`, `until`, or `loop`.)

## for — and why you'll rarely see it

Ruby does have a `for` loop:

```ruby
for i in 1..3
  puts i
end
```

It's mentioned here mainly so you can recognize it if you run into it, not because you should reach for it. The reason idiomatic Ruby avoids `for` in favor of `each`/`times`/etc. is a scoping difference, and it's worth seeing once directly:

```ruby
for i in 1..3
end
i #=> 3, still accessible here

[1, 2, 3].each do |i|
end
i #=> NameError: undefined local variable or method 'i'
```

`for` doesn't introduce a new scope — the loop variable leaks into whatever scope surrounds it, exactly like a plain `while` loop's counter would. A block passed to `each` *does* introduce its own scope, so `i` inside the block is a fresh local variable that stops existing the moment the block ends. That containment is exactly the behavior you want almost all the time, which is the real reason `each` won out over `for` as Ruby idiom — not "for is deprecated," just "each's scoping is safer by default."

## Truthy and falsy, one level deeper

Episode 1 established the rule: only `nil` and `false` are falsy. Now that `if`/`while`/`unless` are all in place, here's the idiom that rule enables, and that shows up constantly in real Ruby code:

```ruby
name = nil
display_name = name || "Guest"
```

`||` doesn't just produce `true`/`false` — it evaluates its left side, and if that's truthy, **returns that value itself**; only if the left side is falsy does it evaluate and return the right side. `&&` works the same way in reverse: it returns its left side if that's falsy (short-circuiting before ever touching the right side), otherwise returns the right side. That's the actual mechanism behind the extremely common `x || default` pattern — it's not a special "default value" syntax, it's `||` doing exactly what it always does, applied to a case where that behavior happens to be exactly what you want.

One more thing worth flagging now rather than after it bites you: Ruby also has `and`/`or`/`not` as English-word equivalents of `&&`/`||`/`!`. They are **not** interchangeable with the symbol operators, because they bind at a much lower precedence — lower than `=`:

```ruby
x = false or true
x #=> false
```

That reads like it should set `x` to `true`, but `=` binds tighter than `or`, so it's actually parsed as `(x = false) or true` — `x` gets `false`, and the `or true` does nothing useful at all. This is a well-known gotcha, not a stylistic footgun you're unlikely to hit: stick to `&&`/`||`/`!` for actual logic, and treat `and`/`or`/`not` as effectively unused in idiomatic Ruby.

## The exercises

Ten methods in [`02-control-flow/exercises.rb`](https://github.com/AntoninoScaffidi/ruby-deep-dive/blob/main/02-control-flow/exercises.rb), one per concept covered above: `if`/`elsif`/`else`, `unless`, the ternary operator, `case/when` with ranges, `case/when` with comma-separated values, `while`, `until`, `loop`/`break` with a value, `next`, and Ruby's own truthiness via `!!`.

```bash
git clone https://github.com/AntoninoScaffidi/ruby-deep-dive.git
cd ruby-deep-dive
bundle install
ruby 02-control-flow/exercises_test.rb
```

## What's next

Episode 3 covers methods: definitions, arguments (positional, keyword, default, splat), and what a method actually returns.
