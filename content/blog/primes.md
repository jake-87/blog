+++
title = "A tale of four theorem provers, or: A (reasonably) opinionated comparison of Isabelle/HOL, Lean, HOL4, and Agda"
date = 2026-10-10
+++


# The above (wow, long title)

In which I compare Lean, Isabelle/HOL, Agda, and HOL4 with only mild regard for "fairness".


# What are we comparing, anyway?

All four of the above are theorem proving applications; that is, their purpose is to computer-formalize mathematics.
Broadly speaking, both Lean and Agda are dependent-types based systems, utilizing the Curry-Howard correspondence to prove theorems via a complex type system, whereas Isabelle/HOL and HOL4 are LCF-style systems, with a small "proof kernel" containing base rules (e.g. `forall x, x = x`) that all proofs must be constructed via.
Both foundations have advantages and disadvantages, and some will be discussed. 

# And on what?

I have formalized in all four a proof of the infinitude of primes; that is, the statement "For any number n, there is a prime larger than n". The exact proof formalized is the "usual" construction by Euclid, which you can find on Wikipedia if you want; <a href="https://en.wikipedia.org/wiki/Euclid%27s_theorem">see here</a>.

All of the proofs structurally look similar (mostly, we'll get to that), and hence it was the user experience that made the difference. For transparency, before this experiment, I was most familiar with Isabelle/HOL and Agda, and to some extent learnt HOL4 and Lean as part of this review.

# Comparison time!

What follows is a bunch of opinions and somewhat arbitrary categories. Don't expect an unbiased review, please. If you only want proof comparisons, skip to Proof Comparisons, and if you only want more opinions, read the below and then skip to Arbitrary Rankings. The opinions go first.

## Overarching similarities/differences

There are a few categories by which we can group these theorem provers. We have the above mentioned:

- LCF-style: Isabelle/HOL, HOL4
- Dependent-style: Lean, Agda

But there's also:

### Interactivity

- High-interactivity: Isabelle/HOL, Lean
- Low-interactivity: HOL4, Agda

Reasoning: Both Isabelle/HOL and Lean provide "live updates" as you type, and their structured proofs can be "down-arrow"'d through, to see intermediate steps. In contrast, both completed HOL4 and Agda proofs exist as fully-put-together terms that must be manually taken apart if you wish to inspect their internals. Both HOL4 and Agda can show you your current state and related information at any time, of course.

### Automation

- High-automation: Isabelle/HOL, HOL4
- Medium-automation: Lean
- Low-automation: Agda 

Isabelle/HOL has `sledgehammer`, which calls out to a number of external proof generation methods (SAT/SMT solvers, various FOL solvers, etc). For any goal that looks doable-but-annoying, there's a solid chance `sledgehammer` can solve it - this is nice because it saves you work, but the proofs it generates are also indecipherable, which is perhaps less than ideal. There also exists an equivalent for HOL4 called HolyHammer, but I didn't realize it existed until after I was writing this. Oh well. Isabelle/HOL and HOL4 both have very good support for many automated simplification and proof methods, which come in quite handy when trying to work with complex assumptions, for example. It may not be obvious how to proceed without simplification kicking in to chunk everything down. 

Lean has decent automation, but not to the same level as Isabelle/HOL or HOL4. Its `simp` is not nearly as productive, and while `grind` is a very neat approach that can sometimes rival `sledgehammer`, a lot of the time it's quite useless for reasons beyond me. Both `sledgehammer` and `grind` are all-or-nothing; if they don't solve the goal, they make no progress. This is in contrast to `simp` (in all three), or more specialized tools like `auto` (Isabelle/HOL) or `gvs` (HOL4), which can make some progress and then leave the context in (ideally) a better place. The fact Lean doesn't have nearly as good partial-automation is a bit of a shame, because in my experience that's what actually matters more. Notably, both Lean and Isabelle/HOL have a `try` (In Isabelle/HOL, `try`/`try0` for with/without `sledgehammer`; in Lean, `try?`/`exact?`/`rw?`) that "have a go" at your goal with various automated methods and direct solve attempts. HOL4 doesn't have an equivalent, as far as I can tell, which is unfortunate, as it's quite handy.

Agda has essentially no automation. The only simplification you get is what can be computed based on inputs to functions. This, to be a bit frank, sort of sucks. It's quite hard to get things done because there's so much manual fiddling that must be taken into account. I am personally not a fan.

### Foundations

Isabelle/HOL and HOL4 are both extremely classical in their foundations, which means that they accept the Law of Excluded Middle (∀ P. ¬P ∨ P), and both also axiomatize Hilbert's epsilon, which leads also to the Axiom of Choice. This seems to have a very positive effect on automated solvers, which quite often rely on laws such as double-negation elimination (¬¬P --> P), which is equivalent to the LEM. Lean is theoretically constructive (and hence does not have the LEM by default), but some of the "good" proof automation requires the LEM, and it seems the norm to use Lean classically, so that's what I did. `grind` for example just assumes you're using it. This does have the disadvantage that Lean isn't as good at dealing with e.g. existentials, for example.

Agda is constructive by default, and it seems the norm in the Agda world to keep your proofs that way, so I did. The big advantage of this is that after proving there are infinitely many primes, I can actually generate them! I can give my proof a number, and it'll spit out a prime bigger than that number. The disadvantage is that it is _horribly_ slow to do so:

- 0: 2 (instant)
- 1: 3 (instant)
- 2: 7 (instant)
- 3: 5 (instant)
- 4: 11 (quarter second)
- 5: 7 (quarter second)
- 6: 71 (3s)
- 7: 61 (10s)
- 8: 19 (40s)

This may make sense if you looked at the proof structure above; we consider `(n + 1)! + 1`, so inside that proof, it's checking primality at numbers around ~360,000; that's going to be a bit slow. This construction, it should be noted, isn't meant to be fast, but it's also the most natural one. Is it worth giving up the LEM and good proof automation? You decide.

### Dependent types as a foundation vs LCF

I'm more a fan of LCF because it seems more amenable to automation, and I don't see the point of carrying around proof terms (classical logic is too useful!). If you vehemently disagree, email me at contact AT blueberrywren.dev, and if I like your argument enough I'll post it here.

### Interaction mode

Both Lean and Isabelle/HOL are interacted with interactively. Lean (essentially) forces the use of VSCode, and Isabelle/HOL has its own editor (jEdit) that it also practically forces the use of. This is _fine_; I understand why they do this, as interactive development is reasonably hard to make generic. Both of them make it work.

HOL4 is interacted with via either an Emacs mode or a Vim mode, and keybinds that allow one to copy text in/out of a running HOL4 REPL. This sounds weird because it is, but it works surprisingly well. I was already an Emacs user, so nothing really changed for me.

Agda is also interacted with via an Emacs mode, but everything happens in your file; you can use keybinds to refresh the state, add proof goals, etc. It also works fine. 

As mentioned above, the advantage of the Lean/Isabelle/HOL approach is that one can see proofs in-progress, which you can't with Agda.

### Documentation

Both Isabelle/HOL and HOL4 have extremely good mechanisms for searching for theorems; an editor panel + `find_theorems` for the former, and `DB.find`/`DB.match` for the latter. These allow for the searching of theorems by both name and by patterns, so I could for example search for lemmas of form `_ < SUC _`. This is _so_ handy! Very often you know the _shape_ of something you want, but maybe not the lemma itself.

Lean has leansearch and loogle, which are interesting, but they:

1. Aren't built in.
2. Aren't as good.

Which kind of sucks. `exact?` and `rw?` exist as "here's stuff you might be able to do", but they're nowhere near as flexible.

Agda, as expected perhaps, does not have an equivalent. One must become a master of ~~zen~~ looking through the right parts of the Agda standard library.

# Proof Comparisons

Note: This isn't really meant to be a tutorial for any of these, though I will do a little explaining at the start. It's mostly for the reader to compare them, and see what ~equivalent statement proofs look like in different languages.

Let's get into the proofs! We'll go segment by segment, exploring sections of the proof and explaining as we go. Before that, some vaguely interesting stats:

- Isabelle/HOL: 117 LOC, with 19 proofs.
- Lean: 155 LOC, with 12 proofs.
- HOL4: 190 LOC, with 16 proofs.
- Agda: 264 LOC, with 45 proofs (29, not counting `where` blocks)

These are nowhere near apples-to-apples, but they're still fun.

We start with defining divisibility. In Agda we cheat and use the standard library version, so we can get proofs around computing divisors, which I didn't feel like redoing.

Isabelle/HOL:
```
definition divides :: "nat ⇒ nat ⇒ bool" where
  "divides n k = (∃j. j * n = k)"
```

Lean:
```
@[grind]
def divides (n k : Nat) : Prop :=
  ∃q, k = n * q
```

HOL4:
```
Definition divides_def:
  divides n k = ∃q. q * n = k
End
```

Agda (looks like, in the standard library):
```
record _∣_ (m n : ℕ) : Set where
  constructor divides
  field quotient : ℕ
        equality : n ≡ quotient * m
```

So, basically the same. Then there's a few lemmas around divisibility proved (`divides k 0`, `divides k k`, etc). One we'll show off is `divides n k ==> divides n j ==> divides n (k - j)`, as it's mildly interesting in some of the theorem provers. In all of the following, the mult/sub lemma essentially states `a * (b - c) = a * b - a * c`.

Isabelle/HOL:
```
lemma divides_diff: "divides x k ⟹ divides x n ⟹ divides x (k - n)"
  using divides_def by (metis diff_mult_distrib)
```

The proof search procedure `metis` does most of the work.

Lean:
```
theorem divides_sub (n j k : Nat) (h1 : divides n k) (h2 : divides n j)
  : divides n (k - j) := by
  obtain ⟨q1, h1⟩ := h1
  obtain ⟨q2, h2⟩ := h2
  unfold divides
  exists (q1 - q2)
  grind [Nat.mul_sub]
```

We do some unpacking, then identify `q1 - q2` as the other term (such that `n * (q1 - q2) = k - j`). Then grind with the appropriate lemma gets us there.

HOL4:
```
Theorem divides_sub:
  divides n k ⇒ divides n j ⇒ divides n (k - j)
Proof
  rpt strip_tac
  >> metis_tac[RIGHT_SUB_DISTRIB, divides_def]
QED
```

Like Isabelle/HOL, when supplied with the appropriate lemmas, `metis_tac` takes it out.

Agda:
```
divides-sub : ∀ {i j k} → i ∣ j → i ∣ k → i ∣ (j ∸ k)
divides-sub {i} (divides-refl q₁) (divides-refl q₂) = divides (q₁ ∸ q₂) (sym (*-distribʳ-∸ i q₁ q₂))
```

Very similar. `divides-refl` is an abbreviation for `divides _ refl`, and like in Lean, we must manually point out `q₁ ∸ q₂`.

It was interesting to me that the Lean proof was, to some extent, such a pain. I had to put more effort into that one than any of the others, despite Lean having reasonably decent automation! Finding the appropriate lemmas was mildly more painful, and this was while I was still puzzling over syntax, to be fair.


We next need to define the product of a list of numbers, which looks like the following:

Isabelle/HOL:
```
fun prod_list :: "nat list ⇒ nat" where
"prod_list [] = 1" | 
"prod_list (x # xs) = x * prod_list xs"
```

Lean:
```
@[simp, grind]
def prod_list (xs : List Nat) : Nat :=
  match xs with
  | [] => 1
  | x :: xs => x * prod_list xs
```

The `simp` and `grind` markers were meant to help the automated proof methods out, and they did!

HOL4:
```
Definition prod_list_def:
  (prod_list [] = 1) ∧
  (prod_list (x :: xs) = x * prod_list xs)
End
```

Agda:
```
prod-list : List Nat → Nat
prod-list [] = 1
prod-list (x ∷ xs) = x * prod-list xs
```

Defining things in HOL4 is a little interesting, because you're just defining the body as a proof! That definition spits out a theorem `prod_list_def` that is literally
```
val it = ⊢ prod_list [] = 1 ∧ ∀x xs. prod_list (x::xs) = x * prod_list xs: thm
```
!

Here in the Agda proof we also do a bunch of work to set up what will become a decision procedure for primality. This is because later we wish to ask "Is this prime?", and without the LEM to say "It must either be prime or not prime", we need to write an algorithm to decide this for us.

Back on track, we need to show a few lemmas around with prod list function. One of the interesting ones is as follows, where we prove that a number in said list will divide the product of the list.

Isabelle/HOL:
```
lemma divides_prod_list: "x ∈ set xs ⟹ divides x (prod_list xs)"
  apply (induct xs)
   apply simp
  apply auto
   apply (case_tac xs; simp add: divides_def)
  apply (case_tac xs; simp)
  apply (rule mult_divide_l)
  by assumption
```

Very implicit; it's hard to tell what's going on, but the basic structure is there. Induct on the list, do some casing, apply a lemma about `divides _ (_ * _)`.

Lean:
```
theorem divides_prod_list : ∀ xs x, x ∈ xs -> divides x (prod_list xs) := by
  intros xs x mem
  induction xs with
  | nil => grind
  | cons y ys ih =>
    by_cases h : (x = y)
    · rw [h]
      simp
      exists (prod_list ys)
    · cases mem with
      | head => grind
      | tail =>
        rename_i a
        have ⟨h, hq⟩ := ih a
        simp at *
        rw [hq]
        exists (y * h)
        grind
```

Reasonably large. We have to destructure the membership quite manually, which gets a little troublesome. It's a fairly straightforward proof, though.

HOL4:
```
Theorem divides_prod_list:
  x ∈ set xs ⇒ divides x (prod_list xs)
Proof
  Induct_on ‘xs’
  >- fs[]
  >- (fs[]
      >> rpt strip_tac
      >- (fs[prod_list_def, divides_def]
          >> qexists_tac ‘prod_list xs’
          >> simp[])
      >- (fs[divides_def, prod_list_def]
          >> qexists_tac ‘h * q’
          >> rev_drule EQ_SYM
          >> strip_tac
          >> fs[]))
QED
```

Also not crazy, although it's hard to see the exact structure without comments (which I didn't write :P). This is a good time to point out HOL4 proofs are literally just SML terms! There's nothing more to it! The combinators `>>` and `>-` (for apply latter to all subgoals of former, and to one subgoal of former resp.) are just infix functions composing other functions! It's all just SML! You interact with HOL4 through a REPL, so you construct stuff dynamically, but then you have to puzzle piece your function together afterwards. Once you know that, it's more obvious what's going on; we induct, handle the first case, then simplification on `MEM x (h ∷ xs)` give us two goals (`x = h` and `x ≠ h, MEM x xs` resp.)

