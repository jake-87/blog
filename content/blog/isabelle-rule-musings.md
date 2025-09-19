+++
title = "Isabelle/HOL rule musings"
date = 2025-09-19
+++

The following is very adapted from the <a href="https://isabelle.in.tum.de/dist/Isabelle2025/doc/tutorial.pdf">Isabelle/HOL 2025 tutorial</a>, particularly 5.2 and onwards. It will explain it better than I can.

I also make no promise that the following is entirely correct. It is my understanding, and while I have verified it best I can, I am sure there are errors.

# Notes

These rule tactics can be used in many different ways. While some general principles are mentioned below, if the prerequisits are met, 
every rule can theoretically be used with every rule tactic. Many of these various combinations will not be beneficial to proving a given
goal, and hence it is useful to know how the rule tactics interact with rules and proof states in order to select the correct pairing.

In that vein, the same rule will be interpreted differently by these different rule tactics. Read more for how they work, and hence why.

# `rule`, `erule`, `drule`, and `frule`, informally

We use `P`, `Q`, and `R` as arbitrary assumptions and goals throughout.

## `rule`

Consider the introduction rule `conjI`:

```hs
P   Q
----- conjI
P ∧ Q
```

If we have a goal of form `R ==> P ∧ Q`, from `conjI` we can infer that it is sufficent to seperately prove `P` and `Q`. Indeed, performing `apply (rule conjI)` on a goal of this shape will result in two goals:

```hs
1. R ==> P
2. R ==> Q
```

which can be then proven seperately. `rule` is for "backwards reasoning"; here we can see that as moving "backwards up" the introduction rule (Starting with the bottom, and ending with the top.) It is therefore useful for introduction rules, as demonstrated.

## `erule`

`erule` is designed to function with elimination rules. Take for example the elimination rule for disjunction:

```hs
P ∨ Q     (P ==> R)    (Q ==> R)
-------------------------------- disjE
              R
```
(Informally, to prove something about a disjunction, we must handle both possible cases.)


If we have a goal of the form:

```hs
P ∨ Q ==> R
```

we can perform `apply (erule disjE)` to transform this into two goals:

```hs
1. P ==> R
2. Q ==> R
```

where we must handle each possible case of the disjunction. `erule` is a bit of both backwards and forwards reasoning. In the above elimination rule, we can see that we need three assumptions to prove `R` --- we already have `P ∨ Q`, so it is sufficient to then only additionally prove `P ==> R` and `Q ==> R`, which are exactly the goals `erule` gives us.

Notable: `erule` deletes the already-held assumption, as it is "rarely useful" (quoting the Isabelle/HOL tutorial). See below's note for more.

## Note on `rule` vs `erule`

As mentioned, `erule` deliberately deletes the already-held assumptions when using an elimination rule. Above, we could see that `P ∨ Q` was not present in the generated goals, as it was already assumed. By instead using `rule`, we can keep the original goal in our assumptions as follows. If we apply the principles from the `rule` section to `disjE`, we would get:

```hs
P ∨ Q ==> R

apply (rule disjE)

⟦ P ∨ Q ⟧ ==> P ∨ Q
⟦ P ∨ Q; P ⟧ ==> R
⟦ P ∨ Q; Q ⟧ ==> R
```

i.e., `rule`, as expected, replaces the goal R with the goals sufficent to prove it. In this case, however, the first goal is trivial, and in the latter two, `P ∨ Q` holds little worth, as we know either `P` or `Q`. This is why `erule` automatically discharges the first goal and deletes the already-held assumption.

## `drule`

For the sake of example, recall (one of) the conjunction elimination rules:

```hs
P ∧ Q
----- conjunct1   (* We stick to the Isabelle/HOL naming. *)
  P
```

This rule is more specifically called a "destruction" rule, as it takes apart (destroys) the original term (here the conjunction) and concludes with one of its subterms. The `d` in `drule` therefore stands for `destruction` or `destroy`. Destruction rules are used for forwards-reasoning, as you move "down" the rule.

`apply (drule conjunct1)` therefore replaces an assumption of form `P ∧ Q` with one of `P`, deleting the original `P ∧ Q` assumption.

```hs
P ∧ Q ==> R

apply (drule conjunct1)

P ==> R
```

## `frule`

`frule` functions the same as `drule`, but does not delete the original premise. The `f` stands for `forwards`, as it is used for forwards-reasoning.

