---
layout: post
title:  "Abusing type checking for fun and profit"
date:   2017-12-11 17:00:00 +0100
categories: HelenOS C errno
---

This is a post about error handling in the C programming language in general,
and in HelenOS in particular.

First, a bit of context. C traditionally doesn't have very strong type system,
especially when it comes to integer values. There is basically no support for
defining new integer types -- there are `char`, `signed char`, `unsigned char`
(yes, those are three distinct types; while `char` is semantically identical
to one of the other two, it's still separate and not just an alias), `short`,
`unsigned short`, `int`, `unsigned int`, `long`, `unsigned long`, `long long`,
`unsigned long long`, and `_Bool`. That's the exhaustive listing for standard
types. On several platforms, compilers also define the non-standard `__int128`,
but it's not universally supported.

Other non-stardard integer types are
allowed, but this is not done in practice. All the other types you get from
various header files -- e.g. `int32_t`, `size_t`, `wchar_t`, etc. -- are all
just aliases to one of the above, a measly `#define int32_t int` (although
these days `typedef` is more commonly used, semantically, it makes no difference
-- `typedef` creates an alias, not a new type). Enumerated types defined via
`enum` are no different. Although recent compilers come with scores of
diagnostics for `enum` types and their constants, it's a far cry from strong
type checking. C++ gained its `enum class` some time ago, but C, sadly, doesn't
have anything like that.

It comes as no surprise, then, that most C code doesn't really distinguish
between various numeric types, whether they are enumerations, bitflags, or
file descriptors. I jokingly call it the "all-int" situation. See a parameter
or a return value typed `int`? Well great, you learned next to nothing. It
could be anything. The designation has no semantic value.

Naturally, this extends to error-handling. In HelenOS code base, the go-to
error handling mechanism has been to return negative error codes on failure
and positive valid returns on success. Correspondingly, its `<errno.h>`
header defined negative constants, contrary to the C language standard.
This led to problems. Interfacing HelenOS libraries with code written for
standard environment (typically POSIX) has been more painful than necessary,
and where using standardized error codes just doesn't cut it, domain-specific
error codes have been used with mixed results. On several occasions, different
kinds of error returns have been mixed improperly, resulting in hidden bugs
that only manifest in the rare exceptional conditions.

# Towards the solution

The issue with negative error codes is probably the single greatest blocker
for a standards-compliant libc in the heart of HelenOS. However, since the code
depends on them being negative, just changing the constants would break pretty
much everything. Annoyingly, just separating error returns from actual results
is not by itself sufficient, because some code would still (improperly) check
for negativity, and it wouldn't help with existing error handling bugs, or
with bugs inadvertently introduced during the transition.

My first attempt was to simply rename the constants and keep them negative,
reintroducing standard error codes on a case-by-case basis. This turned out
to be a spectacularly useless idea. It would create many problems and probably
cause more pain than it solved. I still thought the solution would be in
splitting the errors into independent, API-specific groups, but had little
idea how to turn that into practice. At the very least, I decided it would
help to introduce the C11 `errno_t` type, and see where it goes.

Then, a week ago, Jiří Svoboda started his own efforts of separating error
returns from valid results, which at the time duplicated/conflicted-with my own
efforts. However, this pointed me back to the idea of adding output parametes
instead of working with negative returns by another name, something that I
originally dismissed as distruptive.
After a short e-mail conversation, I asked Jiří to give me until the
end of the week to work on this my way, to which he agreed.

Solving all the issues by the end of the week, in the entire code base?
Insane! Well, not quite. And I would have managed if I didn't make some silly
mistakes in the process, but I digress. I was already considering how to utilize
compiler diagnostics to detect problems, so when Jiří started separating the
error values, I got an idea how to exploit it fully.

# s/int/errno_t

The idea is simple. If we mark every error value by a specific type (such as
`errno_t`, because why not?), then we can make the compiler fail-out on every
instance of errors getting mixed with non-errors. "But wait," you say, "didn't
you just explain that C can't do that?". Well, sort of. You see, the typing
doesn't necessarily have to make sense or work at runtime, it just needs to
typecheck. If the typechecker guarantees that no mixing is happening, you
can change the type and constants after the fact and the guarantee still
applies (at least until you make new bugs). And C actually does have decent
diagnostics for various types, even if not all of them in any single type.

So I started by defining `errno_t` to be a unique pointer type, and all `Exxxx`
constants to be pointers of that type. This gives us some rather strong
guarantees: no assigments from or to other types without explicit casts,
no comparison to integers (except for equality with zero, which doesn't hurt
us), no printf as an integer (not strictly a problem, but it's always nice to
see a string representation instead of a random number).

That leaves the issue of actually changing the type of thousands of instances
of function parameter/return values and variables. As Jiří pointed out, in
HelenOS almost every function that returns `int` returns an error code. Which is
exactly what makes it easy. We can just mechanically rename all `int` return
types, along with a select few variable names (`rc`, `ret`, `retval`, a few
others that came up). There are far fewer exceptions than there are errors,
so doing the automatic replace and then fixing the problems is much easier than
going the other direction (remember, at this point `errno_t` is type-checked,
so there's no way to miss an `errno_t` variable that has a non-error number
assigned). And the great thing about it is that applying a reverse rename from
`errno_t` to `int` doesn't change semantics and gives a nice, manageable diff
of actual changes. Naturally, there are a lot of instances where new variables
had to be introduces to separate errno errors from other numbers, but faced with
the certainties we get in exchange, it's a rather small price to pay.

It was still a lot more demanding that I anticipated, mostly because I made
some mistakes early on that forced me to redo a lot of the work (automatic
renames can be tricky to use right), but I still consider it well worth the
effort. As of now, I finished userspace, with major changes committed and
remaining minor changes (and final gargantuan reverse-automatic-rename patch)
pending review. Kernel is still in the works (the uspace part exhausted me),
but should be ready in a few days.






