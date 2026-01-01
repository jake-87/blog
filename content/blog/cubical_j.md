+++
title = "Axiom J in Homotopy Type Theory (HoTT)"
date = 2026-01-01
+++

## Preface

Happy new years!

### Notation

`∀` may be used in place of a Pi-type in some cases. A nested `Π (a b : A). ...` (or sim. for `Σ`) represents `Π (a : A). Π (b : A). ...`.

### J

Axiom J is the following:
```
Given:
A : Type
x y : A
P : Π (a b : A). a = b -> Type
with
P x x refl
and
eq : x = y

we can conclude

P x y eq
```
where `refl : ∀x. x = x`. In short, for any property `P`, if `P` is true for `x x refl`, 
and we know that `eq : x = y`, then `P` is true for `x y eq`.

In a non-cubical context, this is obvious - the only proof of `x = y` for any `x` and `y` is `refl`, so it's close to a no-op. However, in HoTT, where there may
be many proofs of `x = y`, Axiom J is slightly less obvious. Why can we conclude that `P` is true for any path if we are only given that
it is true at `refl`?

## Why?

First, let's curry `P`, turning the initial arguments from a Pi-type to a Sigma-type:
```
P : Π (a b : A). a = b -> Type
==>
P : (Σ(a : A). Σ(b : A). a = b) -> Type
```
And rewrite the above accordingly.
```
Given:
A : Type
x y : A
P : (Σ(a : A). Σ(b : A). a = b) -> Type
with
P (x, x, refl)
and
eq : x = y

we can conclude

P (x, y, eq)
```

We will first apply the principle that if `a = b`, then `P a -> P b`, generally called something like `subst`itution. We can use `P (x, x, refl)` as our "`P a`". We then aim to show that:

```
Given:
eq : x = y
then
(x, x, refl) = (x, y, eq)
```
The first element is obvious - clearly `x = x`. Now we have to show that `(x, refl) = (y, eq)`. 
This seems impossible! We do _not_ have that `refl = eq`, due to the nature of HoTT.

Let's approach this by analysing the type of `(x, refl)` and `(y, eq)` (it's the same for both; otherwise the equality above wouldn't make sense).

Looking at the type of `P` above, we can conclude that
```
(x, refl) : Σ(b : A). x = b
                      ^ note the `x` here we got from 
                        the first element of the tuple
and then necessarily
(y, eq)   : Σ(b : A). x = b
```
Now for the trick. While we do _not_ have that `refl = eq`, we don't actually need it! 
It turns out that we can prove that any element of `Σ(b : A). x = b` is equal to any other element of `Σ(b : A). x = b`, irregardless of 
the proof of equality used - in other words, `Σ(b : A). x = b` only has one inhabitant. This property (only having one inhabitant) is called "contractibility".

A "proper" proof of this requires some careful thinking 
(One can be found <a href="https://1lab.dev/1Lab.Path.html">here as `Singleton-contract`, in Cubical Agda, courtesy of the 1lab</a>, 
and it's Lemma 3.11.8 in the HoTT book, at time of writing.), but I'll give an informal one here.

Consider what `Σ(b : A). x = b` _means_. One way of interpreting it is like it's a subset - that 
"`Σ(b : A). x = b` is the type of things that are elements of `A` equal to `x`".
However, there's only one of those! Namely, `x` itself. If we chose anything else, we wouldn't have `x = b` in the first place.
This means that our choice of equality doesn't matter at all - when we consider the tuple as a whole, there can only be one inhabitant.
Hence, as we already have that `x = y`, we have `(x, refl) = (x, eq) = (y, eq)`, completing our proof.

## TLDR

1. 
```
J : Π (A : Type) 
      (x y : A)
      (eq : x = y)
      (P : Π(a b : A). x = y -> Type).
         P x x refl 
      -> P x y eq
Goal : P x y eq
```
2. Consider the equivalent "curried" version, so curry `P`:
```
J : Π (A : Type) 
      (x y : A)
      (eq : x = y)
      (P : (Σ(a : A). Σ(b : A). a = b) -> Type).
         P (x, x, refl)
      -> P (x, y, eq)
Goal : P (x, y, eq)
```
3. Use `subst : a = b -> P a -> P b`:
```
Goal : (x, x, refl) = (x, y, eq)
```
4. Use that `Π x a b. a = b -> (x, a) = (x, b)` (with `a = (x, refl)` and `b = (y, eq)`:
```
Goal : (x, refl) = (y, eq)
```
5. Use that `Σ(b : A). x = b` is contractible (`Π (one two : Σ(b : A). x = b)). one = two`):
```
Done!
```
A handwavy "proof term" may look like (ommitting some "obvious" arguments, as one may do with implicit arguments in Agda):
```
Given:
subst : Π P a b. a = b -> P a -> P b
subst-fst : Π x y p q. (x , p) = (x , q) -> x = y -> (x , p) = (y , q)
pair-snd-eq : Π x a b. a = b -> (x , a) = (x , b)
singleton-contractible : Π x (one two : Σ(b : A). x = b)). one = two

We have:

J : Π (A : Type) 
      (x y : A)
      (eq : x = y)
      (P : (Σ(a : A). Σ(b : A). a = b) -> Type).
         P x x refl 
      -> P x y eq
J A x y eq P Pxxrefl = 
  subst  (pair-snd-eq singleton-contractible) Pxxrefl
```






