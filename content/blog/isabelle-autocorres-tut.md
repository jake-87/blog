+++
title = "Verifying (simple) C in Isabelle/HOL with AutoCorres"
date = 2026-08-28
+++

# Verifying (simple) C in Isabelle/HOL with AutoCorres

This post details the first steps of verifying a C function in Isabelle/HOL using AutoCorres. It'll go over the basic setup for Isabelle and AutoCorres, what AutoCorres gives you, and how we can verify some basic properties of a function (here, the sum of a list.) I use the latest versions of Isabelle and AutoCorres available at time of writing (24/08/26).

# Caveats

This is _not_ official documentation for AutoCorres, nor may it be 100% correct in all places. All the proofs go through, but I do not work on AutoCorres, nor have I used it professionally; most of my experience is hobby verification. However, I have found resources on it are woefully lacking, so I wished to introduce some more. Much of this information has been gleaned from the official documentation (which can be found at the aforementioned,) and <a href="https://cgi.cse.unsw.edu.au/~cs4161/">this course</a>. When it runs, the slides/similar may be removed for some time - they exist on the internet archive also.

# Assumed knowledge

This article assumes a little either Isabelle/HOL or general verification knowledge, but I try to explain wherever feasible. A bit of C knowledge is required as well, and so is a little knowledge about program verification - a little Hoare logic, and the like. I'll try to explain as much as I can without being excessively verbose, and much of it is very searchable.

# Installing Isabelle

You can find Isabelle <a href="https://isabelle.in.tum.de/">here</a>. Install as is appropriate for your platform. You can decide whether to put `isabelle` in your path or not. 

# Installing AutoCorres

First, pick a directory for your project. It may be feasible to use AutoCorres globally, but I wouldn't recommend it for versioning reasons. Then, AutoCorres can be found by scrolling down <a href="https://trustworthy.systems/projects/OLD/autocorres/">here</a>. (It will probably take you <a href="https://github.com/seL4/l4v/releases">here</a>, whereupon you should scroll down to the latest AutoCorres release). You only need the AutoCorres download, as it bundles the C parser. Take that file, and extract it in your directory of choice.

Then we need to build AutoCorres. From the directory where you unpacked it, run 

```bash
$ L4V_ARCH=X64 <path-to-isabelle> build -v -b -o document=false -d autocorres-1.12 AutoCorres
```

This'll take a second.

Replace paths as appropriate. Note that `L4V_ARCH` is _not_ the architecture you are on - it determines how the tool translates various C sizes into Isabelle. This will need to be the same architecture you launch with later. There are some others, and on a related note:

# Look at the AutoCorres README!

It may not all make sense yet, but it may answer some questions.

# Launching Isabelle with AutoCorres

I recommend making a little shell script for this step (`launch.sh` perhaps.) You'll need to invoke Isabelle in a manner similar to the following:

```bash
#!/bin/sh
L4V_ARCH=X64 /Applications/Isabelle2025.app/bin/isabelle jedit -d autocorres-1.12 -l AutoCorres my_theory.thy
```

Remember that `L4V_ARCH` needs to be the same as earlier.
    
Now that we have Isabelle running, we can use AutoCorres to generate Isabelle versions of C files. The manner in which this is performed is long, complex, and interesting - I recommend a rabbit hole evening - but in short, the C parser first translates it to a deep embedding in a language called Simpl, and then AutoCorres takes this Simpl representation and turns it into a monadic shallow embedding best it can.

# Some C to verify

Here's what we'll be verifying today:

```c
unsigned int list_sum(unsigned int *list, unsigned int length) {
    unsigned int sum = 0;
    
    for (unsigned int i = 0; i < length; i++) {
        sum += list[i];
    }
    
    return sum;
}
```
The use of unsigned will be expanded on later.

We'll put it in a C file named `list_sum.c`.

# Verifying it

At this point, your directory should look something like
```
autocorres-1.12
my_theory.thy
list_sum.c
launch.sh
```

or similar.

Begin a regular Isabelle theory, importing AutoCorres (and whatever else you want):

```ocaml
theory my_theory
  imports Main "AutoCorres.AutoCorres"
begin

end
```

We then need to let AutoCorres perform its magic. First, we "install" the C file with the C parser:

```ocaml
external_file "sum_list.c"
install_C_file "sum_list.c"
```

