---
title:  Parsing awk is tricky
layout: page
nav_order: 3
date:   2024-01-24 05:00:00 -0600
---

## Parsing awk is tricky

[//]: # (Markdown comment? First, a bit of a rant about awk syntax.)

What is the grammar and meaning of awk code?
Maybe a yacc expert could know.

If you only know what the _Awk book_ (by A, K, & W, 1st or 2nd Ed.) says, you'd be missing some details.
For example, the Awk book and many other documents say a `for` statement is:
```awk
for (expression₁; expression₂; expression₃)
    statement
```
(or any of the three expressions can be empty).

But try this: `awk 'BEGIN{for (; x++ < 3; print x);}'` (using gawk, nawk, nnawk, or goawk; mawk and bbawk (busybox awk) fail.)

This works because `expression₁` and `expression₃` are actually `simple_statement` which can be an expression, `delete` statement, or `print` statement.
(This is in both the yacc grammar for nawk/nnawk and in the POSIX reference grammar.)
Why?
Maybe no one knows.
Maybe it somehow made the syntax better for yacc.
I asked gawk maintainer Arnold Robbins about this, and he suggested

> I can imagine something like
>
>        for (printf("Enter data: ");
>                getline data > 0;
>                printf("Enter data: ")) {
>                ## stuff with data
>        }

But even if it could be useful, very few users would ever know about it because it's undocumented outside the formal grammar.

### A few interesting cases

#### Case 1
Consider this: `awk 'BEGIN{if (1 < 2 && x = 3) print x}'`.
According to the [POSIX grammar](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/awk.html#tag_20_06_13_16) this should be a syntax error, but it prints 3.

The spec says
> This grammar has several ambiguities that shall be resolved as follows:
> Operator precedence and associativity shall be as described in [Expressions in Decreasing Precedence in awk](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/awk.html#tab41).

Assignment `=` has lower precedence than `&&`, which has lower precedence than `<`.
The statement should parse as: `if (((1 < 2) && x) = 3)`, and the expression to the left of `=` must be an _lvalue_, which it clearly is not.
(The additional parentheses are notional; no parentheses are allowed around an lvalue.)
The rules are similar in C, and a statement like this is a syntax error in C.

#### Case 2

Another odd one, adapted from gawk test `getline.awk`: running this in bash (as e.g. `./getline_test1.sh gawk`):
```bash
#!/usr/bin/bash
awk=$1
echo -e 'A\nB\nC'|$awk 'BEGIN {
	x = y = "!"
	a = (getline x y); print a, x
	a = (getline x + 1); print a, x
	a = (getline x - 2); print a, x
}'
```
with any of gawk, nawk, nnawk, goawk, bbawk gives the same result:
```
1! A
2 B
-1 C
```

But running this (as `getline_test2.awk`):
```awk
BEGIN {
	x = y = "!"
	cmd = "echo A"; a = (cmd | getline x y); close(cmd); print a, x
	cmd = "echo B"; a = (cmd | getline x + 1); close(cmd); print a, x
	cmd = "echo C"; a = (cmd | getline x - 2); close(cmd); print a, x
}
```
gives the same result as above in gawk, mawk, and bbawk, but goawk gets a parsing error on the "echo A" line, and nawk/nnawk gives:
```
1! A
11 B
1-2 C
```

This seems really odd to me.
The original awk (nawk/nnawk -- Kernighan's OneTrueAwk) apparently parses `(cmd | getline x y)` as `((cmd | getline x) y)`, but parses `(cmd | getline x + 1)` as `((cmd | getline x) (+ 1))` and `(cmd | getline x - 2)` as `((cmd | getline x) (- 2))`.
(That is, the `y`, the `+ 1`, and the `- 2` are treated as strings concatenated with the result of `(cmd | getline x)`.)
This is despite the fact that the right side of a concatenation excludes an expression beginning with `+` or `-`, according to POSIX, as it must because it otherwise makes `a + b` ambiguous as to whether it's `(a + b)` or `(a) (+b)`.

#### Case 3
`BEGIN {$0 = "2 3 4"; $$0++; print}` prints `3`, except bbawk says "Unexpected token", and my awk as of this writing prints `2 4 4`.

Which is "correct"? Apparently nawk, gawk, mawk, and goawk work as follows: in `$$0++`, the `$0` is `"2 3 4"` but the `++` forces it numeric, as 2, so the post-increment applies to `$0` which sets the entire record to 3. The subsequent outer `$` selects field 3 of `$0`, which does not exist, so the value of `$$0++` is the empty string.

My awk tries to obey the precedence rules in POSIX, which have `$` at higher precedence than `++`, so evaluates `$$0` first as `$2`, selecting the second field, and then the `++` increments it to 4.

But wait, there's more!
Try `BEGIN {a = 2; b[1] = 2; $0 = "33 44"; print $a; $a++; print; print $b[1]; $b[1]++; print b[1]; print}`
In gawk, mawk, goawk, bbawk, and my awk, it prints
```
44
33 45
45
2
33 46
```
But in nawk/nnawk, it prints
```
44
33 45
45
3
33 45
```
What's happening in nawk/nnawk is: `$a++` increments the field `$a` (i.e. `$2`), but `$b[1]++` increments the array value `b[1]` and does not change the field. All the other awks increment field 2 in both cases, as one might expect. Once again, the original One True Awk's behavior seems inconsistent and not easily discernible from the yacc grammar, nor from the documentation.


#### Case 4
`BEGIN {$0="3 4 5 6 7 8 9"; a=3; print $$a++++; print}` prints 
```
7
3 4 6 6 8 8 9
```
except bbawk and my awk currently indicate a syntax error.

Similar to case 3 above; `$a` selects field 3, which is 5, increments it, but then `$$a++` is a reference to field 5, which is 7, and that is printed as the value of `$$a++++`, but then that field is incremented, giving `$0` the value "3 4 6 6 8 8 9".

My awk (and apparently bbawk?) parse `$$a++++` as if it were `(($($a))++)++` and reject the outer ++ because `(($($a))++)` is a value, not an lvalue.
The parentheses here are again notional; an lvalue cannot be in parentheses.


#### Case 5
`BEGIN {a[y] = 1; x = y in a + 2; print x}` prints    
an empty line in bbawk    
`3` in wak    
`12` in nnawk/nawk (!)    
a syntax error message at '+' in gawk, mawk, goawk

Apparently nnawk/nawk treats the `+ 2` as a string to be concatenated with the `1` from evaluating `y in a` as true.


What causes these peculiarities?

Aho, Weinberger, and Kernighan have [written](https://awk.dev/awk.spe.pdf) that
> The development of awk was significantly shortened by using UNIX tools. The grammar is specified with yacc; the lexical analysis is done by lex. Using these tools made it easy to vary the syntax of the language during development. 

In [_The Unix Programming Environment_](https://scis.uohyd.ac.in/~apcs/itw/UNIXProgrammingEnvironment.pdf), Kernighan and Rob Pike wrote (footnote, p. 254):
> The yacc message "reduce/reduce conflict" indicates a serious problem, more often the symptom of an outright error in the grammar than an intentional ambiguity.

Compiling One True Awk (nawk/nnawk) gets 85 reduce/reduce conflicts in addition to 44 shift/reduce conflicts.

The original yacc grammar for awk has many ambiguities.
I don't know much about yacc yet, but I understand it's basically an LALR(1) parser generator, but with additional features to resolve what would otherwise be ambiguities.
I suspect that these features make it hard to understand exactly what the parser does if you are not a yacc expert.
The actual awk language is determined by what the parser accepts and what it does with that.
My guess is that there may not be any true LALR(1) grammar for awk, maybe no LR(1) grammar, and probably no LL(1) grammar either.
This makes writing a recursive-descent parser by hand somewhat difficult.

The POSIX grammar is in a yacc-like format and was obviously written with some reference to the original yacc grammar for awk.
I can say that with some confidence because it includes the `for (simple_statement ...)` business discussed above, which one would never know without studying the original yacc grammar.
But as shown above, existing implementations do not always follow POSIX grammar, and when they differ they usually follow the original awk behavior.

I asked Arnold Robbins (gawk maintainer and contributor to One True Awk) about case 1.
He replied "The answer, as unsatisfying as it may be, is that Awk has always worked this way."
Also, regarding case 1 above, Ben Hoyt had the same issue in goawk.
He resolved it with the [commit](https://github.com/benhoyt/goawk/commit/799b2b092c7862506ee3dc0395f35665e072f6d1) that included this comment:
> The other awks support this by using a yacc grammar which supports backtracking, and as Vitus13 said on reddit: "If there are two syntactically valid parsings and one is a semantic error, the error handling may resolve the ambiguity towards the valid parsing. In this case, you can only assign to L values, so trying to assign to (1&&x) doesn't make any sense."

I believe this is a misconception; yacc does not support backtracking (though there is a different program, btyacc, that does).
I think Ben and Vitus13 are wrong here; yacc does not see multiple syntactically valid parsings, it sees only the parsing that it performs deterministically, and many of the problems understanding what nawk/nnawk does are due to the way yacc resolves conflicts.