Agda:
```
prod-list-divides : ∀ xs x → x ∈ xs → x ∣ prod-list xs
prod-list-divides [] x ()
prod-list-divides (y ∷ xs) x (here refl) = divides (prod-list xs) (*-comm y (prod-list xs))
prod-list-divides (y ∷ xs) x (there p) with prod-list-divides xs x p
... | divides q eq rewrite eq = divides (y * q) (sym (*-assoc y q x))
```

Very explicit, but also quite concise. Matching on the list membership is quite intuitive, because the Agda mode in Emacs includes a command `C-c C-c` to automatically case split on basically everything.

While it's much more implicit in Isabelle/HOL and HOL4 (a common theme), all four of these proofs take the form of asking whether the value we care about is at the head of the list, or somewhere later on, and that decides what we fill in divisibility with.

Then, Primality!

Isabelle/HOL:
```
definition prime :: "nat ⇒ bool" where
  "prime p = ((p > 1) ∧ (∀x. divides x p ⟶ x = p ∨ x = 1))"
```

Lean:
```
@[simp, grind]
def prime (n : Nat) : Prop :=
  (n > 1) ∧ (∀k, divides k n -> k = 1 ∨ k = n)
```

HOL4:
```
Definition prime_def:
  prime n = ((1 < n) ∧ (∀k. divides k n ⇒ k = 1 ∨ k = n))
End
```