You can use `print_theorems` to see what this defines. Of particular interest is `list_sum_body_def`. Then, AutoCorres:

```ocaml
autocorres [skip_word_abs] "list_sum.c"
```

## Tangent: What's going on?

The idea the C parser and AutoCorres use for verifying C is that it is reasonable to do a very direct translation of C to Simpl, and then a _refinement_ to the monadic representation. The C parser is correct through inspection, and extensive testing. However, the translation of Simpl to the monadic form is verified - there exist Isabelle-checkable proofs that show that the monadic representation, however different it may be, behaves identically to the Simpl equivalent. This makes the monadic form a refinement of Simpl, and is why the things we prove about the monadic forms translate back down to C.

## Back to AutoCorres

This might also take a second.

We choose to use `skip_word_abs`. This makes unsigned integers perform modular arithmetic instead of using overflow checks; this has positives and negatives. Read the README for more.

Again, you can use `print_theorems` to see what this gives you. Of particular interest is `list_sum'_def`. This is the monadic embedding of our function.

We then need to enter the locale (think of it as an environment) defined by the C parser and AutoCorres, so your file should look something like:

```ocaml
theory my_theory
  imports Main "AutoCorres.AutoCorres"
begin


external_file "list_sum.c"
install_C_file "list_sum.c"

autocorres [skip_word_abs] "list_sum.c"

context list_sum begin

(* See below. *)
thm list_sum_body_def
thm list_sum'_def

term heap_w32
term heap_w32_update

end

end
```

We do our work in this locale. If you haven't already, `term list_sum_body` and `term list_sum'`. The former is the deep embedding produced by the C parser, and the latter is the result of AutoCorres's shallow embedding. These can be unfolded with `list_sum_body_def` and `list_sum'_def` respectively. You can examine the types of things with ctrl-hover (cmd-hover on Macs.) 

Also examine `term heap_w32`. The type for the monadic state used by AutoCorres is called `lifted_globals` (You may see it displayed as `'a lifted_globals_scheme` in some places.) It's a record containing fields for each type of pointer used; ours only uses `unsigned int *`, so it only contains information for 32 bit words. We can examine the extract and update functions used in `list_sum'_def` with `term heap_w32` and `term heap_w32_update`.

Which monad AutoCorres chooses to embed a function into depends on the function, and can also be configured. This function is simple enough that it can be encoded with purely `option`, but others include `pure`, `gets` (option with state,) and `nondet`. 

# Tangent: What does it mean to verify something?

Yes, this is a reasonable question to ask. What does it actually mean to verify this function? Generally, there are a few reasons to verify something:
1. To prove it never "fails" (what failing means is another question)
2. To prove it produces some desired output
3. To prove it holds some desired property

We'll look at all three.

## On level-of-depth 

The not failing example will go into quite a lot of depth, whereas the correctness example will go into much less, only covering broad strokes. This is because the proofs are quite similar, and if you find the former excessively verbose, you might want to skip to the latter.

# Not failing (good)

Let's consider what we need for this function to not fail in C. The obvious constraint is that `list[i]` must be defined for all `0 <= i < length`. Using `term is_valid_w32 :: "lifted_globals ⇒ 32 word ptr ⇒ bool" `, we can state this as a definition:

```ocaml
definition list_defined_to :: "lifted_globals ⇒ 32 word ptr ⇒ 32 word ⇒ bool" where
    "list_defined_to s list len ≡ ∀i. 0 ≤ i ∧ i < len ⟶ is_valid_w32 s (list +⇩p uint i)"
