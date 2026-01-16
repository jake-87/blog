+++
title = "Interpreters and type safety"
date = 2026-01-17
+++

It is well known that most software has bugs<sup><i>[Citation needed]</i></sup>. 
It is also well known that interpreters are software[^1]. One issue that one might
run into in an interpreter is _type safety_: It may very well be possible for your
typed language's interpreter to nevertheless end up in a state where it is asked 
to evaluate `1 + "hi!"`. What can we do about that?

## GADTs: Generalized ADTs[^2]

GADTs are a technique for increasing type safety in a program, by "indexing" a constructor by a type.
One of the canonical examples for this is an `Option`-like. Imagine we wanted a type-safe way to ensure
that in some cases, we could always extract the value inside. One way of doing this is to add an extra
type parameter, and use that to mark whether something is inside. We will use OCaml for this demonstration.

```ocaml
(* Dummy constructors; we are only using these for their types *)
type yes = Yes
type no = No

type (_, _) maybe =
      |  ^ regular "information inside" type
      ^ marker of whether the information exists 
  | Nothing : (no, 'a) maybe
  | Just : 'a -> (yes, 'a) maybe
```
Note that we have used a different syntax than usual. Normally, we put a type parameter in the constructor, 
like `type 'a list`, and then use that throughout (`'a * 'a list`). However, as we are changing that type
in the "return" of the constructor, we need a syntax that allows us to change that. `_` marks a type parameter 
we "don't need to name" (as we always specify it), and in this case, we always specify both. Then, we
add our elements before a `->`, mimicking a function. The general form of this in OCaml is `t1 * t2 * ... * tn -> t`.

Now, if we have something of type `(yes, 'a) maybe`, we know it contains a value! The only other way a `maybe` can be
formed is via `Nothing`, but that gives it type `(no, 'a) maybe`, so it'd be type-incorrect. This means we
can write a function like
```ocaml
let get (x : (yes, 'a) maybe) -> 'a = 
  match x with
  | Just a -> a
```
and have it be total; there is no missing case, because the missing case is a type error. Note that when writing a 
function like `or_else : (_, 'a) maybe -> 'a -> 'a`, we need to use this interesting piece of syntax:
```
let or_else : type m. (m, 'a) maybe -> 'a -> 'a = fun x -> 
  match x with
  | Just a -> a
  | Nothing -> def
```
This is as otherwise OCaml eagerly thinks "Ah, `m` must be `yes`!" (or equivalent for `no`) when seeing the `Just` case
in the pattern match. The `type` designator tells it to keep `m` as general as possible, instead of refining it.

## Type safe interpreters

How does this allow us to get a more type-safe interpreter, though? Well, let's start by introducing our expression 
type:
```ocaml
type _ expr =
```
We've added space for a type parameter. What should integers and booleans look like? Maybe they should be indexed by
their respective "meta" types.
```ocaml
  | Lit : int -> int expr
  | Bool : bool -> bool expr
```
Now, we want to add a case for `Add` and `Negate`. Naturally, we shouldn't be able to add two booleans, so let's restrict it
to only taking in `int expr`s. Similar for negate.
```ocaml
  | Add : int expr * int expr -> int expr
  | Negate : int expr -> int expr
```
We've already added type safety! It's impossible to write `Add (Bool true, Lit 5)`, because `Add` expects a `int expr`,
but `Bool true` is a `bool expr`. Let's keep going: `<=` (or `Leq`) should take two integers and return a boolean,
and `If` should take a boolean as its first argument, but anything for its second and third.
```ocaml
  | Leq : int expr * int expr -> bool expr
  | If : bool expr * 'a expr * 'a expr -> 'a expr
```
Hence note that we can still use generic parameters - we just need to think about when it makes sense to do so,
to get the type safety we want.

Now for functions. What should a function look like here? One way of implementing them is with a "Higher-order abstract syntax"(HOAS)[^3] approach, 
where we use the functions of the host language to make functions in the interpreted language. In that case, we should
take a function as the argument to our constructor, but when what is our result type? That function type! We'll
need to know it later to safely write `Apply`, so we "store" it in the type now. We also want the function
to take and return an `_ expr`, so we can do constructions like `Lam (fun x -> Add(x,x))`.
[^3]: Higher-order abstract syntax. A bit too in-depth to explore in full right now, but worth exploring.
```ocaml
  | Lam : ('a expr -> 'b expr) -> ('a expr -> 'b expr) expr
```
Note that the function is not an `expr` at first, as otherwise we would have no way to construct it in the first
place. (Remember, this is how we make functions.)

Then `Apply`. It should take in a function, and a value, and then return the function applied to the value. 
This hence translates as:
```ocaml
  | Apply : ('a expr -> 'b expr) expr * 'a expr-> 'b expr
```

As a finale, we'll add pairs and projections. I won't elaborate on these, hoping that it's becoming obvious how
they work.
```ocaml
  | Pair : 'a expr * 'b expr -> ('a * 'b) expr
  | Fst : ('a * 'b) expr -> 'a expr
  | Snd : ('a * 'b) expr -> 'b expr
```

<details><summary>All of the above in one code block</summary>

```ocaml
type _ expr =
  | Lit : int -> int expr
  | Bool : bool -> bool expr
  | Add : int expr * int expr -> int expr
  | Negate : int expr -> int expr
  | Leq : int expr * int expr -> bool expr
  | If : bool expr * 'a expr * 'a expr -> 'a expr
  | Lam : ('a expr -> 'b expr) -> ('a expr -> 'b expr) expr
  | Apply : ('a expr -> 'b expr) expr * 'a expr -> 'b expr
  | Pair : 'a expr * 'b expr -> ('a * 'b) expr
  | Fst : ('a * 'b) expr -> 'a expr
  | Snd : ('a * 'b) expr -> 'b expr
```
</details>

