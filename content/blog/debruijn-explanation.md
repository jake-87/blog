+++
title = "Debruijn indexes + levels, and why they're handy"
date = 2025-05-26
+++

<script>
 function r(str) {
while (str.charAt(0) == ' ') {
  	str = str.slice(1);
  }
  return str;
}

function get_args(str) {
 	str = r(str);
  if (str == "") {
  	return ["", []]
  }
  else if (str.charAt(0) == '.') {
  	return [str.slice(1), []];
  }
  let [rest, l] = get_args(str.slice(1));
  return [rest, [str.charAt(0)].concat(l)];
}

function parse_app(str) {
	str = r(str);
  if (str == "") {
  	return ["", []];
  }
  else if (str.charAt(0) == '(') {
  	let [rest, l] = parse(str.slice(1));
    let [rest1, k] = parse_app(rest);
    return [rest1, [l].concat(k)];
  }
  else if (str.charAt(0) == ')') {
  	return [str.slice(1), []]
  } else {
  	let c = str.charAt(0);
    let [rest, l] = parse_app(str.slice(1));
    return [rest, [c].concat(l)];
  }
}

function parse(str) {
	str = r(str);
  if (str == "") {
  	return ["", {}]
  }
  else if (str.charAt(0) == '\\') {
  	let [rest, a] = get_args(str.slice(1));
    let [rest1, bd] = parse_app(rest);
    return [rest1, {args : a, bd}]
  } else {
  	return parse_app(str);
  }
}

function iso(x) {
return typeof x === 'object' && !Array.isArray(x) && x !== null;
}

function islist(x) {
return (x.constructor.name == "Array")
}

function helper(args, p) {
let r = [];
for (let l of p) {
  if (iso(l)) {
  	r.push(debru_h(args, l));
  } else if (islist(l)) {
  	r.push(helper(args, l));
  }
  else {
    	r.push(args.indexOf(l));
  }
  }
  return r;
}

function debru_h(arg, p) {
	let args = p.args.reverse().concat(arg);
 	return {l : p.args.length, bd : helper(args, p.bd)};
}

function debru(p) {
	let [_, bd] = p;
 	return debru_h([], bd);
}

function printer(p) {
	if (iso(p)) {
  	let rest = p.bd;
    return "(λ ".repeat(p.l) + printer(rest) + ")";
  }
  else if (islist(p)) {
  	k = "";
  	for (let e of p) {
    	k += " " + printer(e);
    }
    return "(" + k.slice(1) + ")";
  }
  else {
   	return p;
  }
}
</script>

# De Bruijn and why we use it

## Assumed knowledge

At least a familiarity with the lambda calculus, including how it is evaluated. Some base knowledge of programming languages is also assumed.

## The problem

Let's look at a little imaginary term, in some lambda-calculus-like language. For future note, we call the lambda a "binder", as it binds a variable. There are other types of binders, e.g. `let`, but we will only consider lambdas for the moment. 

```hs
λ f. (λ y f. (f y)) f
```

We can perform what's called "beta reduction" on this term — essentially, function application, applying `f` to the lambda.

```hs
λ f. (λ y f. (f y)) (f)

substitute [y := f] in λ y f. (f y)

λ f. (λ f. f f)
```

Uh oh. We've accidentally captured a variable! Instead of `f` referring to the outer `f`, now it refers to the inner `f`. This is "the capture problem", and it is quite annoying. Generally to avoid this, we need to rename _everything_[^1] in our "substituted" term to names that are free (do not occur) in the "subsitutee" so nothing is accidentally captured. What if we could introduce a notation that avoids this?

## Presenting: De Bruijn Indexes!

De Bruijn indexes are a naming scheme where:

- We use natural numbers to refer to lambdas, and
- Zero refers to the "most recent" lambda; one refers to the second most recent, etc.

Let's rewrite that term above using this system:

```hs
λ (λ λ (0 1)) 0
```

Here's some ascii art showing what refers to what:

```hs
 λ   (λ   λ   (0   1))   0
 |    |    \———/   |     |
 |    |            |     |
 |    \————————————/     |
 \———————————————————————/
```

Now, how does this help us with our substituion problem? Surely if we naively subtitute we will still have binding issues - and indeed we do:

```hs
λ (λ λ (0 1)) 0
->
λ (λ (0 0))
```
No good!

What De Bruijn indexes allow us to do is simply avoid capturing. The rule is simple: Every time we go past a binder when substituting, we increment every free variable[^2] in our substituted term by one, to avoid the new binder:

```hs
 λ (λ λ (0 1)) 0
 ->
 λ (λ (0 1))
 ^  ^    ^ incremented by one when we passed "through" lambda b
 |  |
 |  \ lambda b
 |
 \ lambda a
```

Now we're cool! Everything works as expected, and it takes much less work (and is much more predictable!).

At the bottom of this post there's a little widget that can convert terms to de Bruijn for you, if you want to play around!

## Presenting: De Bruijn levels!