```

Notes:
1. `list` is a `32 word ptr`, as perhaps expected. `len` has been converted into a `32 word`.
2. `s` is our state for the function, and carries information as mentioned about the heap.
3. We use the perhaps confusing ` +⇩p ` for pointer addition. You can write this in jedit with `+ \<^sub> <enter> p`. We also need an explicit type conversion `uint i`, as you can add negatives to a pointer, so the argument is an `uint` (this is _not_ unsigned int; that's `unat`. `int` here corresponds to Isabelle `int`, which is signed.)

Now we have our suitable precondition, let's set up our "doesn't fail" lemma. We expect that:
- If:
  - The list is defined properly 
  - Some other property Q is true
- Then:
  - Our function does return successfully, so
  - That property is still true.
  
We can state this using the combinator `ovalidNF`. The NF stands for `no fail`, and it adds the additional condition that our program does not fail in some way during execution. Precisely what we want! It takes three arguments:

```ocaml
term ovalidNF
(*
"ovalidNF"
  :: "('a ⇒ bool) ⇒ ('a ⇒ 'b option) ⇒ ('b ⇒ 'a ⇒ bool) ⇒ bool"
*)
```
The arguments are:
1. A precondition function.
2. A computation function. 
3. A postcondition function.

Here, `'b option` is used because of the choice of state monad AutoCorres makes. Note that our postcondition takes both the state `'a` and the return value `'b`. When we use `ovalidNF` with `list_sum'`, `'a` will be our state `lifted_globals`. So, our lemma then becomes:

```ocaml
lemma list_sum_no_fail: "ovalidNF (λs. list_defined_to s list len ∧ Q s)
                                  (list_sum' list len)
                                  (λret s. Q s)"
```

If we have our list defined properly and some property Q, and we run our program, then it does not fail and Q is still true.

## Proof time! (detailed)

If you're familiar with Hoare logic, you'll know we probably want to use some sort of weakest precondition reasoning. A weakest precondition is roughly "what is the smallest amount of information we need to know for this to be true", which allows us to simplify our proof obligations. Indeed, AutoCorres provides us with a family of tactics such as `wp` and `wpsimp`. However, I find it nice to start these proofs by unfolding the function at hand, and applying `auto` or similar to get some simplification going. Then we can apply `wp` to apply relevant weakest precondition rules automatically.

```ocaml
  apply (unfold list_sum'_def)
  apply auto
  apply wp
```

This should leave you with a state something like:

```ocaml
proof (prove)
goal (1 subgoal):
 1. ovalidNF (λs. list_defined_to s list len ∧ Q s) (owhile (λ(i, sum) s. i < len) (λ(i, sum). do {
       y ← oguard (λs. is_valid_w32 s (list +⇩p uint i));
       sum ← ogets (λs. sum + heap_w32 s (list +⇩p uint i));
       oreturn (i + 1, sum)
     })
                                                       (0, 0))
     (λr. case r of (x, y) ⇒ Q)
```

I _highly_ recommend using the Query tab of jedit throughout (or equivalents like `find_theorems`.) `name: foo` will come in handy, and it's a nice fuzzy search. `find_theorems solves` will do what's on the tin, and `find_theorem intro` will find theorems that could apply.

As we have a while-loop, a reasonable step is to add an invariant. An invariant is something that is _invariant_ over the loop - it is always true, at the top of every loop cycle, and right after the loop finishes. This is how we conclude things about what a loop does. Indeed, we can see a theorem of use:
```ocaml
Reader_Option_VCG.owhile_add_inv: owhile ?C ?B ?x = owhile_inv ?C ?B ?x ?I ?M
```
Most of the time, the prefixes can be omitted.

So first, we add an invariant, and then we can use `wp` to transform our `owhile_inv` into theorems we can work with.

The invariants we care about right now are: 
1. ` i ≤ len` (for termination)
2. `list_defined_to s list len` (for no-failure)
3. `Q s` (for simplified correctness)

We also care that this loop terminates, so let's add a suitable measure.

```ocaml
  apply (subst owhile_add_inv[where I="λ(i, sum) s. i ≤ len ∧ list_defined_to s list len ∧ Q s"
                                and M="λ(i, sum) s. unat (len - i)"])
```

`I` is our invariant, and `M` is the termination measure for the loop. Note the type conversion in `M`. Then, `wp` again:

```ocaml
  apply wp
```

This'll probably give you a few more normal looking goals. `apply auto` can come in handy again to chunk these down.

This leaves me with:

```ocaml
proof (prove)
goal (3 subgoals):
 1. ⋀a s. ⟦a < len; a ≤ len; list_defined_to s list len; Q s⟧ ⟹ is_valid_w32 s (list +⇩p uint a)
 2. ⋀a s. ⟦a < len; a ≤ len; list_defined_to s list len; Q s⟧ ⟹ a + 1 ≤ len
 3. ⋀a b m.
       ⦉λs. a ≤ len ∧ list_defined_to s list len ∧ Q s ∧ a < len ∧ unat (len - a) = m⦊ do {
         y ← oguard (λs. is_valid_w32 s (list +⇩p uint a));
         sum ← λc. Some (b + heap_w32 c (list +⇩p uint a));
         λb. Some (a + 1, sum)
       } ⦉λr s. (case r of (i, sum) ⇒ λs. unat (len - i)) s < m⦊
```