Agda:
```
record Prime (n : Nat) : Set where
  constructor isprime
  field
    gt1 : n > 1
    div : ∀ k → k ∣ n → k ≡ 1 ⊎ k ≡ n
```

In Agda we also define what it means to be composite as a "positive" definition, instead of just "not prime"; this makes working with it quite a bit easier.


The next "interesting" proof is proving that every not-prime number greater than one has a prime factor. In Agda this is part of the definition of being composite, so we don't bother including it. We're going to move slightly faster from now on, so I won't explain each snippet. Just compare yourselves.

Isabelle/HOL:
```
lemma prime_factor: "¬(prime k) ⟹ k > 1 ⟹ ∃p. prime p ∧ divides p k"
  apply (induct k rule: measure_induct[of "id"]; simp)
  apply (rotate_tac 1)
  apply (subst (asm) prime_def)
  apply clarsimp
  apply (case_tac "prime xa")
   apply blast
  apply (erule_tac x=xa in allE)
  (* found with sledgehammer *)
  by (metis divides_gt divides_trans less_Suc0 not_less_iff_gr_or_eq zero_not_divides)
```

Lean:
```
theorem prime_factor : ∀k, ¬(prime k) -> 1 < k -> ∃p, prime p ∧ divides p k := by
  intros k nprime kgt1
  induction k using Nat.strongRecOn with
  | ind k' ih =>
    simp at nprime
    obtain ⟨x,⟨xd,dvds⟩,xn1,xnk⟩ := nprime kgt1
    clear nprime
    by_cases h : prime x
    · exists x
      grind
    · have lt : x < k' := by
        apply Nat.lt_of_le_of_ne
        · apply divides_less <;> grind
        · assumption
      have gt : 1 < x := by
        cases x <;> grind
      obtain ⟨p,⟨prm,dvds⟩⟩ := ih x lt h gt
      exists p
      refine ⟨prm, ?_⟩
      apply divides_trans
      · assumption
      · grind
```

HOL4:
```
Theorem prime_factor:
  ∀k. ¬(prime k) ⇒ 1 < k ⇒ ∃p. prime p ∧ divides p k
Proof
  completeInduct_on ‘k’
  >> rpt strip_tac
  >> qpat_x_assum ‘¬_’
                  (fn h => assume_tac
                          (REWRITE_RULE [prime_def] h))
  >> gvs[]
  >> Cases_on ‘prime k'’
  >- (qexists ‘k'’ >> simp[])
  >- (first_x_assum $ qspecl_then [‘k'’] mp_tac
      >> strip_tac
      >> ‘0 < k'’ by metis_tac[divides_gt1_gt0]
      >> ‘1 < k'’ by decide_tac
      >> ‘k' <= k’ by gvs[divides_less]
      >> ‘k' < k’ by decide_tac
      >> first_x_assum drule
      >> strip_tac
      >> gvs[]
      >> qexists ‘p’
      >> metis_tac[divides_trans])
QED
```

The steps of `0 < k'` ~> `1 < k'` and `k' <= k` ~> `k' < k` in the HOL4 one annoyed me a lot, but I couldn't figure out how to golf them down. Similarly, this line:
```
    obtain ⟨x,⟨xd,dvds⟩,xn1,xnk⟩ := nprime kgt1
```
of the Lean proof causes me pain.


We're almost there now! Two more steps to go: Prove there's always a prime outside a given set (list) of numbers, and use that to show the final statement. First, the former:

Isabelle/HOL:
```
lemma another_prime: "(∀x∈set xs. x > 1) ⟹ ∃p. prime p ∧ p ∉ set xs"
  apply (case_tac "prime (Suc (prod_list xs))")
  using prod_list_lt apply fastforce
  apply (frule prime_factor)
   apply clarsimp
  apply (case_tac "length xs = 0"; clarsimp)
   apply (rule prod_list_gt_zero; clarsimp)
  using prime_gt_one apply fastforce
  using divides_prod_list divides_diff prime_gt_one one_divides
  by (metis One_nat_def Suc_diff_Suc cancel_comm_monoid_add_class.diff_cancel lessI nat_less_le)
```

Lean:
```
theorem another_prime (xs : List Nat) (xsgt : ∀x, x ∈ xs -> 1 < x)
  : ∃p, prime p ∧ p ∉ xs := by
  by_cases h : prime (Nat.succ (prod_list xs))
  · exists (Nat.succ (prod_list xs))
    refine ⟨h, ?a⟩
    by_contra
    have lt : Nat.succ (prod_list xs) ≤ prod_list xs := by
      apply prod_list_lt <;> grind
    grind
  · have ⟨pf, isprm, dvds⟩ : ∃p, prime p ∧ divides p (Nat.succ (prod_list xs)) := by
      apply prime_factor
      · exact h
      · grind [prod_list_nz]
    by_cases g : pf ∈ xs
    · have pfdvds : divides pf (prod_list xs) := by
        apply divides_prod_list <;> grind
      have divone : divides pf ((Nat.succ (prod_list xs)) - prod_list xs) := by
        apply divides_sub <;> grind
      have also : divides pf 1 := by grind
      grind [divides_not_one]
    · grind
```

HOL4:
```
Theorem another_prime:
  ∀(xs : num list). (∀x. x ∈ set xs ⇒ 1 < x)
                    ⇒ ∃p. prime p ∧ p ∉ set xs
Proof
  rpt strip_tac
  >> Cases_on ‘prime (SUC (prod_list xs))’
  >- (qexists ‘SUC (prod_list xs)’
      >> metis_tac[LESS_REFL, OR_LESS, prod_list_lt, gt1_gt0_weaken])
  >- (drule prime_factor
      >> impl_tac
      >- metis_tac[ONE, LESS_MONO, prod_list_nz,gt1_gt0_weaken]
      >- (strip_tac
          >> Cases_on ‘MEM p xs’
          >- (‘divides p (prod_list xs)’
                by metis_tac[divides_prod_list]
              >> ‘divides p ((SUC (prod_list xs)) - prod_list xs)’
                by metis_tac [divides_sub]
              >> gvs[]
              >> metis_tac[prime_one, prime_zero, divides_not_one])
          >- (qexists ‘p’ >> metis_tac[])))
QED
```

Agda:
```
prime-not-in : ∀ xs → (∀ x → x ∈ xs → x > 0) → Σ _ λ k → k ∉ xs × Prime k
prime-not-in xs gt with 2 ≤? prod-list xs
... | no ¬a = 2 , lemma , two-prime
  where
    lemma : 2 ∈ xs → ⊥
    lemma pf with prod-list-lt xs gt 2 pf
    ... | also = ¬a also
... | yes a with prime-or-composite (suc (prod-list xs)) (s≤s (≤-trans (s≤s z≤n) a))
... | inj₁ x = suc (prod-list xs) , lemma , x
  where
    lemma : (suc (prod-list xs)) ∈ xs → ⊥
    lemma pf with prod-list-lt xs gt (suc (prod-list xs)) pf
    ... | also = 1+n≰n also
... | inj₂ (composite p Pp p<n p∣) with p ∈? xs
... | no ¬b = p , ¬b , Pp
... | yes b with prod-list-divides xs p b
... | divs with divides-sub p∣ divs
... | sothen = ⊥-elim (lemma {p} {prod-list xs} sothen λ{ refl → one-prime Pp })
  where
    suc-sub : ∀ i → suc i ∸ i ≡ 1
    suc-sub zero = refl
    suc-sub (suc i) = suc-sub i

    lemma : ∀ {i j} → i ∣ (suc j ∸ j) → i ≢ 1 → ⊥
    lemma {_} {j} div neq rewrite suc-sub j = divides-not-one div neq
```

