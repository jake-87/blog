+++
title = "Crafting a dependent typechecker, part 2"
date = 2026-03-28
draft = true
+++

Last time we left off by exploring arithmetic expressions, and how we could seperate them into "terms" and "values", with values being a generally more desirable state. 
This time, we are going to move to something much more applicable to programming: The Lambda Calculus. It is not the intent of this series to give a thorough introduction to the lambda calculus, so I will refer the reader to the wider internet, and a tutorial I enjoyed: https://personal.utdallas.edu/~gupta/courses/apl/lambda.pdf

I'd also like to float the concept of de Bruijn indexes and levels. While they will not be used in this part, they form a core part of normalization by evaluation, and so I would recommend the reader start to think about what the presented terms would look like using de Bruijn. To assist in this process, I wil shamelessly link my own tutorial on the subject, <>, although I highly encourage the reader to do their own research also. Many of the terms presented will also have their de Bruijn forms shown.

First, let's define what "the" lambda calculus term looks like. This is fairly simple; a term is either a variable, an abstraction (which we will denote `Lam`, as `Abs` is slightly too close to `App`), or an application. We refer to `Lam` as a "constructor", as it constructs (read: creates) functions, and `App` as an "eliminatior", as it eliminates (read: removes) functions.

```ocaml
type variable = string

type term =
    | Var of variable
    | Lam of variable * term
    | App of term * term
```

We covered let bindings last time, and while you can emulate them in the lambda calculus proper using immediately-applied functions, we are going to add them to our core language for the sake of example.

```ocaml
    | Let of variable * term * term
```

As quickly mentioned last time, `Let` is an example of a binder, as it binds names to terms. `Lam` is also a binder, as it functions the same during application. As a primer, here is a vaguely complicated term and its internal representation:

```ocaml
λf. (λx y. f x y)
    (let q = (λx. x);
     f q)
Lam("f",
    App(
        Lam("x", Lam("y", App(App(Var "f", Var "x"), Var "y"))),
        Let("q", Lam("x", Var "x"),
                 App(Var "q", Var "f"))))
```

Evaluating the lambda calculus is not terribly complicated. Both when we encounter an applied lambda (something of form `(λx. stuff) k`, like above) and when we encounter a let binding, we simply add the name/term pair to our enviroment and continue on. Then, name lookups become searching this enviroment. Lookup is the same as in the previous part, so we'll skip reimplimenting it here and just reference `List.assoc`.

```ocaml

(* Don't worry, we're about to define value. *)
type enviroment = (variable * value) list

let lookup = List.assoc
```

At this point in the previous post, we discussed what a value should be for integer arithmatic, as having potentially unevaluated terms in our enviroment is not ideal. We will do similar here. We wish for our values to be in a normal form that is suitably reduced for our purposes - but what does "suitable reduced" mean for a lambda calculus?

It is obvious at first to simply try and reduce terms as much as possible. In that case, we may end up being left with a lambda, so a lambda is reasonably in our normal form and should be added to our value type.

```ocaml
type value =
    | VLam of ?
```

What goes there? We need the name of the lambda, which we can get straight from the term. Using this lambda value, we aim to possibly continue evaluation later - but if more bindings are added in the meantime, using the "current" enviroment could result in nasty scope escapes:

```ocaml
(*
    This would be valid! The context when we finally evaluate the body of `f`'s lambda would contain the
    binding for `g`.
*)
let f = λx. g x;
let g = λy. y;
f "10"
```

Hence, we must capture the enviroment at the moment the value is created. 

Then, we need to put the body of the lambda in. We can't have this be a value, because there's no reasonable way to evaluate that body without information on what the argument should be - hence, we leave it as a term, assuming that we can supply a term to continue evaluation later. This is consistent with our goal of "as reduced as possible" - indeed, we cannot reduce the lambda any more than we have.

```ocaml
type value =
    | VLam of enviroment * variable * term

```

There are two 

```
    | VNeu of neutral

and neutral =
    | NVar of variable
    | NApp of neutral * value
```

The (rather) observant reader may note something: Our neutral structure is essentially just a list! Let's consider an example:
```haskell
f (λx. x) (λa f. f a) (λy. y)
```

Becomes:

```haskell
1: (λx. x)
2: (λa f. f a)
3: (λy. y)

VNeu(NApp(NApp(NApp(NVar("f"), 1), 2), 3))
```

Surely we could do better - and indeed, we can. Instead of making our neutrals recursive, we can split them out into two different structures:

1. What goes at the "bottom" of the pile (at current, only `NVar`)
2. What goes at each level (at current, only `NApp`)

Each `NApp` has one value as argument. Hence, we could rewrite our neutral type as follows (`elims` for eliminators, of which application is one):

```ocaml
type neu_head =
    | NVar of variable

type neu_elims =
    | NApp of value

type neutral = Neu of neu_head * neu_elims list
```

TODO: back?

The first element of the `neu_elims list` is the "outermost" of the previous recursive structure, the second the second outermost, and so on. These are really best written with a "backwards" list, but that is difficult to nicely do in OCaml. Hence, our previous example becomes:

```
f (λx. x) (λa f. f a) (λy. y)

1: (λx. x)
2: (λa f. f a)
3: (λy. y)

VNeu((NVar "f", [NApp 1; NApp 2; NApp 3]))
                 ^ head  ^ snd   ^ third

(*
    When we wish to add another eliminator onto the neutral, we append it to the end.
*)
```

Much cleaner, and it also means we have immediate access to what is "blocking" evaluation of the neutral. If `f` changes at some point, we don't have to traverse the whole structure just to figure out "can we now do more reduction?". We also have immediate access to the innermost eliminator, which is the one we care the most about. If `f` changes to e.g. a lambda, we want to perform the application with the innermost eliminator.

# Aside: Why we can't use names for our final product

The complication mentioned earlier is as follows. Consider what will happen if we perform this application:

```haskell
let f = ...;
(λx f. f x) f
```

We substitute every occurence of `x` for `f`, like so:

```haskell
let f = ...;
(λf. f f)
```

Before, the "outer" `f` referred to the outermost lambda, but now it refers to the innermost lambda! The binder for `f` has changed during subsitution. This is something that de Bruijn complately resolves, and is one of the reasons that we use it. However, I want to leave de Bruijn for next time, so we're going to conveniently ignore this. Be wary of it however!<footnote>