De Bruijn levels work similar to De Bruijn indexes, in that we use numbers to refer to binders. However, in De Bruijn levels, the lowest number refers to the *least* recently bound item.

Recall that:
```hs
Named:   λ f. (λ y f. (f y)) f
Indexes: λ (λ λ (0 1)) 0
```
Now, with levels:
```hs
Levels:  λ (λ λ (2 1)) 0
```
This has the same diagram of what refers to what:

```hs
 λ   (λ   λ   (2   1))   0
 |    |    \———/   |     |
 |    |            |     |
 |    \————————————/     |
 \———————————————————————/
```

(As it should! These two representations represent the same term.)

As you might expect, de Bruijn indexes and levels are each beneficial in their own situations.

Generally, De Bruijn indexes are "more useful" than De Bruijn levels, as they're "more local". In order to work with levels, you need to know "how deep" you are in a term at all times.

De Bruijn indexes give us the advantage that we can freely create new binders without the need for any information about where in a term we are, whereas de Bruijn levels give us the advantage when moving a term under a binder, free variables do not need to be modified.

```hs
 λ (λ λ (2 1)) 0
               ^ this zero...
 ->
 λ (λ (1 0))
       ^ ^ is still zero!
       |
       \ we had to modify this one though
```
## Other advantages
Something that can come up quite a lot in various contexts is comparing whether two terms are equal or not. There are many complicated ways to do so, but de Bruijn gives us an advantage in a critical one, called "alpha-equivalence". Consider the following two terms:

```hs
λf. λx. f x
λg. λy. g y
```
These terms should clearly be equal, right? They do the exact same thing. In this case, we consider them "alpha-equivalent", meaning they are equal up to the names of variables. Alpha renaming is the process of renaming one term to match the names of another, so that they are "clearly" equal.

Let us consider the de Bruijn index representation of both of these terms:
```hs
λf. λx. f x => λ λ 1 0
λg. λy. g y => λ λ 1 0
```
Isn't that nice? They've gone from being alpha-equivalent, but not quite equal, to being equal. de Bruijn gives us the ability to compare terms for equality without having to consider alpha-equivalence at all.

## Wrapup

De Bruijn indexes and levels can also be summed up via the following:

```hs
λa. λb. λc. c

indexes:
λ λ λ 0
^ ^ ^
2 1 0

levels:
λ λ λ 2
^ ^ ^
0 1 2
```
if you’re “here”
```
  v
λ λ λ
```
- Using indexes, adding any binders further left doesn’t affect the current binder's variables, or any further right.
- Using levels, adding any binders further right doesn’t affect the current binder's variables, or any further left.

## De Bruijn index widget

Try it out! Some example terms to try:
```
\f x. f x
\x x. x
\x. (x (\y. y))
```
(sorry, it does over-parenthesize a bit :P)

<input name="searchTxt" type="text" maxlength="512" id="searchTxt" class="searchField"/>
<pre data-lang="hs" style="background-color:#383838;color:#e6e1dc;" class="language-hs "><code class="language-hs" data-lang="hs" id="goober">

</code></pre>
<script>
document.getElementById('searchTxt').style.height="100px";
document.getElementById('searchTxt').style.fontSize="14pt";

//creates a listener for when you press a key
window.onkeyup = keyup;
 var inputTextValue;

function keyup(e) {
  //setting your input text to the global Javascript Variable for every key press
  inputTextValue = e.target.value;

  //listens for you to press the ENTER key, at which point your web address will change to the one you have input in the search box
  let p = parse(inputTextValue);
  let k = debru(p);
  let s = printer(k);
  console.log(s);
  const thing = document.getElementById('goober');
  thing.innerHTML = "<span>" + s + "</span>"
}
</script>

## Alternatives
It is worth noting that there are several other methods for gaining the same, or similar, advantages as de Bruijn gives. This post is not intended to explain them, but I will list several here so that the curious reader may read further (tip: when searching, append "lambda calculus" to find the right results quicker):

- HOAS, or "Higher Order Abstract Syntax"
- PHOAS, or "Parametric HOAS"
- Locally nameless
- Nominal signatures
- Well-scoped de Bruijn indices
- Well-scoped names
- <a href="http://doi.acm.org/10.1145/2034773.2034817">“Nameless,
Painless”</a>
- Abstract scope graphs
- Abstract Binding Trees
- Co-de Bruijn indices

As you can see, there are many approaches! Jesper Cockx has an excellent summary of almost all of these, which can be found <a href="https://jesper.sikanda.be/posts/1001-syntax-representations.html">here.</a> Notably, many are intended for formalization efforts rather than for computational usage.

## Footnotes

[^1]: Technically, this is not actually needed. It is sufficient to keep track of everything and rename only names that would overlap. See <a href="https://www.microsoft.com/en-us/research/wp-content/uploads/2002/07/inline.pdf">here</a> for more.

[^2]: A free variable is one that is not bound by the expression we are currently considering. For example, in `λ x. f x`, `f` is free, but `x` is not.