The Agda proof differs slightly to accommodate the lack of ranges later.

The Isabelle/HOL proof clearly wins in terms of length here, but it's also really unclear what's going on. Everyone else gets progressively more verbose, and the HOL4 proof in particular here is a bit nastily nested. `metis_tac[]` (a first-order solver) does a lot of heavy lifting, as does `grind`. If you're wondering why both Isabelle/HOL and HOL4 have something named `metis`/`metis_tac`, it's because it was ported to Isabelle/HOL from HOL4.

We arrive at our final statement! Agda requires some more fiddling as it doesn't have ranges built in like the other two do, but we use our lemma above to construct the list `[2..n]`, and then show there's a prime outside that (and that it hence must be above `n`).

Isabelle/HOL:
```
lemma infinite_primes: "∃p. prime p ∧ p > n"
  apply (insert another_prime[where xs="[2 ..< Suc n]"])
  apply (case_tac "n ≤ 1"; clarsimp)
   apply (rule_tac x=2 in exI)
   apply simp
  apply (rule_tac x=p in exI)
  apply simp
  using prime_gt_one by force
```

Lean:
```
theorem infinite_primes : ∀ n, ∃p, prime p ∧ p > n := by
  intros n
  by_cases h : ¬(1 < n)
  · exists 2
    grind [prime_two]
  · simp at h
    obtain ⟨p, prm, notin⟩ := another_prime (List.range' 2 n) (by grind)
    refine ⟨p, prm, ?notin⟩
    grind
```

HOL4:
```
Theorem infinite_primes:
    ∀n. ∃p. prime p ∧ p > n
Proof
  strip_tac
  >> Cases_on ‘n < 1’
  >- (qexists ‘2’ >> simp[prime_two])
  >- (qspec_then ‘listRangeINC 2 n’ assume_tac another_prime
      >> ‘∀x. MEM x [2 .. n] ⇒ 1 < x’
        by (rpt strip_tac >> gvs[MEM_listRangeINC])
      >> first_x_assum rev_drule
      >> rpt strip_tac
      >> qexists ‘p’
      >> gvs[MEM_listRangeINC, prime_gt_one]
      >> ‘p ≠ 0’ by metis_tac[prime_zero]
      >> ‘p ≠ 1’ by metis_tac[prime_one]
      >> decide_tac)
QED
```

It annoys me I couldn't get this smaller, but oh well.

Agda:
```
infinite-primes : ∀ i → Σ _ λ p → p > i × Prime p
infinite-primes zero = 2 , s≤s z≤n , two-prime
infinite-primes (suc i) with prime-not-in (list-up-to (2+ i)) (list-up-to-nz (2+ i))
... | p , notin , Pp = p , DNE (2+ i ≤? p) lemma , Pp
  where
    notzero : p ≢ 0
    notzero = prime-not-zero p Pp

    implies : ∀ {a b} → (suc a ≤ b → ⊥) → (a ≡ b) ⊎ (a > b)
    implies {a} {b} x with compare a b
    ... | less .a k = ⊥-elim (x (s≤s (m≤m+n a k)))
    ... | equal .a = inj₁ refl
    ... | greater .b k = inj₂ (s≤s (m≤m+n b k))

    lemma : ((2+ i ≤ p) → ⊥) → ⊥
    lemma x with implies x
    ... | inj₁ refl = notin (there (here refl))
    ... | inj₂ (s≤s a) = notin (list-up-to-contains (2+ i) p (≤-trans a (≤-trans (n≤1+n i) (n≤1+n (suc i)))) (prime-not-zero p Pp))
```

We've done it! Euclid would be proud. (probably)

It's now time for more opinions!

# Arbitrary Rankings

I have five criteria I'll be ranking on:

- Ease of proof _discovery_ (how easy is it to figure out what I want to do)
- Ease of proof _manipulation_ (how easy is it to do what I want)
- Enjoyment (how much fun did I have)
- Annoyance factor (how often was I going "ugh!")
- Puzzled factor (how often did I go "why can't you solve this??")


## Ease of proof discovery (higher better)

1. Isabelle/HOL
2. HOL4
3. Lean 
4. Agda 

Not much to comment on here, really. `sledgehammer` is a great boon, and both Isabelle/HOL and HOL4's theorem discovery tools are great. Lean was reasonably close behind, with `try?` often giving good related lemmas as a solve, and Agda was clearly last. Paging through `.agda` files online to find lemmas is annoying.

## Ease of proof manipulation (higher better)

1. Agda 
2. Lean 
3. HOL4 
4. Isabelle/HOL 

Love it or hate it, Agda being entirely raw proof terms means things are essentially exactly what you tell them to be. There's never a moment where you're going "Damn, why won't the simplifier just expand this but not that!". Lean is pretty good here, as it's quite conservative around what it chooses to manipulate and everything is explicitly named. HOL4 has a pretty reasonable learning curve as one learns to use things like `qpat_assum` that can target based on patterns (e.g. `¬_`), but once you figure it out it's not bad. Isabelle/HOL really isn't stunning here; it's often hard to get it to do _exactly_ what you want. 

## Enjoyment (higher better)

1. HOL4
2. Lean 
3. Isabelle/HOL
4. Agda 

I had a great time learning both HOL4 and Lean, but HOL4 edges out because it's such a unique interaction mode and it's still very powerful. Lean was fun, although annoying at some times, and Isabelle/HOL wasn't particularly interesting, but some of that is because i'm familiar with it. Agda sort of sucked at times; writing a whole primality decision procedure and fiddling with type nonsense got quite annoying after a while. 

## Annoyance factor (lower better)

1. Agda 
2. HOL4 / Lean (tied second) 
4. Isabelle/HOL 

Agda "wins" for the same reasons as above. Having to do really manual proof search and fiddling constantly wasn't super pleasant. HOL4 and Lean both had their own annoyances and I think it's unfair to rank one over the other; the learning curve on assumption manipulation / sim was quite significant in HOL4 and Lean's automated tools were finicky enough it quite sucked at times. Isabelle/HOL i'm just used to, so there's some bias there. 

## Puzzled factor (lower better)

1. HOL4 
2. Lean 
3. Isabelle/HOL 
4. Agda 

Same reasons as above; there were a lot of times where HOL4 just had me going "huh????" because some theorem-tactic wasn't doing what I expected it to, or was transforming the goal in an unpredictable way. Lean had similar, where it would just randomly decide "erm actually i'm not going to solve this really simple goal for you with `grind` do it yourself please" in ways that left me baffled. Seriously, sometimes `grind` is smarter than `sledgehammer` and sometimes it's stupider than `simp`. Weird. Isabelle/HOL had some of the same but was generally fine, and Agda was utterly predictable.

# Overall best in class 

HOL4! I had a great time learning it, it's a seriously interesting system. I didn't _not_ enjoy Lean, but there's enough odd stuff going on to make me slightly wary of it, I suppose. Isabelle/HOL remains the one I'm best at (I am somewhat paid to write it, so that helps), and Agda is Agda.

What should you try? Well, all of them, but I would at least try out something new. If you've only used dependent theorem provers before, try Isabelle/HOL or HOL4, and vice versa. If you've only used theorem provers that work fully interactively like Lean, try HOL4 or Agda! New experiences are the joys of life.

# All the code 

Enjoy.


<details>
<summary>Unfold for everything.</summary>