The first we talked about earlier - we can solve it by unfolding `list_defined_to` via `list_defined_to_def`, and basic reasoning.

```ocaml
   apply (simp add: list_defined_to_def)
```

For the second, it seems obvious - why hasn't `auto` solved it? (If it were on `nat`s, it certainly would have.) Alas, it's on `32 word`s, which as we have chosen to use modular arithmetic, are slightly less nice. It's hard to search for theorems involving `+` as it's so overloaded, but luckily here `find_theorems solves` finds `Word.inc_le: ?i < ?m ⟹ ?i + 1 ≤ ?m`.
```ocaml
   apply (simp add: inc_le)
```

The final goal is more interesting. The initial intuition might be to use `wp` again, but this leaves us with nasty metavariables because of the chaining nature of the binds, and the fact that AutoCorres here is slightly too general. With a little searching we have the following:

```ocaml
  Reader_Option_VCG.obind_wp: ⟦⋀r. ⦉?R r⦊ ?g r ⦉?Q⦊; ⦉?P⦊ ?f ⦉?R⦊⟧ ⟹ ⦉?P⦊ ?f >>= ?g ⦉?Q⦊
```
But we only really need `?R` to be identical to `?P`. We could instantiate `?R` manually each time, or we can make a little helper lemma (which I will do.) Above this lemma, we add

```ocaml
lemma obind_wp_weak: " ⟦⋀r. ⦉P⦊ g r ⦉Q⦊; ⦉P⦊ f ⦉λs. P⦊⟧ ⟹ ⦉P⦊ obind f g ⦉Q⦊"
  by (erule obind_wp, assumption)
```

Then we can apply `obind_wp_weak` twice (There are two binds.)

```ocaml
  apply (rule obind_wp_weak)+
```

We hence have:

```ocaml
proof (prove)
goal (3 subgoals):
 1. ⋀a b m r ra.
       ⦉λs. a ≤ len ∧ list_defined_to s list len ∧ Q s ∧ a < len ∧ unat (len - a) = m⦊ 
       λb. Some (a + 1, ra) 
       ⦉λr s. (case r of (i, sum) ⇒ λs. unat (len - i)) s < m⦊
 2. ⋀a b m r.
       ⦉λs. a ≤ len ∧ list_defined_to s list len ∧ Q s ∧ a < len ∧ unat (len - a) = m⦊ 
       λc. Some (b + heap_w32 c (list +⇩p uint a)) 
       ⦉λs sa. a ≤ len ∧ list_defined_to sa list len ∧ Q sa ∧ a < len ∧ unat (len - a) = m⦊
 3. ⋀a b m.
       ⦉λs. a ≤ len ∧ list_defined_to s list len ∧ Q s ∧ a < len ∧ unat (len - a) = m⦊ 
       oguard (λs. is_valid_w32 s (list +⇩p uint a)) 
       ⦉λs sa. a ≤ len ∧ list_defined_to sa list len ∧ Q sa ∧ a < len ∧ unat (len - a) = m⦊
```

It's finally time to start unfolding the definition of `ovalid` so we can crunch down these last few goals. If we hit the first goal with:

```ocaml
    apply (simp add: ovalid_def, clarsimp)
```

we're left with

```ocaml
 1. ⋀a s. ⟦a ≤ len; list_defined_to s list len; Q s; a < len⟧
          ⟹ unat (len - (a + 1)) < unat (len - a)
```

which `sledgehammer` takes out nicely.

```ocaml
    apply (metis diff_diff_eq gt0_iff_gem1 less_diff_gt0 unat_mono)
```

Tip: I fiddled with this for a bit manually, but if it looks fiddly and obvious, there's a good chance sledgehammer can do it.

Finally, the last two are easy:
```ocaml
  by (simp add: ovalid_def)+
```

We have a proof of non-failure! This is very exciting.

The final proof is as follows:

