+++
date = "2017-05-26T02:29:08+02:00"
title = "Understanding Nix's lib.fix"

+++

Understanding Nix's [*lib.fix*](https://github.com/NixOS/nixpkgs/blob/dd2b1744ba36ab8d5b352cdac9c98a5e1eb71e8f/lib/trivial.nix#L63) — a.k.a. ["fixed point combinator"](https://en.wikipedia.org/wiki/Fixed-point_combinator) — and then based on it *lib.extend*, is crucial for understanding various NixOS features, including *overlays*.

Conceptually, *lib.fix* is a very similar mechanism to `rec` — it allows defining some attributes of a set "recursively", using values of other attributes from the same set. The main difference is, that while `rec` is "eager", *lib.fix* is its "lazy" sibling. This magically allows using it to build various powerful features, such as *overlays*, *lib.extend*, etc.

For a super simple example of using *lib.fix*, if we first type [its definition](https://github.com/NixOS/nixpkgs/blob/dd2b1744ba36ab8d5b352cdac9c98a5e1eb71e8f/lib/trivial.nix#L63) in *nix-repl* by hand, we can then call it like below:
```nix
nix-repl> mySetBuilder = self: { foo = "foo"; bar = "bar"; foobar = self.foo + self.bar; }
nix-repl> fix mySetBuilder
{ bar = "bar"; foo = "foo"; foobar = "foobar"; }
```

For comparison, **`rec`** would be used like below:
```nix
nix-repl> rec { foo = "foo"; bar = "bar"; foobar = foo + bar; }
```

As we can see here, the main differences in usage are:
  - `rec` takes a *set*; *lib.fix* takes a (single parameter) *function returning a set*;
  - `rec` takes dependencies (foo, bar) from "local scope"; *lib.fix* requires that you formally take them from the parameter (but then somehow magically they resolve to the local values).

### How *lib.fix* works?

The [definition of *lib.fix*](https://github.com/NixOS/nixpkgs/blob/dd2b1744ba36ab8d5b352cdac9c98a5e1eb71e8f/lib/trivial.nix#L63) is a single, deceiptively simple looking line:
```nix
fix = f: let x = f x; in x;
```
In simple human-readable words, this would decode roughly as: "*fix* is a function, which takes a function *f*, and calls it (the *f*) on its own result."

Uh, oh — OK; but then, um... to start with, where to take the f's *initial argument* from? or does it, I dunno, magically appear out of thin air, or what? Also, while we're at it — this kinda looks like calling the function f infinitely; isn't this a classic example of infinite recursion? why does nix-repl actually print *some result*, not to mention that *so quickly*?

To answer above questions, one must notice some characteristics of the Nix language, which are not usually found in imperative languages (but much more often encountered in functional/declarative ones). Specifically:
  1. **Nix is a *"lazy"* language.** This means, that expressions are not evaluated immediately; quite the reverse — they're kept in their "symbolic" form as long as possible, and evaluated only as late as possible, and *only if really necessary*. If not necessary, they're simply happily ignored and forgotten!
  2. Lexically, all symbols defined inside the `let` keyword are "mutually recursive", also to themselves — **order of definitions inside single `let` is totally irrelevant.**

Armed with this knowledge, we can now try putting ourselves in "Nix's mind", to try and analyze how Nix would understand a sample *lib.fix* call &mdash; for example, the one we tried earlier:
```
nix-repl> f = self: { foo = "foo"; bar = "bar"; foobar = self.foo + self.bar; }
nix-repl> fix f
{ bar = "bar"; foo = "foo"; foobar = "foobar"; }
```

Let's see how Nix would expand `fix f` in the above expression, but do it step by step &mdash; and in a *lazy* way:
```nix
1.  fix f             # hmh? oook, you want me to do some work? ehhhh. Ooook, ok; now, now, don't push me,
                      # ehhh; yeeeeaaaa, I'm totally already starting to get to the work. Ok? ok??? gosh.
                      # Ok, ok, cool out, man. I dig it. You want me to analyze it. I see it, it's some fix
                      # and some f, yes? Hmh. So it's some function call, that's what my parser tells me.
                      # fix is the function. Heeey, man! so how do you want me to work on it, if I don't know
                      # what this "fix" of yours is?? uh, oh, right; you've given me it earlier, ok, ok.
                      # Jeez.
2. (f: let x = f x; in x) f       # Ha! I expanded your "fix", see?
                                  # Now, what does all this mean? This function totally does some stuff;
                                  # but now let's say I want to cheat and not do any work; can I get away
                                  # with it? This whole stuff is some function call; but the function inside
                                  # generally really really wants to return to me some "x". That's one thing
                                  # I for sure cannot avoid, so let's start with it.
3.                     x      # Hah, you like your "x" now? Hmmmh; ok, you want me to explain this "x".
                              # Yeah, it's not *some* x, it's *the* x, of which we have a very precise
                              # definition. Let's get to it, then.
4.     let x = ...  in x         # ...and moooreee....
5.     let x = f x; in x      # Hah, got you! Now I have your x, see? it's just "f x"! Eat it!!!
6.                     f x       # Happy now???
                                 # No? :( ok, ok. But we've seen such a thing already, didn't we?
                                 # It's a Function Call, again. So let's expand the "f" function.
                                 # You've given me its definition, too. Kinda heavy, so it will take more
                                 # space, prepare for it.
7.                     (self: {foo="foo"; bar="bar"; foobar= self.foo+self.bar; }) x
                                 # Much more heavy. But, now, we call it with "x", so "self" becomes "x",
                                 # and then it's not anymore a function, just a set.
8.                     {foo="foo"; bar="bar"; foobar= x.foo+x.bar; }
                                 # Done.
                                 # What, no? Ohhhh, x, the x, this little fella again. Ooook.
                                 # We've had it once already, didn't we. Let's go get it back again.
9.     let x = ...  in {foo="foo"; bar="bar"; foobar= x.foo+x.bar; }
10.    let x = f x; in {foo="foo"; bar="bar"; foobar= x.foo+x.bar; }
                                 # Look, "x = f x". So I can replace all "x" with "f x"!
11.                    {foo="foo"; bar="bar"; foobar= (f x).foo + (f x).bar; }
                                 # But, but, but! I think I've seen an "f x" already. Ha, yes, I did!
                                 # See above, in step 6.! An "f x"! And then we expanded it a bit, and
                                 # in step 8., we got some stuff. Let's take it here. But only what I need!
                                 # Do it laaaazy. When I need .foo, I need *only* foo!
12.                    {foo="foo"; bar="bar"; foobar= {foo="foo"; ...}.foo + {...; bar="bar"; ...}.bar; }
13.                    {foo="foo"; bar="bar"; foobar=      "foo"           +           "bar"          ; }
                                 # Soooo.....:
14.  {foo="foo"; bar="bar"; foobar= "foo" + "bar"; }     # now the +...
15.  {foo="foo"; bar="bar"; foobar="foobar"; }

# Ta-DAAAAAAAA!!!!!!!!
```

See now? What do you think?