Isabelle/HOL:
```
theory primes
  imports Main
begin

definition divides :: "nat ⇒ nat ⇒ bool" where
  "divides n k = (∃j. j * n = k)"

lemma zero_not_divides: "k ≠ 0 ⟹ ¬(divides 0 k)"
  by (simp add: divides_def)

lemma one_divides: "divides k 1 ⟹ k = 1"
  by (simp add: divides_def)

lemma divide_mult_l: "divides x k ⟹ divides x (n * k)"
  using divides_def by auto

lemma divides_diff: "divides x k ⟹ divides x n ⟹ divides x (k - n)"
  using divides_def by (metis diff_mult_distrib)

lemma divides_gt: "k ≠ 0 ⟹ n > k ⟹ ¬(divides n k)"
  by (clarsimp simp add: divides_def)

lemma divides_less: "k ≠ 0 ⟹ divides n k ⟹ n ≤ k"
  by (clarsimp simp add: divides_def)

lemma divides_trans: "divides n k ⟹ divides k j ⟹ divides n j"
  using divides_def by force

fun prod_list :: "nat list ⇒ nat" where
"prod_list [] = 1" | 
"prod_list (x # xs) = x * prod_list xs"

lemma prod_list_gt_zero: "(∀x∈set xs. x ≠ 0) ⟹ length xs > 0 ⟹ prod_list xs > 0"
  apply (induct xs)
   apply simp
  by (case_tac xs; clarsimp)

lemma prod_list_lt: "(∀x∈set xs. x > 0) ⟹ ∀x∈set xs. x ≤ prod_list xs"
  apply (induct xs)
   apply simp
  apply auto
  using prod_list_gt_zero apply fastforce
  apply (erule_tac x=x in ballE)
  using mult_eq_if apply auto[1]
  by blast

lemma divides_prod_list: "x ∈ set xs ⟹ divides x (prod_list xs)"
  apply (induct xs)
   apply simp
  apply auto
   apply (case_tac xs; simp add: divides_def)
  apply (case_tac xs; simp)
  apply (rule divide_mult_l)
  by assumption

definition prime :: "nat ⇒ bool" where
  "prime p = ((p > 1) ∧ (∀x. divides x p ⟶ x = p ∨ x = 1))"

lemma prime_zero[simp]: "¬(prime 0)"
  by (clarsimp simp add: prime_def)

lemma prime_one[simp]: "¬(prime 1)"
  by (clarsimp simp add: prime_def)

lemma prime_two[simp]: "prime 2"
  apply (clarsimp simp add: prime_def)
  apply (case_tac x; clarsimp simp add: divides_def)
  apply (case_tac nat; clarsimp simp add: divides_def)
  by (case_tac j; simp)

lemma prime_gt_one: "prime p ⟹ p > 1"
  using prime_def by simp

lemma notprime_split: "¬(prime k) ⟹ k > 1 ⟹ ∃n j. n < k ∧ j < k ∧ k = n * j"
  apply (clarsimp simp add: prime_def divides_def)
  apply (rule_tac x=x in exI)
  apply (rule conjI)
   apply (metis less_Suc0 linorder_neqE_nat mult_is_0 n_less_m_mult_n)
  apply (rule_tac x=j in exI)
  apply auto
  by (metis bot_nat_0.not_eq_extremum mult_zero_left not_less_zero)

lemma prime_not_divides_one: "prime p ⟹ ¬(divides p 1)"
  by (clarsimp simp add: prime_def divides_def)

lemma prime_factor: "¬(prime k) ⟹ k > 1 ⟹ ∃p. prime p ∧ divides p k"
  apply (induct k rule: measure_induct[of "id"]; simp)
  apply (rotate_tac 1)
  apply (subst (asm) prime_def)
  apply clarsimp
  apply (case_tac "prime xa")
   apply blast
  apply (erule_tac x=xa in allE)
  by (metis divides_gt divides_trans less_Suc0 not_less_iff_gr_or_eq zero_not_divides)


lemma another_prime: "(∀x∈set xs. x > 1) ⟹ ∃p. prime p ∧ p ∉ set xs"
  apply (case_tac "prime (Suc (prod_list xs))")
  using prod_list_lt apply fastforce
  apply (frule prime_factor)
   apply clarsimp
  apply (case_tac "length xs = 0"; clarsimp)
   apply (rule prod_list_gt_zero; clarsimp)
  using prime_gt_one apply fastforce
  using divides_prod_list divides_diff prime_gt_one one_divides
  by (metis One_nat_def Suc_diff_Suc cancel_comm_monoid_add_class.diff_cancel lessI nat_less_le)

lemma infinite_primes: "∃p. prime p ∧ p > n"
  apply (insert another_prime[where xs="[2 ..< Suc n]"])
  apply (case_tac "n ≤ 1"; clarsimp)
   apply (rule_tac x=2 in exI)
   apply simp
  apply (rule_tac x=p in exI)
  apply simp
  using prime_gt_one by force

end
```


Lean:
```
import Primes.Basic
import Mathlib.Tactic.ByContra

set_option linter.style.setOption false
set_option linter.flexible false
set_option linter.style.whitespace false

@[grind]
def divides (n k : Nat) : Prop :=
  ∃q, k = n * q

theorem divides_one (n : Nat) : divides n 1 -> n = 1 := by
  intro ⟨q, hq⟩
  cases n <;> cases q <;> grind

theorem divides_not_one (n : Nat) (d : divides n 1) (neq : n ≠ 1) : False := by
  grind [divides_one]

theorem divides_sub (n j k : Nat) (h1 : divides n k) (h2 : divides n j)
  : divides n (k - j) := by
  obtain ⟨q1, h1⟩ := h1
  obtain ⟨q2, h2⟩ := h2
  unfold divides
  exists (q1 - q2)
  grind [Nat.mul_sub]

theorem divides_less (n k : Nat) : k ≠ 0 -> divides n k -> n ≤ k := by
  intro knz div
  simp at *
  obtain ⟨q,hq⟩ := div
  cases q <;> grind

theorem divides_trans (n k j) : divides n k -> divides k j -> divides n j := by
  intro ⟨a,b⟩ ⟨c,d⟩
  simp [divides] at *
  exists (a * c)
  grind

@[simp, grind]
def prod_list (xs : List Nat) : Nat :=
  match xs with
  | [] => 1
  | x :: xs => x * prod_list xs

theorem divides_prod_list : ∀ xs x, x ∈ xs -> divides x (prod_list xs) := by
  intros xs x mem
  induction xs with
  | nil => grind
  | cons y ys ih =>
    by_cases h : (x = y)
    · rw [h]
      simp
      exists (prod_list ys)
    · cases mem with
      | head => grind
      | tail =>
        rename_i a
        have ⟨h, hq⟩ := ih a
        simp at *
        rw [hq]
        exists (y * h)
        grind

theorem prod_list_nz : ∀ xs, (∀ x ∈ xs, x > 0) -> prod_list xs > 0 := by
  intros xs f
  induction xs with
  | nil => grind
  | cons x xs ih =>
    obtain b : prod_list xs > 0 := ih (by grind)
    simp [*]

theorem prod_list_lt : ∀ xs, (∀ x ∈ xs, x > 0) -> ∀ y ∈ xs, y <= prod_list xs := by
  intro xs f y yin
  induction xs with
  | nil => grind
  | cons yp ys ih =>
    cases yin with
    | head =>
      obtain a : prod_list ys > 0 := by grind [prod_list_nz]
      exact Nat.le_mul_of_pos_right y a
    | tail _ a =>
      obtain a : y <= prod_list ys := ih (by grind) a
      obtain b : yp > 0 := f yp (by grind)
      apply Nat.le_trans
      · exact a
      · exact Nat.le_mul_of_pos_left (prod_list ys) b

@[simp, grind]
def prime (n : Nat) : Prop :=
  (n > 1) ∧ (∀k, divides k n -> k = 1 ∨ k = n)

theorem prime_two : prime 2 := by
  simp
  intros k x
  simp [divides] at x
  have ⟨q,hq⟩ := x
  (cases q <;> cases k <;> grind)


theorem prime_factor : ∀k, ¬(prime k) -> 1 < k -> ∃p, prime p ∧ divides p k := by
  intros k nprime kgt1
  induction k using Nat.strongRecOn with
  | ind k' ih =>
    simp at nprime
    obtain ⟨x,⟨xd,dvds⟩,xn1,xnk⟩ := nprime kgt1
    clear nprime
    by_cases h : prime x
    · exists x
      grind
    · have lt : x < k' := by
        apply Nat.lt_of_le_of_ne
        · apply divides_less <;> grind
        · assumption
      have gt : 1 < x := by
        cases x <;> grind
      obtain ⟨p,⟨prm,dvds⟩⟩ := ih x lt h gt
      exists p
      refine ⟨prm, ?_⟩
      apply divides_trans
      · assumption
      · grind


theorem another_prime (xs : List Nat) (xsgt : ∀x, x ∈ xs -> 1 < x)
  : ∃p, prime p ∧ p ∉ xs := by
  by_cases h : prime (Nat.succ (prod_list xs))
  · exists (Nat.succ (prod_list xs))
    refine ⟨h, ?a⟩
    by_contra
    have lt : Nat.succ (prod_list xs) ≤ prod_list xs := by
      apply prod_list_lt <;> grind
    grind
  · have ⟨pf, isprm, dvds⟩ : ∃p, prime p ∧ divides p (Nat.succ (prod_list xs)) := by
      apply prime_factor
      · exact h
      · grind [prod_list_nz]
    by_cases g : pf ∈ xs
    · have pfdvds : divides pf (prod_list xs) := by
        apply divides_prod_list <;> grind
      have divone : divides pf ((Nat.succ (prod_list xs)) - prod_list xs) := by
        apply divides_sub <;> grind
      have also : divides pf 1 := by grind
      grind [divides_not_one]
    · grind

theorem infinite_primes : ∀ n, ∃p, prime p ∧ p > n := by
  intros n
  by_cases h : ¬(1 < n)
  · exists 2
    grind [prime_two]
  · simp at h
    obtain ⟨p, prm, notin⟩ := another_prime (List.range' 2 n) (by grind)
    refine ⟨p, prm, ?notin⟩
    grind
```