```hs
P ∧ Q ==> R

apply (frule conjunct1)

⟦ P ∧ Q; P ⟧ ==> R
```

## Useful related tools 

For all the above rule tactics, `[of P₁ P₂ ... Pₙ]` can be used to explicitly instantiate them, instead of using `rule_tac` (which is explained below). This instantiates left-to-right by occurence, so with 
```hs
thm conjI (* ⟦?P; ?Q⟧ ==> ?P ∧ ?Q *)
```
`conjI [of A B]` will result in the instantiation `⟦A; B⟧ ==> A ∧ B`, from where the rule will proceed.

```hs
R ==> A ∧ B
(* Isabelle can solve this on its own, but for the sake of example *)
apply (rule conjI [of A B]) 
1. R ==> A
2. R ==> B
```

Notably, `[of ...]` does still work with the `_tac` family, but there is (as far as I know) no reason to use it over `_tac`'s explicit instantiation. Speaking of:

## The `_tac` suffixes

For each of the above,  there is a corresponding `_tac` (standing for tactic): `rule_tac`, `drule_tac`, etc. These `_tac`s are slightly more powerful than their `_tac`less equivalents. Each allows for the various premises to be explicitly instantiated by name. For example, with the `conjI` rule, one could explicitly write

```hs
R ==> A ⋀ B

apply (rule_tac P=A and Q=B in conjI)

1. R ==> A
2. R ==> B
```

This allows the rule to refer to meta-forall-bound variables, which most commonly arise when performing induction or similar. For a goal of form:

```hs
P n
```

with some `n :: nat` mentioned in `P`, `apply (induction n)` will generate the goals

```hs
P 0 
⋀ nat. P nat ==> P (Suc nat) 
```

`nat` is a meta-forall bound variable. This functions as expected; during the induction step of an inductive proof, we must prove that for any `nat`, `P nat ==> P (Suc nat)`. However, `rule` and friends cannot refer to `nat` directly using `[of ...]` (Although, automatic unification can work). Instead, we must use `rule_tac` and friends to refer to it. I am not quite sure why this is, but I presume it is due to internal complexities around the meta-forall.

<details><summary>A small artificial example of this difference, if the above is not clear.</summary>

Recall that `[of ...]` can be used to explicitly instantiate:

```hs
lemma without_forall: "A ∧ B ⟹ A"
  apply (erule conjE [of A]) (* succeeds *)

lemma with_forall: "⋀ a b. a ∧ b ⟹ a"
  apply (erule conjE [of a]) (* fails *)
  apply (erule_tac P=a in conjE) (* succeeds *)
```
These lemmas are otherwise equivalent; only their presentation differs.

</details>


In general, these `_tac` rules have the form
```hs
rule_tac v₁ = t₁ and v₂ = t₂ and ... and vₙ = tₙ in RULE
```
where `vₙ` is a variable occuring in `RULE`, and `tₙ` is a locally bound variable or assumption.

`rule_tac` and friends have an additional advantage, which is that they can be told to refer to a specific goal. If we have goals of the form

```hs
1. A ∧ B ==> R
2. P ∧ Q ==> R
```

`d/frule conjE` will by default refer to the first rule. Using `d/frule_tac [i] conjE`, we can refer to the `ith` goal (starting indexing at 1). When combining this with specifying variables, the goal specifier comes first (i.e., `apply (erule_tac [1] P=a in conjE)`.)

# `rule`, `erule`, `drule`, and `frule`, (more) formally

This section is almost entirely copied from the aformentioned Isabelle/HOL tutorial.

Given an arbitrary rule R of the form:

```hs
P₁ P₂ ... Pₙ
------------  R
      Q
```

For elimination rules, `P₁` is referred to as the "Major premise". This will usually be the object being eliminated. In `disjE` for example, it is `P ∨ Q`.

1. Method `rule` unifies `Q` with the current subgoal, and replaces it with the `n` subgoals `P₁ ... Pₙ`.

2. Method `erule` unifies `Q` with the current subgoal, and `P₁` with the first suitable assumption. The subgoal is then replaced with the `n - 1` subgoals `P₂ ... Pₙ`, and the assumption `P₁` is deleted.

3. Method `drule` unifies `P₁` with the first suitable assumption, which it then deletes. It then adds the `n - 1` subgoals `P₂ ... Pₙ`, and the original subgoal gains the assumption `Q`.

4. Method `frule` is like `drule`, but it does not delete `P₁`.