```ocaml
lemma obind_wp_weak: " ⟦⋀r. ⦉P⦊ g r ⦉Q⦊; ⦉P⦊ f ⦉λs. P⦊⟧ ⟹ ⦉P⦊ obind f g ⦉Q⦊"
  by (erule obind_wp, assumption)

lemma list_sum_no_fail: "ovalidNF (λs. list_defined_to s list len ∧ Q s)
                                  (list_sum' list len)
                                  (λret s. Q s)"
  apply (unfold list_sum'_def)
  apply auto
  apply wp
  apply (subst owhile_add_inv[where I="λ(i, sum) s. i ≤ len ∧ list_defined_to s list len ∧ Q s"
                                and M="λ(i, sum) s. unat (len - i)"])
  apply wp
     apply auto
     apply (simp add: list_defined_to_def)
   apply (simp add: inc_le)
  apply (rule obind_wp_weak)+
    apply (simp add: ovalid_def, clarsimp)
    apply (metis diff_diff_eq gt0_iff_gem1 less_diff_gt0 unat_mono)
  by (simp add: ovalid_def)+
```


# Correctness (also good)

Then, what does it mean for our function to be correct? Well, a reasonable definition is that it produces the output we expect. We then need to figure out what "output we expect" means.

We could have also defined the sum of a list as similar to the following recursive function:

```ocaml
function list_sum_spec :: "lifted_globals ⇒ 32 word ptr ⇒ 32 word ⇒ 32 word ⇒ 32 word" where
"list_sum_spec s list len i = (if i = 0 then 0
                               else heap_w32 s (list +⇩p uint (i - 1))
                                    + list_sum_spec s list len (i - 1))"
  (* The pattern matching is trivially complete. *)
  by pat_completeness auto
```

Note that we explicitly check for 0 to ensure the recursion terminates (We can't pattern match on 0/Suc, as we're working with `32 word`s.). Unfortunately, this function's termination can't be proven automatically due to the use of a `32 word`, which Isabelle isn't as good with as its own `nat`s. This is why we use a `function` instead of `fun`, and we must do the termination proof ourselves:

```ocaml
termination
  (* Note that the measure is just i, as it's decreasing. *)
  apply (rule local.termination[where R="measure (λ(_,_,len,i). unat i)"])
   apply blast
  apply (unfold measure_def)
  by (simp add: measure_unat)
```

We also delete `list_sum_spec.simps` from the default simp set because it seems to cause some solvers to loop.

```ocaml
declare list_sum_spec.simps [simp del]
```

We can set up our lemma like before, but this time, we use the `ret` parameter:

```ocaml
lemma list_sum_correct: "ovalid (λs. list_defined_to s list len)
                                (list_sum' list len)
                                (λret s. list_sum_spec s list len len = ret)"
```

We want the result to be equivalent to summing the entire list with our recursive `spec` function. We also don't bother to prove that it doesn't fail here.

We can begin as before, omitting the `auto` step. We then need to annotate with a suitable invariant. A hint: We want the sum _at the end_ to be correct, so a good invariant should capture correctness _at every step_, which gives us full correctness when the loop finishes.

```ocaml
  apply (subst owhile_add_inv[where I="λ(i, sum) s. i ≤ len ∧ list_defined_to s list len
                                                            ∧ list_sum_spec s list len i = sum"
                                and M="λ(i, sum) s. unat (len - i)"])
```

We have to manually `subst list_sum_spec.simps` a few times as we deleted it from the simp set, but otherwise the proof is very straightforward. Mine came out to be:

```ocaml
function list_sum_spec :: "lifted_globals ⇒ 32 word ptr ⇒ 32 word ⇒ 32 word ⇒ 32 word" where
"list_sum_spec s list len i = (if i = 0 then 0
                               else heap_w32 s (list +⇩p uint (i - 1))
                                    + list_sum_spec s list len (i - 1))"
  by pat_completeness auto
termination
  apply (rule local.termination[where R="measure (λ(_,_,len,i). unat i)"])
   apply blast
  apply (unfold measure_def)
  by (simp add: measure_unat)

declare list_sum_spec.simps [simp del]

lemma list_sum_correct: "ovalid (λs. list_defined_to s list len)
                                (list_sum' list len)
                                (λret s. list_sum_spec s list len len = ret)"
  apply (unfold list_sum'_def)
  apply wp
  apply (subst owhile_add_inv[where I="λ(i, sum) s. i ≤ len ∧ list_defined_to s list len
                                                            ∧ list_sum_spec s list len i = sum"
                                and M="λ(i, sum) s. unat (len - i)"])
  apply wp
    apply auto
     apply (simp add: inc_le)
   apply (subst list_sum_spec.simps)
   apply (simp add: less_is_non_zero_p1)
   apply (subst list_sum_spec.simps)
  by simp
```