HOL4:
```
open arithmeticTheory listTheory prim_recTheory listRangeTheory;
     
Definition divides_def:
  divides n k = ∃q. q * n = k
End

Theorem divides_not_one:
  ∀n. divides n 1 ⇒ n ≠ 1 ⇒ F
Proof
  simp[divides_def]
QED

Theorem divides_less:
  k ≠ 0 ⇒ divides n k ⇒ n ≤ k
Proof
  rpt strip_tac
  >> gvs[divides_def]
QED

Theorem divides_trans:
  divides a b ⇒ divides b c ⇒ divides a c
Proof
  rpt strip_tac
  >> gvs[divides_def]
  >> qexists ‘q * q'’
  >> gvs[MULT_ASSOC_COMM]
QED

Theorem divides_sub:
  divides n k ⇒ divides n j ⇒ divides n (k - j)
Proof
  rpt strip_tac
  >> metis_tac[RIGHT_SUB_DISTRIB, divides_def]
QED
            
Definition prod_list_def:
  (prod_list [] = 1) ∧
  (prod_list (x :: xs) = x * prod_list xs)
End

Theorem divides_prod_list:
  x ∈ set xs ⇒ divides x (prod_list xs)
Proof
  Induct_on ‘xs’
  >- fs[]
  >- (fs[]
      >> rpt strip_tac
      >- (fs[prod_list_def, divides_def]
          >> qexists_tac ‘prod_list xs’
          >> simp[])
      >- (fs[divides_def, prod_list_def]
          >> qexists_tac ‘h * q’
          >> rev_drule EQ_SYM
          >> strip_tac
          >> fs[]))
QED      

Theorem prod_list_nz:
  ∀xs. (∀x. x ∈ set xs ⇒ 0 < x) ⇒ 0 < prod_list xs
Proof
  rpt strip_tac
  >> Induct_on ‘xs’
  >> fs[prod_list_def]
QED
     
Theorem prod_list_lt:
  ∀xs. (∀x. x ∈ set xs ⇒ 0 < x) ⇒ ∀y. y ∈ set xs ⇒ y ≤ prod_list xs
Proof
  rpt strip_tac
  >> Induct_on ‘xs’
  >> gvs[]
  >> rpt strip_tac
  >> gvs[prod_list_def]
  >> metis_tac[prod_list_nz, LE_MULT_CANCEL_LBARE, LE_TRANS]
QED

Definition prime_def:
  prime n = ((1 < n) ∧ (∀k. divides k n ⇒ k = 1 ∨ k = n))
End

Theorem prime_zero:
  ¬(prime 0)
Proof
  fs[prime_def]
QED        


Theorem prime_one:
  ¬(prime 1)
Proof
  fs[prime_def]
QED

Theorem prime_gt_one[simp]:
  prime n ⇒ 1 < n
Proof
  fs[prime_def]
QED
              
Theorem prime_two:
  prime 2
Proof        
  fs[prime_def]
  >> rpt strip_tac
  >> fs[divides_def]
  >> (Cases_on ‘k’ >> Cases_on ‘q’ >> fs[MULT_SUC])
QED

Theorem divides_gt1_gt0:
  ∀n k. divides n k ⇒ 1 < k ⇒ 0 < n
Proof
  rpt strip_tac
  >> gvs[divides_def]
  >> (Cases_on ‘q’ >> gvs[MULT_SUC])
  >> (Cases_on ‘n’ >> gvs[])
QED

Theorem prime_factor:
  ∀k. ¬(prime k) ⇒ 1 < k ⇒ ∃p. prime p ∧ divides p k
Proof
  completeInduct_on ‘k’
  >> rpt strip_tac
  >> qpat_x_assum ‘¬_’
                  (fn h => assume_tac
                          (REWRITE_RULE [prime_def] h))
  >> gvs[]
  >> Cases_on ‘prime k'’
  >- (qexists ‘k'’ >> simp[])
  >- (first_x_assum $ qspecl_then [‘k'’] mp_tac
      >> strip_tac
      >> ‘0 < k'’ by metis_tac[divides_gt1_gt0]
      >> ‘1 < k'’ by decide_tac
      >> ‘k' <= k’ by gvs[divides_less]
      >> ‘k' < k’ by decide_tac
      >> first_x_assum drule
      >> strip_tac
      >> gvs[]
      >> qexists ‘p’
      >> metis_tac[divides_trans])
QED

Theorem gt1_gt0_weaken:
  ∀x. 1 < x ⇒ 0 < x
Proof
  strip_tac >> decide_tac
QED

Theorem another_prime:
  ∀(xs : num list). (∀x. x ∈ set xs ⇒ 1 < x)
                    ⇒ ∃p. prime p ∧ p ∉ set xs
Proof
  rpt strip_tac
  >> Cases_on ‘prime (SUC (prod_list xs))’
  >- (qexists ‘SUC (prod_list xs)’
      >> metis_tac[LESS_REFL, OR_LESS,
                   prod_list_lt, gt1_gt0_weaken])
  >- (drule prime_factor   
      >> impl_tac
      >- metis_tac[ONE, LESS_MONO, prod_list_nz,gt1_gt0_weaken]
      >- (strip_tac
          >> Cases_on ‘MEM p xs’
          >- (‘divides p (prod_list xs)’
                by metis_tac[divides_prod_list]
              >> ‘divides p ((SUC (prod_list xs)) - prod_list xs)’
                by metis_tac [divides_sub]
              >> gvs[]
              >> metis_tac[prime_one, prime_zero, divides_not_one])
          >- (qexists ‘p’ >> metis_tac[])))
QED

                    
Theorem infinite_primes:
    ∀n. ∃p. prime p ∧ p > n
Proof
  strip_tac
  >> Cases_on ‘n < 1’
  >- (qexists ‘2’ >> simp[prime_two])
  >- (qspec_then ‘listRangeINC 2 n’ assume_tac another_prime
      >> ‘∀x. MEM x [2 .. n] ⇒ 1 < x’
        by (rpt strip_tac >> gvs[MEM_listRangeINC])

      >> first_x_assum rev_drule
      >> rpt strip_tac
      >> qexists ‘p’
      >> gvs[MEM_listRangeINC, prime_gt_one]
      >> ‘p ≠ 0’ by metis_tac[prime_zero]
      >> ‘p ≠ 1’ by metis_tac[prime_one]
      >> decide_tac)
QED
```

