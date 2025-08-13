+++
title = "Crafting a dependent typechecker, part 1"
date = 2025-07-19
+++

This series is intended to give an interested layperson some idea of what goes on behind-the-scenes in a dependent typechecker, and how they might implement one of their own. 

I personally hold the belief that dependent languages (or similar) will eventually be the future of programming, so knowing this information seems worthwhile to me. One may also simply be interested for the sake of it, or may be interested in implementing their own dependent language.

This series does assume a small base amount of knowledge around dependent programming languages. There are many tutorials out there far better than I could provide, so I will refer the reader to the wider internet if they are curious. (Return here afterwards, though!).

Additionally, the code snippets will be in OCaml, although a Haskell or Rust version may be made available at a later date.

The technique we will implement in this series can most succinctly be described as "Bidirectional typechecking, utilizing Normalization by Evaluation". I do not expect the reader to understand that sentence for at least a little while ;).

All code snippets will be collated at the end of each page, so scroll all the way down if you're only interested in that.

## Resources

The following are resources I highly recommend for exploring this topic on your own.

- András Kovacs' "Elaboration Zoo", <a href="https://github.com/AndrasKovacs/elaboration-zoo">found here</a>. It was a primary source of inspiration for this series.
- "A tutorial implementation of a dependently typed lambda calculus", by Andres Löh, Conor McBride, and Wouter Swierstra; <a href="https://www.andres-loeh.de/LambdaPi/LambdaPi.pdf">found here</a>.
- "Checking Dependent Types with Normalization by Evaluation: A Tutorial", by David Christiansen, <a href="https://davidchristiansen.dk/tutorials/nbe/">found here</a>, for the more lisp-minded.

## The fundamental operation of typechecking

If you have implemented a typechecker before, you may have come to the realization that deciding when two things are the same is a very large component of typechecking. Whether it be simply checking that two arguments to `+` are both integers, or larger problems with polymorphism, the arguable core of typechecking is deciding equality.

In a dependent language, our types can (and often do) contain expressions that must be evaluated. Therefore, in order to explore how we check equality in the presence of evaluation, we will first start by considering how we do this for a very simple language - integer arithmetic. Take two expressions:

```js
1: 17 * 2 + 4 - 2
2: 7^2 - 4 * 4 + 3
```

How might we decide whether these are equal? The obvious solution is to evaluate them. Indeed, (sparing the reader the derivation), we can see they both evaluate to `36`, and so they are equal.

Let us now introduce a small extension: Variables with a known value, using "let". We will refer to the variables themselves as "bound variables", as they are bound by the "let". In this case, they are additionally bound to a value. Terms containing only bound variables are referred to "closed" terms - this alludes to, for example, closed and open systems in the real world. Getting back to evaluation, we could simply substitute the computed value:

```js
let x = 8^2 - 3;

1: x * 2 - 5
2: 13 * 39

substitute...

1: (8^2 - 3) * 2 - 5
2: 13 * 39
```

And then evaluate. (In this case, our result is 117.) However, what if the variable occurs multiple times? The following is a little bad, but not awful:

```js
let x = 8^2 - 3;

1: x * 2 - x^2 + 5

substitute...

1: (8^2 - 3) * 2 - (8^2 - 3)^2 + 5
```

But what if `x` was the following?

```js
let x = 8 ^ 2 - 3 + (14 * 2 - 4) + 8 - (14 ^ 2) * 3 + 5 * 5 - 18 ^ 3 + 5 * 6 - 12 * 4 * (6 ^ 2) + (8^2 - 3) * 2 - 18 * 14 ^ 3 - 2 * (4 - 3 ^ 2) + 12 * 18 - 3^4 + 1
```

Duplicating that would clearly not be ideal!

Perhaps we could compute `x` in advance, and then substitute that result in. This is much better:

```js
let x = 8^2 - 3;

evaluate and substitute...

let x = 61;

1: 61 * 2 - 61^2 + 5
```

(In fact, there is yet another optimization to do: What if we don't need to compute it at all, say because it's not used? We will explore this possibility later.)

A simple OCaml interpretation of the terms of this language may be the following.
(sidetrack: A "term" is some element of a language. Above, we have the language of (restricted) integer arithmetic, and our terms are expressions of said. Hence, we call our OCaml interpretation of this language `term`.)
```ocaml
type variable = string

type term =
    | Literal of int 
    | Name of variable 
    | Mul of term * term 
    | Exp of term * term 
    | Add of term * term 
    | Sub of term * term 
    | Let of variable * term * term 
      (* let x = ...; ... *)
```

We then must make an environment that associates, or "binds", our names to terms. `Let` is often called a "binder" because it functions as the source of this binding. We will make our context a list of pairs `(variable name, term)`, and will reimpliement the lookup function for example's sake, although a suitable function is available as `List.assoc`. 


```ocaml
type environment = (variable * term) list

let rec lookup (env : environment) (name : variable) : term =
    match env with
    (* See note below. *)
    | [] -> failwith ("Variable " ^ name ^ " not found.")
    | (nm, value) :: rest ->
        if nm = name then
            value
        else
            lookup rest name
```

