+++
title = "Opinion piece: On Zig (and the design choices within)"
date = 2025-10-14
+++

(From someone who has spent far too much time thinking about the designs of programming languages)

This post is split up into a few sections. I would also like to preface this post with:
1. This is a little bit of a rant. I believe it presents some very reasonable points about Zig as a language, but it's not going to be
perfect or completely objective. It is an opinion piece, and you're free to ignore it if you want.
2. Existing memory safe languages are not perfect. Rust will be used as an example quite a bit, as from what is currently available, it embodies closer to what I think is reasonable in a programming language. However, there is still much room for improvement in this space.
3. All of the following is ultimately my opinion. If you disagree, you are welcome to find me somewhere on the internet and ask me more.
4. I bear no grudge or ill will against anyone who contributes to, works on, or is otherwise involved in the Zig community. The following are things I have noted about the language, not the people behind it. If you enjoy Zig, go and write it, and make cool stuff - I am not the programming language police.
5. Exception to the above: I have one gripe about the Zig community. This gripe "targets" no one person in particular; rather, larger trends.

## Ideological differences

### Memory

First, and very much foremost, Zig is not memory safe. This is, in my opinion, the most 
egregious thing in this post, by a very large margin. Moreso, Zig does not make any attempt to 
be memory safe - 
it can catch some things at runtime, with specific allocators, but so can C these days. Indeed, there
are some cases, like use-after-realloc, that `asan` can catch and Zig cannot.

A language in the modern day that does not make an attempt at memory safety is, in my opinion, not reasonable. It has been shown that in some areas, up to 70% of security bugs are due to memory safety issues (<a href="https://www.chromium.org/Home/chromium-security/memory-safety/">Source</a>).

I subscribe to the idea that the user must be constrained. It is perhaps harsh to say, but for large and complex programs, I believe that there are very few programmers who will write memory-correct code nine times out of ten. When writing code with others, that goes down. I personally do not believe I fit into that category.

The fact that Zig allows the user to write faulty software is supported by various somewhat informal, but still useful, statistics. Notably, the following statistic disregard duplicates, and unreported errors. However, general trends are still of note. <details><summary>Here are some:</summary>

1. The Rust compiler has had a lifetime 59,780 issues reported. Of these, 4,158 contain one of "crash", "segmentation fault", or "segfault".
2. The Zig compiler has had a lifetime 13,269 issues reported. Of these, 2,260 contain one of "crash", "segmentation fault", or "segfault".

This means that, roughly:

1. 7% of lifetime issues reported in the Rust compiler are crashes.
2. 17% of lifetime issues reported in the Zig compiler are crashes.

Not ideal.

1. The Node/v8 JS runtime, written partially itself in JS and partially in C++, has had a lifetime 19,631 issues reported, 1,710 of which contain one of the above keywords.
2. The Deno JS runtime, written primarily in Rust, has had a lifetime 13,422 issues reported, 552 of which contain one of the above keywords.
3. The Bun JS runtime, written in Zig, has had a lifetime 13,828 issues reported, 3676 of which contain one of the above keywords.

Again, this roughly means:
1. 8.7% of lifetime issues reported in Node are crashes.
2. 4.1% of lifetime issues reported in Deno are crashes.
3. 26.5% (!!!) of lifetime issues reported in Bun are crashes.

Also not great. Again, these statistics are slightly off at best simply due to the nature of their collection, but the trends do not lie.

</details>

If you're not reading the above: It can be summarized as "Not ideal."

### Philosophy

At one point, this part of the article contained a runthrough of the `zig zen`, and my opinions on each bullet point. I have decided that that is not a constructive discussion of Zig. It suffices for me to say that I do not believe Zig particularly embodies its own zen.

## Practical 

### Comptime

Zig does generics in an odd way. I believe this is the best way of putting it. 
This post is not meant to be, nor will it contain, a proper explanation of Zig's `comptime`
capabilities, so I refer the reader to the wider internet there. However, doing generics
with "normal" code means that there are multiple ways to write the same generic function. 
There is no standardization between different libraries, different styles of writing code, and 
different users of Zig. Every person can do generics in their own special way. 

This obviously has slightly dire effects on readability. In practice, most Zig users are reasonable enough
to stick to some "common patterns" of doing generics and similar, but it is widely known that if the user is giving the ability to do something, it will be done. I believe that generics are important enough they should be first class.

Arguments can be made that Zig's comptime is also useful for other things, not just generics. Some examples I have been given are conditional compilation, and variadic functions. Beyond these, I am yet to see a convincing motivating example that _requires_ the machinery that Zig provides. Every such example can, in my experience, be solved with less powerful (and hence, for the most part, less confusing) machinery.

As a result, I am inclined to believe that Zig's comptime is a very large and all-encompassing feature that ultimately brings very little to the table that smaller features cannot.

