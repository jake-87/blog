+++
title = "A short excerpt on pattern unification"
date = 2025-04-30
+++

Preface: Low effort post.

The following is an excerpt from a discord conversation about dependent pattern unification. In short, pattern unification is one method for solving
unification goals such as the following:

```haskell
id : (A : Type) -> A -> A
id A x = x

id2 : (B : Type) -> B -> B
id2 B x = id _ x
```

Here, we have that our type argument to `id` is a hole, so we must solve it. 

We are working with a de Brujin representation, so the above actually looks something like:
```haskell
let : (A : Type) -> A -> A = λ λ 0;
let : (B : Type) -> B -> B = λ λ 2 _ 0
```

Once we've gotten to the point of writing that hole, what was once `id` is now `2` (Two levels of lambda binders in the way.). As such, performing this
solving takes slightly more work, as we can't simply plug in the result of unification - the de Brujin wouldn't line up.

In this conversation, we were discussing relative to András Kovács' excellent "elab-zoo", <a href="https://github.com/AndrasKovacs/elaboration-zoo/">found here</a>.
We are discussing <a href="https://github.com/AndrasKovacs/elaboration-zoo/blob/master/03-holes/pattern-unification.txt">`03-holes/pattern-unification.txt`</a> and <a href="https://github.com/AndrasKovacs/elaboration-zoo/blob/master/03-holes/Main.hs">`/03-holes/Main.hs`</a> in particular, the former being a formal treatment of what happens *after* you find a solution, and the latter being an implementation of pattern unification (amongst other things).

### Comments before the fact

I think this explanation is somewhat reasonable for what's actually happening. Nothing will replace the original document, of course! Go give that a read (preferably before you read this.)

## Warning!

This conversation is *only* going to make sense if you've had a look at at *least* the latter linked file above (`Main.hs`). I recommend also having a look at `pattern-unification.txt` if you really want to grasp it, though.

### The conversation as written, minorly edited to exclude comments from people other than me

(In reference to `pattern-unification.txt`)

> most of this is what you do *after* you've "found the solution"
> finding the "solution" is vaguely your normal unification procedure
> when you find a hole, you put a meta in it, when you unify something with that meta you "solve" it

"Normal unification" here is your fairly traditional unification for typechecking; it is somewhat different in a dependent setting, but not
too different. (Look here later for a series on this!)

All of the following is then discussing `Main.hs` and the concrete implementation of solving a metavariable.
This maps somewhat closely from `pattern-unification.txt`, but can be understood standalone.

> Andras' puts it better than i could:

```hs
{-
The actual unification algorithm is fairly simple, it works in a structural
fashion, like conversion checking before.  The main complication, as we've seen,
is the pattern unification case.
-}
```

*A small break while I write the following*

```hs
  (VFlex m sp , t'           ) -> solve l m sp t'
  (t          , VFlex m' sp' ) -> solve l m' sp' t
```

> `VFlex` is what elab-zoo calls the thing holding an unsolved metavariable (here `m` or `m'`), and a "spine" `sp`, which is a list of everything bound in the context when `m` was created

> you can then imagine the meta to be, essentially, a function:

```hs
id : (A : U) -> A -> A
id A x = x

foo B y = id _ y
=>
?m B y = ???
foo B y = id (?m B y) y
```

> here our normal typechecking would try to unify `?m`'s `???` with `B`

> the actual solving is then handled by `solve`:

```hs
--       Γ      ?α         sp       rhs
solve :: Lvl -> MetaVar -> Spine -> Val -> IO ()
solve gamma m sp rhs = do
  pren <- invert gamma sp
  rhs  <- rename m pren rhs
  let solution = eval [] $ lams (dom pren) rhs
  modifyIORef' mcxt $ IM.insert (unMetaVar m) (Solved solution)
```
> `B` would be our `rhs` here

> you need to remember we're working with debrujin, so the solution to our metavariable (which may reference things in the context of where the meta was created, of course) should exist in a different context (a "mostly" empty one, with only the arguments to our meta). 

> `invert`, which is `^⁻¹` in the pattern unification text, creates a renaming from our original context (`sp`) to the desired new context, with only our meta arguments

> `rename` takes our `rhs` and uses the renaming to take it to the new context (it takes `m` as an argument because it does an occurs check at the same time, recursive solutions are no good)

> finally we add the needed lambdas for our function meta (debrujin, remember), evaluate it in the empty context in order to take it into the value domain, and plug it in as the solution


### Comments after the fact

I don't really have any other comments. There will (hopefully shortly) start appearing a "tutorial" sort of series on writing a dependent language with holes, so some of this
will definitely come up in that, and I'll probably write some more comments here after i've gone and written that tutorial.

Hope this helps somebody!