Agda:

```
open import Data.Nat renaming (ℕ to Nat)
open import Data.Nat.Properties
open import Relation.Binary.PropositionalEquality
open import Data.Product
open import Data.Empty
open import Relation.Nullary.Negation
open import Data.List
open import Data.Sum
open import Relation.Nullary.Decidable
open import Relation.Binary
open import Relation.Nullary.Reflects
open import Data.Nat.Divisibility
open import Data.Bool using (Bool; true; false)
open import Data.Fin using (zero; suc; Fin; toℕ; fromℕ; fromℕ<)
open import Data.Fin.Properties using (toℕ-fromℕ; toℕ-fromℕ<)
open import Induction.WellFounded
open import Data.Nat.Induction using (<-wellFounded; <-rec)

import Data.List.Membership.DecPropositional as DecPropMembership
open DecPropMembership Data.Nat._≟_
open import Data.List.Relation.Unary.Any using (here; there)

-- The prime decision procedure was with assistance from https://gist.github.com/copumpkin/1286093

div-goes-into : ∀ i k → k ≢ 0 → i ∣ k → i ≤ k
div-goes-into i k neq (divides zero eq) = ⊥-elim (neq eq)
div-goes-into zero k neq (divides (suc q) eq) = z≤n
div-goes-into (suc i) k neq (divides-refl (suc q)) = s≤s (m≤m+n i (q * suc i))

divides-zero : ∀ i → 0 ∣ i → i ≡ 0
divides-zero zero (divides q eq) = refl
divides-zero (suc i) (divides q eq) rewrite *-comm q 0 = eq

divides-trans : ∀ {i j k} → i ∣ j → j ∣ k → i ∣ k
divides-trans {i} (divides-refl q₁) (divides q₂ eq₂) = divides (q₂ * q₁) (trans eq₂ (sym (*-assoc q₂ q₁ i)))

divides-sub : ∀ {i j k} → i ∣ j → i ∣ k → i ∣ (j ∸ k)
divides-sub {i} (divides-refl q₁) (divides-refl q₂) = divides (q₁ ∸ q₂) (sym (*-distribʳ-∸ i q₁ q₂))

divides-one : ∀ i → i ∣ 1 → i ≡ 1
divides-one zero (divides (suc q) eq) rewrite *-comm q 0 = ⊥-elim (1+n≢0 eq)
divides-one (suc zero) (divides (suc q) eq) = refl

divides-not-one : ∀ {i} → i ∣ 1 → i ≢ 1 → ⊥
divides-not-one {zero} (divides q eq) neq rewrite *-comm q 0 = 1+n≢0 eq
divides-not-one {suc zero} (divides q eq) neq = neq refl
divides-not-one {2+ i} (divides zero eq) neq = 1+n≢0 eq
divides-not-one {2+ i} (divides (suc q) eq) neq = 1+n≢0 (sym (suc-injective eq))

lt-left-suc : ∀ i j → i ≤ j → i ≢ j → suc i ≤ j
lt-left-suc zero zero z≤n p = ⊥-elim (p refl)
lt-left-suc zero (suc j) z≤n p = s≤s z≤n
lt-left-suc (suc i) zero () p
lt-left-suc (suc i) (suc j) (s≤s x) p = s≤s (lt-left-suc i j x λ q → p (cong suc q))


not : {A : Set} → Dec A → Dec (¬ A)
not (yes p) = no (λ z → z p)
not (no ¬p) = yes ¬p

DNE : {A : Set} → Dec A → ¬ ¬ A → A
DNE (yes p) f = p
DNE (no ¬p) f = ⊥-elim (f ¬p)

Decide : {A : Set} (P : A → Set) → Set
Decide P = ∀ i → Dec (P i)

lt-lower : ∀ {i j} → suc i ≤ suc j → i ≢ j → suc i ≤ j
lt-lower {zero} {zero} (s≤s p) q = ⊥-elim (q refl)
lt-lower {zero} {suc j} (s≤s p) q = s≤s z≤n
lt-lower {suc i} {suc j} (s≤s p) q = s≤s (lt-lower p λ x → q (cong suc x))

lt-decide : ∀ i → (P : Nat → Set) (p? : ∀ n → n < i → Dec (P n)) → (∀ n → n < i → P n) ⊎ (Σ _ λ j → j < i × ¬ (P j))
lt-decide zero P p? = inj₁ λ _ ()
lt-decide (suc i) P p? with lt-decide i P h
  where
    h : (n : Nat) → n < i → Dec (P n)
    h n lt = p? n (s≤s (<⇒≤ lt))
... | inj₂ (a , b , c) = inj₂ (a , s≤s (≤-trans (n≤1+n a) b) , c)
... | inj₁ x with p? i ≤-refl
... | no ¬a = inj₂ (i , ≤-refl , ¬a)
... | yes a = inj₁ h
  where
    h : (n : Nat) → suc n ≤ suc i → P n
    h n lt with n ≟ i
    ... | yes refl = a
    ... | no ¬a = x n (lt-lower lt ¬a)

¬lt-decide : ∀ i → (P : Nat → Set) (p? : ∀ n → n < i → Dec (P n)) → (∀ n → n < i → ¬ P n) ⊎ (Σ _ λ j → j < i × (P j))
¬lt-decide i P p? with lt-decide i (λ x → ¬ (P x)) (λ n x → not (p? n x))
... | inj₁ x = inj₁ x
... | inj₂ (a , b , c) = inj₂ (a , b , DNE (p? a b) c)

record Prime (n : Nat) : Set where
  constructor isprime
  field
    gt1 : n > 1
    div : ∀ k → k ∣ n → k ≡ 1 ⊎ k ≡ n

open Prime

zero-prime : ¬ (Prime 0)
zero-prime ()

prime-not-zero : ∀ p → Prime p → p ≢ 0
prime-not-zero p x refl = zero-prime x

one-prime : ¬ (Prime 1)
one-prime (isprime (s≤s ()) div)

two-prime : Prime 2
two-prime .gt1 = s≤s (s≤s z≤n)
two-prime .div zero (divides (suc q) eq) rewrite *-comm q 0 = ⊥-elim (1+n≢0 eq)
two-prime .div (suc zero) (divides (suc q) eq) = inj₁ refl
two-prime .div (2+ zero) (divides (suc q) eq) = inj₂ refl

record Composite (n : Nat) : Set where
  constructor composite
  field
    p : Nat
    Pp : Prime p
    p<n : p < n
    p∣ : p ∣ n

not-prime-and-composite : ∀ p → Prime p → Composite p → ⊥
not-prime-and-composite _ (isprime gt2 div₁) (composite p (isprime gt3 div₂) p<n p∣) with div₁ p p∣
... | inj₁ refl = one-prime (isprime gt3 div₂)
... | inj₂ refl = 1+n≰n p<n

prime-or-composite : ∀ p → p > 1 → Prime p ⊎ Composite p
prime-or-composite (suc zero) (s≤s ())
prime-or-composite (2+ p) lt = <-rec _ h p
  where
    h : (x : Nat) →
         ({y : Nat} → suc y ≤ x → Prime (2+ y) ⊎ Composite (2+ y)) →
         Prime (2+ x) ⊎ Composite (2+ x)
    h x f with ¬lt-decide x (λ k → (2+ k) ∣ (2+ x)) (λ n nlt → 2+ n ∣? 2+ x)
    ... | inj₂ (ev₁ , ev₂ , ev₃) with f ev₂
    h x f | inj₂ (ev₁ , ev₂ , ev₃) | inj₁ prm = inj₂ (composite (2+ ev₁) prm (s≤s (s≤s ev₂)) ev₃)
    h x f | inj₂ (ev₁ , ev₂ , ev₃) | inj₂ (composite p Pp (s≤s p<n) p∣) =
        inj₂ (composite p Pp (s≤s (≤-trans p<n (≤-trans ev₂ (n≤1+n x)))) (divides-trans p∣ ev₃))
    h x f | inj₁ eq = inj₁ (isprime (s≤s (s≤s z≤n)) lemma)
      where
        lemma : (k : Nat) → k ∣ 2+ x → k ≡ 1 ⊎ k ≡ 2+ x
        lemma k dv with k ≟ 1 | k ≟ (2+ x)
        ... | no ¬a | yes refl = inj₂ refl
        ... | yes refl | no ¬b = inj₁ refl
        ... | yes refl | yes ()
        lemma zero dv | no ¬a | no ¬b rewrite divides-zero (2+ x) dv = inj₂ refl
        lemma (suc zero) dv | no ¬a | no ¬b = inj₁ refl
        lemma (2+ k) dv | no ¬a | no ¬b with suc k ≤? x
        ... | yes k≤x = ⊥-elim (eq k k≤x dv)
        ... | no ¬k≤x = ⊥-elim (¬k≤x sothen)
          where
            one : 2+ k ≤ 2+ x
            one = div-goes-into (2+ k) (2+ x) 1+n≢0 dv

            two : k ≤ x
            two with one
            ... | s≤s (s≤s a) = a

            also : k ≢ x
            also x = ¬b (cong suc (cong suc x))

            sothen : suc k ≤ x
            sothen = lt-left-suc k x two also

prod-list : List Nat → Nat
prod-list [] = 1
prod-list (x ∷ xs) = x * prod-list xs

prod-list-nz : ∀ xs → (∀ x → x ∈ xs → x > 0) → prod-list xs > 0
prod-list-nz [] f = s≤s z≤n
prod-list-nz (x ∷ []) f rewrite *-comm x 1 rewrite +-comm x 0 = f x (here refl)
prod-list-nz (x ∷ x₁ ∷ xs) f = h {x} (f x (here refl)) (prod-list-nz (x₁ ∷ xs) (λ x₂ z → f x₂ (there z)))
  where
    h : ∀ {i j} → 1 ≤ i → 1 ≤ j → 1 ≤ i * j
    h {suc i} {suc j} (s≤s a) (s≤s b) = s≤s z≤n

prod-list-NZ : ∀ xs → (∀ x → x ∈ xs → x > 0) → NonZero (prod-list xs)
prod-list-NZ xs f = >-nonZero (prod-list-nz xs f)

prod-list-lt : ∀ xs → (∀ x → x ∈ xs → x > 0) → ∀ y → y ∈ xs → y ≤ prod-list xs
prod-list-lt [] f y ()
prod-list-lt (x ∷ xs) f y (here refl) = m≤m*n x (prod-list xs) ⦃ prod-list-NZ xs λ x₂ z → f x₂ (there z) ⦄
prod-list-lt (x ∷ xs) f y (there py) = also (prod-list-lt xs (λ x₂ z → f x₂ (there z)) y py) (f x (here refl))
  where
    lt+ : ∀ {i j k} → i ≤ j → i ≤ j + k
    lt+ z≤n = z≤n
    lt+ (s≤s x) = s≤s (lt+ x)

    also : ∀ {i j k} → i ≤ k → 1 ≤ j → i ≤ j * k
    also a (s≤s z≤n) = lt+ a

prod-list-divides : ∀ xs x → x ∈ xs → x ∣ prod-list xs
prod-list-divides [] x ()
prod-list-divides (y ∷ xs) x (here refl) = divides (prod-list xs) (*-comm y (prod-list xs))
prod-list-divides (y ∷ xs) x (there p) with prod-list-divides xs x p
... | divides q eq rewrite eq = divides (y * q) (sym (*-assoc y q x))

prime-not-in : ∀ xs → (∀ x → x ∈ xs → x > 0) → Σ _ λ k → k ∉ xs × Prime k
prime-not-in xs gt with 2 ≤? prod-list xs
... | no ¬a = 2 , lemma , two-prime
  where
    lemma : 2 ∈ xs → ⊥
    lemma pf with prod-list-lt xs gt 2 pf
    ... | also = ¬a also
... | yes a with prime-or-composite (suc (prod-list xs)) (s≤s (≤-trans (s≤s z≤n) a))
... | inj₁ x = suc (prod-list xs) , lemma , x
  where
    lemma : (suc (prod-list xs)) ∈ xs → ⊥
    lemma pf with prod-list-lt xs gt (suc (prod-list xs)) pf
    ... | also = 1+n≰n also
... | inj₂ (composite p Pp p<n p∣) with p ∈? xs
... | no ¬b = p , ¬b , Pp
... | yes b with prod-list-divides xs p b
... | divs with divides-sub p∣ divs
... | sothen = ⊥-elim (lemma {p} {prod-list xs} sothen λ{ refl → one-prime Pp })
  where
    suc-sub : ∀ i → suc i ∸ i ≡ 1
    suc-sub zero = refl
    suc-sub (suc i) = suc-sub i

    lemma : ∀ {i j} → i ∣ (suc j ∸ j) → i ≢ 1 → ⊥
    lemma {_} {j} div neq rewrite suc-sub j = divides-not-one div neq


list-up-to : Nat → List Nat
list-up-to 0 = 1 ∷ []
list-up-to (suc n) = suc n ∷ list-up-to n

list-up-to-nz : ∀ k x → x ∈ list-up-to k → x > 0
list-up-to-nz zero x (here refl) = s≤s z≤n
list-up-to-nz (suc k) x (here refl) = s≤s z≤n
list-up-to-nz (suc k) x (there p) = list-up-to-nz k x p

list-up-to-contains : ∀ i j → j ≤ i → j ≢ 0 → j ∈ list-up-to i
list-up-to-contains zero zero z≤n b = ⊥-elim (b refl)
list-up-to-contains zero (suc j) () b
list-up-to-contains (suc i) zero z≤n b = ⊥-elim (b refl)
list-up-to-contains (suc i) (suc j) (s≤s a) b with suc i ≟ suc j
... | no ¬c = there (list-up-to-contains i (suc j) (lt-left-suc j i a λ x → ¬c (cong suc (sym x))) b)
... | yes refl = here refl


infinite-primes : ∀ i → Σ _ λ p → p > i × Prime p
infinite-primes zero = 2 , s≤s z≤n , two-prime
infinite-primes (suc i) with prime-not-in (list-up-to (2+ i)) (list-up-to-nz (2+ i))
... | p , notin , Pp = p , DNE (2+ i ≤? p) lemma , Pp
  where
    notzero : p ≢ 0
    notzero = prime-not-zero p Pp

    implies : ∀ {a b} → (suc a ≤ b → ⊥) → (a ≡ b) ⊎ (a > b)
    implies {a} {b} x with compare a b
    ... | less .a k = ⊥-elim (x (s≤s (m≤m+n a k)))
    ... | equal .a = inj₁ refl
    ... | greater .b k = inj₂ (s≤s (m≤m+n b k))

    lemma : ((2+ i ≤ p) → ⊥) → ⊥
    lemma x with implies x
    ... | inj₁ refl = notin (there (here refl))
    ... | inj₂ (s≤s a) = notin (list-up-to-contains (2+ i) p (≤-trans a (≤-trans (n≤1+n i) (n≤1+n (suc i)))) (prime-not-zero p Pp))
```

</details>
