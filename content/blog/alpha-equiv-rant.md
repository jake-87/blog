+++
title = "Alpha equivalence is Evil (rant)"
date = 2025-03-29
+++

Preface: This is *mostly* in the context of formalization, but it also totally applies to general language creation. Many things (e.g. common subexpression elimination)
require dealing with a version of alpha equivalence in named systems! (also: this is a rant. don't expect nuaced discussion)

Two terms, for example: \\(\lambda x . x\\) and \\(\lambda y . y\\) are considered "alpha equivalent" (essentially) if they are the same up to renaming bound names.

Alpha equivalence is an awful awful property. Trying to create a formalisation that can easily work with alpha equivalence is miserable; you have few good options, and
even for your good options, they are still pretty sucky (e.g. converting to de-brujin and doing structural comparison from there - how do you handle free variables?).

If at all possible, never do named representations, because then you have to deal with alpha equivalence. Nobody should be subjected to that fate.

Please just use one of the *many* other good options available, like de-brujin.

This has been a health and safety PSA. Please spread appropriately, in order to better protect your community from alpha equivalence.