## Examples

Now let's look at some examples and their types.

```ocaml
utop # Lit 5;;
- : int expr = Lit 5

utop # Bool true;;
- : bool expr = Bool true

utop # Add (Lit 4, Lit 5);;
- : int expr = Add (Lit 4, Lit 5)

utop # Lam (fun f -> f);;
- : ('a expr -> 'a expr) expr = Lam <fun>

utop # Apply (Lam (fun x -> Add(x,x)), Lit 5);;
- : int expr = Apply (Lam <fun>, Lit 5)

utop # Lam (fun b -> If (b, Lit 5, Lit 6));;
- : (bool expr -> int expr) expr = Lam <fun>
```
We have that a construction like `Lam (fun b -> If (b, b, Lit 6))` is not just an error, but a _type error_!
This means it's caught at compile time!
```ocaml
utop # Lam (fun b -> If (b, b, Lit 6));;
                               ^^^^^
Error: This constructor has type int expr
       but an expression was expected of type bool expr
       Type int is not compatible with type bool
```

## The actual evaluator

From here, the evaluator itself is very simple. We just write the interpretation of our language, as usual:
```ocaml
let rec eval : type t. t expr -> t = fun x ->
  match x with
  | Lit a -> a
  | Bool b -> b
  | Add (a,b) -> eval a + eval b
  | Negate a -> - (eval a)
  | Leq (a,b) -> (eval a) <= (eval b)
  | If (cond,l,r) -> (if (eval cond) then (eval l) else (eval r))
  | Lam f -> (fun x -> f x)
  | Apply (f,x) -> eval ((eval f) x)
  | Pair (a,b) -> (eval a, eval b)
  | Fst a -> fst (eval a)
  | Snd a -> snd (eval a);;
```
The `Lam` and `Apply` cases are interesting. In the former, we need to return a _function_ of type `'a expr -> 'b expr`.
We can start by introducing a function `fun x ->`, but we have `f : 'a expr -> 'b expr`, so we just apply it.
In the latter, we need to `eval f` to get a function `'a expr -> 'b expr` from `f`, which is of type
`('a expr -> 'b expr) expr`. We can then pass it `x`, which is of type `'a expr` to get a `'b expr`, which
we must then evaluate again to get a `'b`.

Now we can run the above examples.
```ocaml
utop # eval (Lit 5);;
- : int = 5

utop # eval (Bool true);;
- : bool = true

utop # eval (Add (Lit 4, Lit 5));;
- : int = 9

utop # eval (Lam (fun f -> f));;
- : '_weak1 expr -> '_weak1 expr = <fun>
(* The `_weak1` is due to an internal OCaml limitation, called the "value restriction". *)

utop # eval (Apply (Lam (fun x -> Add(x,x)), Lit 5));;
- : int = 10

utop # eval (Apply (Lam (fun b -> If (b, Lit 5, Lit 6)), Bool true));;
- : bool expr -> int expr = <fun>
(* Note that this gives us a function returning `int expr`. We'd need to either use `Apply`, like below,
   or `eval` to get an `int` out of it.
*)

utop # eval (Apply (Lam (fun b -> If (b, Lit 5, Lit 6)), Bool true));;
- : int = 5
```

Remember, all of the above is completely type safe! We have succesfully constructed an interpreter that can never
type error.

## How do you apply this?

In short, it can be hard. One way is to have a "base" language that's parsed, and then have your typechecker produce
a representation like the above that you then know is type correct. However, this does prevent typechecker bugs
from sneaking through, which was our entire goal!

To some extent, this is kicking the bucket down the road - you are now relying that your "host language"'s typechecker
is correct. However, this is a numbers game; if you're writing in OCaml or Haskell or similar, the chances of that
are very very high unless you're using extremely weird features.

## Aside: Denotional semantics
The astute may have noted that this is essentially a version of denotional semantics. We're interpreting into the 
type-safe domain of the host language to ensure that the types always line up (as they must, for the interpretation
to succeed, and the typechecker of the host language ensures that). 

## All code
<details><summary>All the code in one block, for pasting.</summary>

```ocaml
type _ expr =
  | Lit : int -> int expr
  | Bool : bool -> bool expr
  | Add : int expr * int expr -> int expr
  | Negate : int expr -> int expr
  | Leq : int expr * int expr -> bool expr
  | If : bool expr * 'a expr * 'a expr -> 'a expr
  | Lam : ('a expr -> 'b expr) -> ('a expr -> 'b expr) expr
  | Apply : ('a expr -> 'b expr) expr * 'a expr -> 'b expr
  | Pair : 'a expr * 'b expr -> ('a * 'b) expr
  | Fst : ('a * 'b) expr -> 'a expr
  | Snd : ('a * 'b) expr -> 'b expr;;

let rec eval : type t. t expr -> t = fun x ->
  match x with
  | Lit a -> a
  | Bool b -> b
  | Add (a,b) -> eval a + eval b
  | Negate a -> - (eval a)
  | Leq (a,b) -> (eval a) <= (eval b)
  | If (cond,l,r) -> (if (eval cond) then (eval l) else (eval r))
  | Lam f -> (fun x -> f x)
  | Apply (f,x) -> eval ((eval f) x)
  | Pair (a,b) -> (eval a, eval b)
  | Fst a -> fst (eval a)
  | Snd a -> snd (eval a);;
```

</details>

[^1]: No, I won't do the same joke twice. Most of the time.
[^2]: I am not going into what "ADT" stands for, as it's a whole nother debate.
