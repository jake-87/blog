+++
title = "Debruijn indexes + levels, and why they're handy"
date = 2025-05-26
+++

# De Bruijn and why we use it

Let's look at a little imaginary term, in some lambda-calculus-like language. For future note, we call the lambda a "binder", as it binds a variable. There are other types of binders, e.g. `let`, but we will only consider lambdas for the moment. 

```hs
λ f. (λ y f. (f y)) f
```

We can perform what's called "beta reduction" on this term — essentially, applying `f` to the lambda.

```hs
λ f. (λ y f. (f y)) (f)

substitute [y := f] in λ y f. (f y)

λ f. (λ f. f f)
```

Uh oh. We've accidentally captured a variable! Instead of `f` referring to the outer `f`, now it refers to the inner `f`. This is "the capture problem", and it is quite annoying. Generally to avoid this, we need to rename _everything_[^1] in our "substituted" term to names that are free (do not occur) in the "subsitutee" so nothing is accidentally captured. What if we could introduce a notation that avoids this?

## Presenting: De Bruijn Indexes!

De Bruijn indexes are a naming scheme where:

- We use natural numbers to refer to lambdas, and
- Zero refers to the "most recent" lambda; one refers to the second most recent, etc.

Let's rewrite that term above using this system:

```hs
λ (λ λ (0 1)) 0
```

Here's some ascii art showing what refers to what:

```hs
 λ   (λ   λ   (0   1))   0
 |    |    \———/   |     |
 |    |            |     |
 |    \————————————/     |
 \———————————————————————/
```

Now, how does this help us with our substituion problem? Surely if we naively subtitute we will still have binding issues - and indeed we do:

```hs
λ (λ λ (0 1)) 0
->
λ (λ (0 0))
```
No good!

What De Bruijn indexes allow us to do is simply avoid capturing. The rule is simple: Every time we go past a binder when substituting, we increment every free variable[^2] in our substituted term by one, to avoid the new binder:

```hs
 λ (λ λ (0 1)) 0
 ->
 λ (λ (0 1))
 ^  ^    ^ incremented by one when we passed "through" lambda b
 |  |
 |  \ lambda b
 |
 \ lambda a
```

Now we're cool! Everything works as expected, and it takes much less work (and is much more predictable!).

## Presenting: De Bruijn levels!

De Bruijn levels work similar to De Bruijn indexes, in that we use numbers to refer to binders. However, in De Bruijn levels, the lowest number refers to the *least* recently bound item.

Recall that:
```hs
Named:   λ f. (λ y f. (f y)) f
Indexes: λ (λ λ (0 1)) 0
```
Now, with levels:
```hs
Levels:  λ (λ λ (2 1)) 0
```
This has the same diagram of what refers to what:

```hs
 λ   (λ   λ   (2   1))   0
 |    |    \———/   |     |
 |    |            |     |
 |    \————————————/     |
 \———————————————————————/
```

(As it should! These two representations represent the same term.)

As you might expect, de Bruijn indexes and levels are each beneficial in their own situations.

Generally, De Bruijn indexes are "more useful" than De Bruijn levels, as they're "more local". In order to work with levels, you need to know "how deep" you are in a term at all times.

De Bruijn indexes give us the advantage that we can freely create new binders without the need for any information about where in a term we are, whereas de Bruijn levels give us the advantage when moving a term under a binder, free variables do not need to be modified.

```hs
 λ (λ λ (2 1)) 0
               ^ this zero...
 ->
 λ (λ (1 0))
       ^ ^ is still zero!
       |
       \ we had to modify this one though
```
## Wrapup

De Bruijn indexes and levels can also be summed up via the following:

```hs
λa. λb. λc. c

indexes:
λ λ λ 0
^ ^ ^
2 1 0

levels:
λ λ λ 2
^ ^ ^
0 1 2
```
if you’re “here”
```
  v
λ λ λ
```
- Using indexes, adding any binders further left doesn’t affect the current binder's variables, or any further right.
- Using levels, adding any binders further right doesn’t affect the current binder's variables, or any further left.

[^1]: Technically, this is not actually needed. It is sufficient to keep track of everything and rename only names that would overlap. See <a href="https://www.microsoft.com/en-us/research/wp-content/uploads/2002/07/inline.pdf">here</a> for more.

[^2]: A free variable is one that is not bound by the expression we are currently considering. For example, in `λ x. f x`, `f` is free, but `x` is not.