I am personally a proponent of a good macro system, but I will readily admit people can also go overboard with one of those. 

### Casting

Zig's casting is a bit cumbersome. To cast a float to a specific int width, for example, must be done with `@as(i32, @intFromFloat(flt))`. Bit of a mouthful. Inference can help here (For example if a variable is already known to be a `i32`, the outer cast is not needed), but I would think that with Zig's comptime abilities, this could be made a bit nicer. Luckily, it is common Zig practice to annotate everything if possible, so this does come up slightly less in practice. It is still a bit bulky however.

Float to int casting additionally can invoke undefined behavior if the float is outside the integer's range. I personally prefer truncating semantics, with perhaps a specialized method (`intFromFloatUnsafe`?) for UB semantics. That's more of a personal preference though.

## Semantics

### Result location semantics (RLS)

<a href="https://ziglang.org/documentation/0.15.1/#Result-Location-Semantics">See here</a>. 

Result location semantics are in theory a quite nice idea. Knowing predictably where things are going to be placed in memory is, of course, good for any systems-adjacent language! In practice, there are several choices within RLS that I find counterintuitive. Take the following example, where we attempt to swap two struct members in place:

```rust

const std = @import("std");
const What = struct {
    a: i32,
    b: i32,
};

pub fn main() void {
    var x: What = .{ .a = 1, .b = 2 };
    x = What { .a = x.b, .b = x.a };
    std.log.info("x: {}", .{x});

    var y: What = .{ .a = 1, .b = 2 };
    y = .{ .a = y.b, .b = y.a };
    std.log.info("y: {}", .{y});
}
```
This outputs the following:
```rust
info: x: .{ .a = 2, .b = 1 }
info: y: .{ .a = 2, .b = 2 }
```

I will note that the equivalent C can only be written in the former style, and it prints the former.

The only difference between the latter and the former is whether the type name is present. This is intended behavior! Indeed, this very example is given in the Zig reference, a fact that I find odd. Why give a warning against something when you could just fix it? Zig does not have move or copy constructors; I do not think there is very much reason for the latter to ever behave the way it does. One extra register is all you ever need, even for an arbitrary parallel move! (<a href="https://compiler.club/parallel-moves/">Source</a>.)


### Pointer reference optimization

In its previous form, PRO has been removed from Zig. There was a long period where this would print `5`:

```rust
const std = @import("std");

const AAAA = struct {
    foo: [100]u32,
};

fn kindabad(a: AAAA, b: *AAAA) void {
  b.*.foo[0] = 5;

  std.debug.print("unideal: {}", .{ a.foo[0] });
}

pub fn main() !void {
    var f: AAAA = undefined;

    f.foo[0] = 0;

    kindabad(f, &f);
}
```

As Zig would correctly, per PRO at the time, but incorrectly per common sense, pass `a` as a reference. 

This is actually mildly unfortunate. It's a very interesting optimization, and being able to guarantee behavior around optimization of parameter passing would also be quite beneficial. Unfortunately, Zig simply does not have the level of control needed to do it. Aliasing is freely allowed, and in order to make PRO work, it cannot be. Rust can, and in many cases does, do this! It's a shame Zig has to miss out.

There is current work to make PRO work in the case of pure functions (<a href="https://github.com/ziglang/zig/issues/5973#issuecomment-2380332493">Source</a>), so I remain hopeful that it will return in some form eventually.

## Misc. Improvable things

### Speed

The Zig compiler is not particularly fast. The LLVM backend even more so; compared to Clang, (which has the home field advantage, but can still be quite slow), and by my measurements, Clang is consistently between 3-10x faster. A new Zig backend has been written to help fix this; by similar measurements, it is consistently faster than Clang by 1.5-2x, and much faster than the Zig LLVM backend. While these numbers are nothing to scoff at, I believe there is more room for improvement, and I would like to see where it goes. Zig is a simpler language than C when comptime is not involved, and I would be excited to see that reflected in the benchmarks. It is also possible that Zig could use LLVM more efficiently, but I cannot comment on this without digging into the internals of the compiler. 

Of course, all of this is quite reasonable. The Zig compiler has had far, far fewer man-hours put into it, and this of course reflects in aspects of the compiler that simply take a lot of time, such as these. I look forward to seeing what can be done in the future.

### Tooling

#### Build system

The Zig build system is a little confusing. It is very neat to be able to write build system code in the language itself, and this is a feature I believe more languages should have. Unfortunately, it is as of current not well documented enough to justify its own complexity. This of course can be improved, and I hope it is.

#### Language server

Zig currently does not have a "first class" language server. The "Zig Language Server", or ZLS, is unofficial; It does not have real compiler integration, so it is unfortunately limited in some ways. The creator of Zig apparently does not use a language server while programming, so I do slightly understand why it is not a priority, but I personally believe tools like a good language server are quite essential to wider/popular usage. Here are some quotes I have heard from people who have used it far more than I:
- "Deeply horrid"
- "It is one of the worst LSPs I have ever used"
- "Don't get me started"

