# C23 generic functions

This is a (slightly edited) copy of two articles I posted to the
comp.std.c Usenet newsgroup on Thu 2023-06-01.
```
Message-ID: <87fs7afpcw.fsf@nosuchdomain.example.com>
Message-ID: <87bkhyfnm7.fsf@nosuchdomain.example.com>
```

Quick summary:

- The new generic string functions in C23 (`strchr`, `strpbrk`,
  `strrchr`, `strstr`) could break some code is valid in C17 and
  earlier, specifically code that passes a `void*` or `const void*`
  argument.  Such code is probably contrived and unlikely to appear
  in the real world..  A minor update to C23 could address this.
- The `memchr()` generic function has a similar issue that's more
  likely to show up in real-world code, when a pointer to a type
  other than `void` is passed.  I do not offer a solution to this,
  but the issue should be noted.
- The underlying issue is that the implicit type conversions that are
  applied to function arguments are not applied to the operands of
  a generic selection, but these generic functions are intended to
  work like functions and to replace ordinary functions with the same
  names from earlier editions of the C standard.

-- [Keith.S.Thompson@gmail.com](mailto:Keith.S.Thompson@gmail.com)

----

**Subject: Can the new generic string functions accept `void*` arguments?**

The latest draft of the upcoming C23 standard is:  
<https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3096.pdf>  
It introduces several type-generic functions in `<string.h>`, replacing
normal functions of the same names: `memchr`, `strchr`, `strpbrk`, `strrchr`,
`strstr`.

I'll use `strchr()` as an example; the same applies to the other `str*()`
generic functions (but not to `memchr()`).

*But see below for a discussion of `memchr()`.*

The problem this solves is that calling `strchr()` with a `const char*`
argument yields a (non-`const`) `char*` result that points into the array.
For example:
```
#include <stdio.h>
#include <string.h>
int main(void) {
    const char s[] = "hello";
    char *p = strchr(s, 'h');
    *p = 'J';           // Undefined behavior
    printf("%s\n", s);  // Likely to print "Jello"
}
```

This makes it possible to obtain a non-const pointer to a const object
without a pointer cast.

The C23 `strchr()` generic function returns a `char*` if the first argument
is a `char*`, or a `const char*` if the first argument is a `const char*`.

> The stateless search functions in this section (`memchr`, `strchr`,
> `strpbrk`, `strrchr`, `strstr`) are *generic functions*. These functions
> are generic in the qualification of the array to be searched and
> will return a result pointer to an element with the same
> qualification as the passed array. If the array to be searched is
> `const`-qualified, the result pointer will be to a `const`-qualified
> element. If the array to be searched is not `const`-qualified, the
> result pointer will be to an unqualified element.

So far so good, and I definitely approve of this change.  It does break
code that calls `strchr()` with a `const char*` argument and assigns the
result to a (non-`const`) `char*` object.  That's IMHO a minor issue, and
arguably breaking such code is part of the point of the change.  (Making
string literals `const` would be similar, but I suppose that's still a
bridge too far.)

But I've thought of a way in which this could break some existing valid
code, namely code that passes a `void*` or `const void*` argument to
`strchr()`.

Currently (in C17 and earlier), since `void*` can be implicitly
converted to `char*` and vice versa, such a call is valid.  (I can't
think of a *good* reason to write such a call, but my imagination is
not unlimited.)

**Question: Is this a valid call in C23?  (It's valid in C17.)**
```
char hello[] = "hello";
void *p = strchr((void*)hello, 'h');
```

An implementation of the generic `strchr()` will presumably use a generic
selection in a macro definition.  If the generic selection covers only
types `char*` and `const char*`, the call will violate a constraint.  If it
also covers `void*` and `const void*`, the call will be valid.

The current wording in N3096 suggests that only `char*` and `const char*`
are covered, implying that a call with a `void*` or `const void*` argument
is a constraint violation.

I suggest that the C23 standard should specify whether `void*`
arguments are valid or not.  I have a slight preference for making
them valid.  If so, the simplest approach would be for `strchr()`
to return a `char*` given a `char*` or `void*` argument, or a `const
char*` given a `const char*` or `const void*` argument.

----

*Here's the second article I posted to comp.std.c, after realizing
that there's also an issue with `memchr()`.*

----

Just after I posted the above, I thought of a potential issue with
`memchr()` that just might affect real code.

In C17 and earlier, `memchr()` has this declaration:
```
void *memchr(const void *s, int c, size_t n);
```

Given the implicit conversions between `void*` and other object pointer
types, the first argument can be a pointer to any `const` object type.
This is something that might plausibly be used in practice, unlike
(I think) passing a `void` pointer to the `str*()` functions.

It's probably impractical to fix this, since it would require
the generic selection to cover all possible object pointer types.
Any code that depends on the current behavior would have to add
`(void*)` or `(const void*)` casts to ensure that the type actually
matches.

For example, this (contrived) program is valid in C17 and earlier:
```
#include <stdio.h>
#include <string.h>
int main(void) {
    const unsigned u = 0x12345678;
    printf("u = 0x%x", u);
    unsigned char *p = memchr(&u, 0x34, sizeof u);
    if (p != NULL) printf(", p points to 0x%x", *p);
    putchar('\n');
}
```

The output is:
```
u = 0x12345678, p points to 0x34
```

(Conceivably `p` might be a null pointer if `unsigned int` has padding
bits that cause `0x34` not to be stored in a single byte.  That's not
particularly relevant to the point.)

A call to `memchr()` with a `char*` argument is, I suspect, more
likely to appear in real code.

The underlying issue is that the implicit conversions that happen with
function arguments do not happen with operands of a generic selection.
(The generic functions in `<tgmath.h>` are defined in a way that this
isn't an issue, as far as I can tell, though I haven't looked closely
at the C23 updates.)
