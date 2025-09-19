+++
title = "Isabelle rule musings"
date = 2025-09-19
+++

The following is very adapted from the <a href="https://isabelle.in.tum.de/dist/Isabelle2025/doc/tutorial.pdf">Isabelle/HOL 2025 tutorial</a>, particularly 5.2 and onwards. It will explain it better than I can.

I also make no promise that the following is entirely correct. It is my understanding, and while I have verified it best I can, I am sure there are errors.

# Notes

These rule tactics can be used in many different ways. While some general principles are mentioned below, if the prerequisits are met, 
every rule can theoretically be used with every rule tactics
and you will get many results (some of which will be useful!).

In that vein, the same rule will be interpreted differently by these different rule tactics! Read more for how they work, and hence why.

# `rule`, `erule`, `drule`, and `frule`, informally

We use `P`, `Q`, and `R` as arbitrary goals throughout.

## `rule`

Consider the introduction rule `conjI`:

```
P   Q
----- conjI
P ∧ Q
```

If we have a goal of form `R ==> P ∧ Q`, it is hence sufficent to seperately prove `P` and `Q`. Indeed, performing `apply (rule conjI)` on a goal of this shape will result in two goals:

```
1. R ==> P
2. R ==> Q
```

Which can be then proven seperately. `rule` is for "backwards reasoning"; here we can see that as moving "back up" through the introduction rule. It is generally used on the right-hand-side of the meta-implication `==>`, for introduction rules.

## `erule`

`erule` is designed to function with elimination rules. Take for example the elimination rule for disjunction:

```
P ∨ Q     (P ==> R)    (Q ==> R)
-------------------------------- disjE
              R
```
(Informally, to prove something about a disjunction, we must handle both possible cases.)


If we have a goal of the form:

```
P ∨ Q ==> R
```

we can perform `apply (erule disjE)` to transform this into two goals:

```
1. P ==> R
2. Q ==> R
```

Where we must handle each possible case of the disjunction. `erule` is a bit of both backwards and forwards reasoning; it is somewhat "sideways". In the above elimination rule, we can see that we need three elements to prove `R`. We already have `P ∨ Q`, so it is sufficient to then prove `P ==> R` and `Q ==> R`, which are the goals `erule` gives us.

Notable: `erule` deletes the already-held assumption, as it is "rarely useful" (to quote the Isabelle/HOL tutorial). See below's note for more.

## Note on `rule` vs `erule`

As mentioned, `erule` deliberately deletes the already-held assumptions when using an elimination rule. Above, we could see that `P ∨ Q` was not present in the generated goals, as it was already proven.

Note that it is possible to use `rule` for elimination rules as well. If we apply the principles from above, we would get:

```
P ∨ Q ==> R

apply (rule disjE)

⟦ P ∨ Q ⟧ ==> P ∨ Q
⟦ P ∨ Q; P ⟧ ==> R
⟦ P ∨ Q; Q ⟧ ==> R
```

i.e., `rule`, as expected, replaces the goal R with the goals sufficent to prove it. In this case, however, the first goal is trivial, and in the latter two, `P ∨ Q` holds little worth. This is why `erule` automatically proves the first goal and deletes the already-held assumption.

## `drule`

For the sake of example, recall (one of) the conjunction elimination rules:

```
P ∧ Q
----- conjunct1   (* We stick to the Isabelle/HOL naming. *)
  P
```

This rule is more specifically called a "destruction" rule, as it takes apart (destroys) the original term (here the conjunction) and returns one of its subterms. The `d` in `drule` therefore stands for `destruction` or `destroy`. Destruction rules are used for forwards-reasoning, as you move "down" the rule.

`apply (drule conjunct1)` therefore replaces an assumption of form `P ∧ Q` with one of `P`, deleting the original `P ∧ Q` assumption.

```
P ∧ Q ==> R

apply (drule conjunct1)

P ==> R
```

## `frule`

`frule` functions the same as `drule`, but does not delete the original premise. The `f` stands for `forwards`, as it is used for forwards-reasoning.

```
P ∧ Q ==> R

apply (frule conjunct1)

⟦ P ∧ Q; P ⟧ ==> R
```

## `_tac` suffixes

For each of the above,  there is a corresponding `_tac` (standing for tactic): `rule_tac`, `drule_tac`, etc. These `_tac`s are slightly more powerful than their `_tac`less equivalents. They allow the rule to refer to meta-forall-bound variables, which most commonly arise when performing induction or similar. For a goal of form:

```
P n ==> R
```

with some `n :: nat` mentioned, `apply (induction n)` will generate the goals

```
P 0 ==> R
⋀ nat. P nat ==> P (Suc nat)
```

`nat` is a meta-forall bound variable. This functions as expected; during the induction step of an inductive proof, we must prove that for any `nat`, `P nat ==> P (Suc nat)`. However, `rule` and friends cannot work with `nat` directly. Instead, we must use `rule_tac` and similar, with the form `apply (rule_tac P = nat in RULE)`. I am not quite sure why this is, but I presume it is due to internal complexities around the meta-forall.

In general, these `_tac` rules have the form
```
rule_tac v₁ = t₁ and v₂ = t₂ and ... and vₙ = tₙ in RULE
```
where `vₙ` is a variable occuring in `RULE`, and tₙ is a local meta-forall bound variable or assumption.

`rule_tac` and friends have an additional advantage, which is that they can be told to refer to a specific goal. If we have goals of the form

```
1. A ∧ B ==> R
2. P ∧ Q ==> R
```

`d/frule conjE` will by default refer to the first rule. Using `d/frule_tac [i] conjE`, we can refer to the `ith` goal (indexed at 1). When combining this with specifying variables, the goal specifier comes first.

# `rule`, `erule`, `drule`, and `frule`, (more) formally

This section is almost entirely copied from the aformentioned Isabelle/HOL tutorial.

Given an arbitrary rule R of the form:

```
P₁ P₂ ... Pₙ
------------  R
      Q
```

1. Method `rule` unifies `Q` with the current subgoal, and replaces it with the `n` subgoals `P₁ ... P₂`.

2. Method `erule` unifies `Q` with the current subgoal, and `P₁` with the first suitable assumption. The subgoal is then replaced with the `n - 1` subgoals `P₂ ... Pₙ`, and the assumption `P₁` is deleted.

3. Method `drule` unifies `P₁` with the first suitable assumption, which it then deletes. It then adds the `n - 1` subgoals `P₂ ... Pₙ`, and the original subgoal gains the assumption `Q`.

4. Method `frule` is like `drule`, but it does not delete `P₁`.