Note: In this series, we will not be considering the possibility that a name is used without being previously defined in some way. This is an error, and can be handled via normals means, e.g. erroring.

We now note something curious: Our lookup function need not return a fully computed value! It could return `7 + 2`, which we would have to further evaluate before we could do anything with. It would be possible of course to manually insert invariants ensuring that this is not the case, but we could still accidentally violate them. 

Here we hit one of the most important distinctions in our exploration of dependent typechecking.

## Terms vs. Values

A "term", as previously seen, is any element of a chosen language. Above, this was (some subset of) integer arithmetic. We could also consider the language of OCaml programs, of which the above examples would be terms. However, two terms are not always easy to compare. We may need to evaluate one, or both, and we don't know whether that will be needed until we start comparing them.

Instead, what if we defined another type that was guaranteed to be reduced to some suitable level? For this simple case, we could restrict this to contain only fully evaluated terms (i.e., integers).

```ocaml
type value =
    | VLiteral of int
```
We prefix these constructors with `V` to ensure they're distinct. Then, we could modify our environment and lookup function to only deal with values:

```ocaml
- type environment = (variable * term) list
+ type environment = (variable * value) list

- let rec lookup (env : environment) (name : variable) : term =
+ let rec lookup (env : environment) (name : variable) : value =
```

and now we know that `lookup` will always return something we can handle immediately, with no further processing.

In the general case, we call this "suitably-reduced" constraint a "normal form". We can say that `value` contains only elements of `term` in this chosen normal form.

Now we can finish our little evaluator. We do a slight hack to perform arithmetic inside values; if our value type contained more than just the constructor `VLiteral`, we would have to consider how to handle that case. This is something we will cover soon.

```ocaml
(* OCaml does not have integer exponentiation by default. Definition omitted. *)
let pow (a : int) (b : int) : int = ...

let rec do_op (op : int -> int -> int) (a : value) (b : value) : value =
    (* Hack: This is only valid because we know our value type has one constructor, VLiteral. *)
    let VLiteral va = a in
    let VLiteral vb = b in
    VLiteral (op va vb)
    
let rec eval (env : environment) (tm : term) : value =
    match tm with
    | Literal i -> VLiteral i
    | Name nm -> lookup env nm
    | Mul (a, b) -> do_op ( * ) (eval env a) (eval env b)
    | Exp (a, b) -> do_op pow   (eval env a) (eval env b)
    | Add (a, b) -> do_op ( + ) (eval env a) (eval env b)
    | Sub (a, b) -> do_op ( - ) (eval env a) (eval env b)
    | Let (nm, def, rest) ->
        let result = eval env def in
        let new_env = (nm, result) :: env in
        eval new_env rest
```

In the `Let` case, we compute the definition first, then extend our environment with the new name and value before computing the rest of the expression. 

We then, by returning a `value`, have a guarantee that whatever comes out of `eval` is as simple as we want it to be.

Comparing for equality is then very simple: We use `eval` to evaluate our terms into values, then because our values are in a nice normal form, we can compare them easily guaranteed.

## Wrap up

So far we have covered:
1. Languages and terms.
2. Simple evaluation strategies for let bound variables & closed terms.
3. Values and normal forms.

In the next part, we will explore how to take these concepts and apply them to something much closer to a "real" programming language, with `lambda` expressions and application, which will require the introduction of a "closure". From there, we can explore how to deal with open terms, containing free variables, and onwards to dependent types.

## Code

<details>

```ocaml
type variable = string

type term =
    | Literal of int 
    | Name of variable 
    | Mul of term * term 
    | Exp of term * term 
    | Add of term * term 
    | Sub of term * term 
    | Let of variable * term * term 
      (* let x = ...; ...*)
      
type value =
    | VLiteral of int
      
type environment = (variable * value) list

let rec lookup (env : environment) (name : variable) : value =
    match env with
    (* See note below. *)
    | [] -> failwith ("Variable " ^ name ^ " not found.")
    | (nm, value) :: rest ->
        if nm = name then
            value
        else
            lookup rest name
      
(* OCaml does not have integer exponentiation by default. Definition omitted. *)
let pow (a : int) (b : int) : int = ...

let rec do_op (op : int -> int -> int) (a : value) (b : value) : value =
    (* Hack: This is only valid because we know our value type has one constructor, VLiteral. *)
    let VLiteral va = a in
    let VLiteral vb = b in
    VLiteral (op va vb)
    
let rec eval (env : environment) (tm : term) : value =
    match tm with
    | Literal i -> VLiteral i
    | Name nm -> lookup env nm
    | Mul (a, b) -> do_op ( * ) (eval env a) (eval env b)
    | Exp (a, b) -> do_op pow   (eval env a) (eval env b)
    | Add (a, b) -> do_op ( + ) (eval env a) (eval env b)
    | Sub (a, b) -> do_op ( - ) (eval env a) (eval env b)
    | Let (nm, def, rest) ->
        let result = eval env def in
        let new_env = (nm, result) :: env in
        eval new_env rest
```

</details>