Apparently it is very limited when `comptime` comes into play, and cannot handle compound types like 2D arrays.

I cannot offer much firsthand experience however.
### Debug mode

There are always more errors to be caught. I particularly recently noticed that Zig cannot currently catch use-after-realloc errors, like the following:

```rust
const std = @import("std");

pub fn main() !void {
    const allocator = std.heap.page_allocator;

    var buffer = try allocator.alloc(u8, 4);

    buffer[0] = 1;
    
    const new_buffer = try allocator.realloc(buffer, 8);
    
    // Should be caught!
    buffer[0] = 99; 

    allocator.free(new_buffer);
}
```

Address sanitizer with Clang can catch this, so theoretically Zig should be able to catch it (perhaps with a similar system) too.

## Misc. Improvable things that won't be improved for reasons beyond me

### Behavior around `undefined`

#### Debug mode

Apparently, it is desired that comparisons with `undefined` panic. This seems reasonable to me. However, the issue for this has been open <a href="https://github.com/ziglang/zig/issues/63">for almost ten years</a>, which makes me worry that it might be a while still until it is finished. 

This panics in `Debug` and `ReleaseSafe` modes, however, so it does at least work in some cases.

```rust
pub fn main() void {
    const x: u32 = undefined;
    if (x == 0) {
        return 1;
    } else {
        return 0;
    }
}
```

#### Safety

According to <a href="https://github.com/ziglang/zig/issues/21915">this issue</a>, it is "not a goal of the Zig compiler" to catch issues like the following:

```rust
pub fn main() void {
    var buf: []u8 = undefined;
    
    // Very not allowed!
    buf[0] = 1;
}
```

This just seems silly to me. Catching something like this, in the simple case, should be just one pass over the AST; why not do it?

### Tabs and syntax

Zig simply doesn't allow tabs in comments and strings? <a href="https://github.com/ziglang/zig-spec/issues/38">This issue</a> explains more, but the justification of "it (a tab) is ambiguous (in) how it should be rendered", I do not quite agree with.

Strings make some sense; you can still write `\t` for a tab, and they do indeed make output ambiguous. However, they're still allowed in indentation! This means that in commenting out code, you can take a program from valid to invalid. `zig fmt` does reify tabs into spaces, but if your editor is misconfigured and you're not running `zig fmt` constantly?

```
tabs.zig:5:36: error: comment contains invalid byte: '\t'
//        const str: *const [1:0]u8 = "w";
  ^~~~~~~~~~~
```

Very odd.

Similarly, the following will not compile:

```c
if (a >=b) {
    ...
}
```
because the whitespace isn't correct. Zig doesn't have warnings, so this is always a compile error.

### Iteration

Zig does not have a way to iterate through a slice or similar in any way except one-at-a-time forwards. Anything else? You're using a `while` loop, and you'll enjoy it. Zig blanket bans proposals to change the language, so despite issues like <a href="https://github.com/ziglang/zig/issues/21814">this one</a>, I doubt this will happen any time soon.

### Warnings

Zig doesn't have 'em. Everything is an error. In practice, this mostly manifests through reasonably harmless things like "unused variable" notices becoming errors, which is reasonably annoying when just trying to spitball.

## My only note on the community

For the most part, the community is about par for the course for programming language communities; that is to say, quite nice. However, I have had some reasonably negative interactions that I feel are somewhat of a general trend. They usually proceed like the following:

1. I ask a question about solving something in Zig. Often, given my background, it is about adapting my more functional-esque thinking to Zig.
2. Someone chimes in with a half-solution that does not actually address my problem.
3. I attempt to communicate this to them, which often involves explaining the features I would use in \<insert other language\>.
4. They insist that these features are not needed, but do not elaborate on how one would solve the problem without them.

This is unpleasant, and while it's not a particularly uncommon thing to see in programming language communities, Zig seems to have a bit of a bad case of it. I suspect it is due to Zig's fairly minimalistic nature; it lacks a lot of features that one would otherwise use to solve problems. Of course, this is the appeal for many, but still. It is made worse when said topic is something one is particularly knowledgeable about, and the people you are conversing with believe they can solve the issue without that knowledge.

I do not believe that a few bad apples necessarily spoil the whole barrel here, but they definitely sour it.

## Summary

I find Zig interesting, with an unfortunately negative connotation. I believe the goal of a C-like memory unsafe language "for the modern day", while interesting at first glance, ignores many of the issues that make C a problem in said modern day. Much of Zig seems to me like "wishful thinking"; if every programmer was 150% smarter and more capable, perhaps it would work. Alas, they are not; myself included.

I believe that modern concerns of memory safety and correctness require modern solutions; not performing patchwork fixes over the core issue.