We could take this one step further if we wanted, and define a bijection between `32 word list` and `32 word ptr` assuming our precondition, and then also show that our own spec is equivalent to the sum of that list. That'd give us even more confidence our function is correct. I'm personally pretty convinced, but you're welcome to try this yourself!

# Properties (also good!)

We could then finally prove that if some property is true for the sum of a list, it's true for the result of our program. I won't detail this one; it should follow reasonably easily from the former two.

# Conclusion 

So, what have we (hopefully) learnt? 
1. How to install and setup Isabelle and AutoCorres
2. What it means to prove things about programs
3. How to set up appropriate proofs
4. How to prove them 
5. What tools we have available and how we can search for more

I hope this has been informative!

# The whole file

`list_sum.c`:
```c
unsigned int list_sum(unsigned int *list, unsigned int length) {
    unsigned int sum = 0;
    
    for (unsigned int i = 0; i < length; i++) {
        sum += list[i];
    }
    
    return sum;
}
```


`my_theory.thy` (With less fixing of indentation for the web, sorry):
```ocaml
theory my_theory
  imports Main "AutoCorres.AutoCorres"
begin


external_file "list_sum.c"
install_C_file "list_sum.c"

autocorres [skip_word_abs] "list_sum.c"

context list_sum begin

definition list_defined_to :: "lifted_globals ⇒ 32 word ptr ⇒ 32 word ⇒ bool" where
    "list_defined_to s list len ≡ ∀i. 0 ≤ i ∧ i < len ⟶ is_valid_w32 s (list +⇩p uint i)"

lemma obind_wp_weak: " ⟦⋀r. ⦉P⦊ g r ⦉Q⦊; ⦉P⦊ f ⦉λs. P⦊⟧ ⟹ ⦉P⦊ obind f g ⦉Q⦊"
  by (erule obind_wp, assumption)

lemma list_sum_no_fail: "ovalidNF (λs. list_defined_to s list len ∧ Q s) (list_sum' list len) (λret s. Q s)"
  apply (unfold list_sum'_def)
  apply auto
  apply wp
  apply (subst owhile_add_inv[where I="λ(i, sum) s. i ≤ len ∧ list_defined_to s list len ∧ Q s"
                                and M="λ(i, sum) s. unat (len - i)"])
  apply wp
     apply auto
     apply (simp add: list_defined_to_def)
   apply (simp add: inc_le)
  apply (rule obind_wp_weak)+
    apply (simp add: ovalid_def, clarsimp)
    apply (metis diff_diff_eq gt0_iff_gem1 less_diff_gt0 unat_mono)
  by (simp add: ovalid_def)+

function list_sum_spec :: "lifted_globals ⇒ 32 word ptr ⇒ 32 word ⇒ 32 word ⇒ 32 word" where
"list_sum_spec s list len i = (if i = 0 then 0 else heap_w32 s (list +⇩p uint (i - 1)) + list_sum_spec s list len (i - 1))"
  by pat_completeness auto
termination
  apply (rule local.termination[where R="measure (λ(_,_,len,i). unat i)"])
   apply blast
  apply (unfold measure_def)
  by (simp add: measure_unat)

declare list_sum_spec.simps [simp del]

lemma list_sum_correct: "ovalid (λs. list_defined_to s list len) (list_sum' list len) (λret s. list_sum_spec s list len len = ret)"
  apply (unfold list_sum'_def)
  apply wp
  apply (subst owhile_add_inv[where I="λ(i, sum) s. i ≤ len ∧ list_defined_to s list len ∧ list_sum_spec s list len i = sum"
                                and M="λ(i, sum) s. unat (len - i)"])
  apply wp
    apply auto
     apply (simp add: inc_le)
   apply (subst list_sum_spec.simps)
   apply (simp add: less_is_non_zero_p1)
   apply (subst list_sum_spec.simps)
  by simp

end

end
```

A bit underwhelming for how long it took to explain, perhaps!
