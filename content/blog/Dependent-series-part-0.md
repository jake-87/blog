+++
title = "(WIP) Dependent language series, part 0"
date = 2025-05-08
+++

## Assumed knowledge

You should at least be familiar with dependent types, using a language such as Agda, Rocq, Idris, or Lean.
This series will primarily use Agda syntax.

## Motivation (Why learn this, anyway?)

TODO

## Debrujin and why we use it

Let's look at a little imaginary term, in some language.

```hs
λ f. (λ y f. (f y)) f
```

We can perform what's called "beta reduction" on this term — essentially, applying `f` to the lambda.

```hs
λ f. (λ y f. (f y)) (f)

substitute [y := f] in λ y f. (f y)

λ f. (λ f. f f)
```

Uh oh. We've accidentally captured a variable! Instead of `f` referring to the "outer" `f`, now it refers to the "inner" `f`. This is what i'll call "the capture problem", and it is *very* annoying. Generally to avoid this, we need to rename everything in our "substituted" term to names that are free (do not occur) in the "subsitutee", and therefore nothing is accidentally captured. What if we could introduce a notation that completely avoids this?

### Presenting: Debrujin Indexes!

Debrujin indexes are a naming scheme where:

- We use natural numbers to refer to lambdas
- Zero refers to the "most recent" lambda; one refers to the second most recent, etc.

Let's rewrite that term above using this system:

```hs
λ (λ λ (0 1)) 0
```

Here's some ascii art showing what refers to what:

```
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

What Debrujin indexes allow us to do is *simply* avoid capturing. The rule is simple: Every time we go past a binder when substituting, we increment every free variable in our substituted by one, to avoid the new binder:

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

### Presenting: Debrujin levels!

Debrujin levels work similar to debrujin indexes, in that we use numbers to refer to binders. However, in Debrujin levels, the lowest number refers to the *least* recently bound item.

Recall that:
```
Named:   λ f. (λ y f. (f y)) f
Indexes: λ (λ λ (0 1)) 0
```
Now, with levels:

: Levels:  λ (λ λ (2 1)) 0

This has the same diagram of what refers to what:

```
 λ   (λ   λ   (2   1))   0
 |    |    \———/   |     |
 |    |            |     |
 |    \————————————/     |
 \———————————————————————/
```

(As it should! These two representations represent the same term.)

Debrujin indexes gave us the advantage that bound variables in a substituted term need not be interfered with — we simply had to increment free variables.

Debrujin levels have a near-dual advantage. When placing a term using levels under a binder, no shifting needs to take place in said term.


: λ (λ λ (2 1)) 0
: ->
: λ (λ (1 0))
:       ^ ^ this is still zero!
:       |
:       \ we had to modify this one though

### Wrapup

Debrujin indexes and levels can also be summed up via the following:

```
λa. λb. λc. c

indexes:
λ λ λ 0
^ ^ ^
2 1 0

levels:
λ λ λ 2
^ ^ ^
0 1 2

if you’re “here”
  v
λ λ λ

Using indexes, adding anything further left doesn’t affect that binder, or any further right.

Using levels, adding anything further right doesn’t affect that binder, or any further left.
```

Generally, Debrujin indexes are "more useful" than Debrujin levels, as they're "more local". In order to work with levels, you need to know "how deep" you are in a term at all times. However, the property of adding things not affecting binders further left is very handy in some cases! This will be explored later in the series, but that concludes this exploration for right now.


## Evaluation while typechecking

Complex dependent types require computation while typechecking. Consider the following:

```hs
neg-or-not : (b : Bool) -> if b then (Int -> Int) else (Bool -> Bool)
neg-or-not true x = -x
neg-or-not false x = not x
```

When we wish to typecheck `neg-or-not true x`, we *need* to evaluate in the type of `neg-or-not` in order to be able to tell what the type of `x` *should* be. This means our typechecker is going to have evaluation in it! Unlike with "normal" evaluation, we wish to control this evaluation as much as possible. We do not want to be fully computing every term, because that's slow and unnecessary. There are several evaluation techniques that achieve various levels of this — the one we will be exploring is called "Normalization by Evaluation" (NbE). As the title alludes to, the goal of NbE is to normalize (elaborated on later) a term to some desired "normal form" using evaluation, and then stop evaluating at that normal form. Evaluating to a normal form guarantees that two elements, both in this normal form, can be immediately compared for shallow equality without "excess" work. (As is often the goal — in the example above, we wish to compare the type of `x` with the type of `neg-or-not true`.).

The core ideas of NbE are as follows:

– Have two representations of your program: A "term language" and a "value language".
– The term language represents any given term, with no restrictions (except being type correct).
– The value language represents only terms in a normal form, correct-by-construction. (It is only possible to form elements of the value type if they are in a normal form.)
– The term language uses Debrujin indexes, as they are more suitable for direct translation from user input, and do not rely on knowing "nonlocal information" in order to create new binders.
— The value language uses Debrujin levels, so that values can be placed under binders without the need for shifting.
- We have a function, called "evalutate" or "eval", that takes expressions in the term language to the value language (i.e., reduces them to normal form through evaluation)
– We have a function, called "quote", that takes expressions in the value language back to the term language.

Using NbE as a basis, we can then build a typechecker for a dependent language! (using a style called bidirectional typechecking, which will be explained later).

