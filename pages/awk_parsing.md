---
title:  Parsing awk
layout: page
nav_order: 4
date:   2024-01-24 05:00:00 -0600
---

## Parsing

I wanted to try parsing awk using recursive descent along with Pratt's method of handling expression syntax.
Pure recursive descent without backtracking requires a grammar meeting certain requirements, specifically LL(1).
The reference grammar for POSIX awk does not seem to meet this, and I am doubtful a correct LL(1) awk grammar exists.
There are some difficulties discussed below, so a bit of finagling in the parser is needed.


If you are not familiar with Pratt parsing, there are [several](https://crockford.com/javascript/tdop/tdop.html) [tutorials](https://journal.stuffwithstuff.com/2011/03/19/pratt-parsers-expression-parsing-made-easy/) and [discussions](https://eli.thegreenplace.net/2010/01/02/top-down-operator-precedence-parsing) around on the Web.
Vaughan Pratt introduced the idea in a [1973 paper](https://dl.acm.org/doi/10.1145/512927.512931), then Douglas Crockford sort of rediscovered and popularized it in the first paper linked above.

The basic idea is that instead of having a separate routine to parse each operator or precedence level as in pure recursive descent, you have a main "expression" routine with a loop to run through the tokens of the expression that calls "nud" and "led" routines for each operator, and those routines can recursively call the main expression routine to process subexpressions with the same or higher operator precedences.

The "led" and "nud" (left denotation and null denotation, in Pratt's terms) routines are for operators that take a left operand (e.g. infix and postfix operators) or no left operand (prefix operators), respectively.

I was blocked for a time by the concatenation operator being present by its absence.
That is, in awk, there is no explicit operator for string concatenation; it happens when two expressions appear adjacent with no operator between them.
I figured I had to determine it and insert it by looking for places where a symbol that can end an expression is followed by a symbol that can start an expression.
But the `++` and `--` operators threw me.
They can either end or begin an expression.
So I could not see how to have `++` and `--` as both prefix and postfix operators, that is, handled by both the "nud" and "led" routines. 

Then I saw that if they followed an lvalue they were always taken as postfix.
I modified my "nud" routine, which was handling prefix operators, to also consume `++` / `--` operators if they followed an lvalue.

I also had my main expression routine `exprn()` looking for the start of an expression where it could be the right side of a concatenation.
Note that the right side cannot start with `+` or `-`, or there will be confusion between `a + b` and `a` (concatenated with) `+ b`.
If you look at the [POSIX grammar](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/awk.html#tag_20_06_13_16) for awk, you'll see a long list of nearly identical productions for `unary_expr` and `non_unary_expr`, that are needed solely to exclude the right side of a concatenation beginning with unary `+` or `-`.
There is nothing like that in the yacc grammar for nawk/nnawk/OneTrueAwk, and I'm not clear on how yacc handles this.

But now my "nud" routine was processing not only prefix operators and plain variables, but also subscripted variables (array references), as well as postfix `++` and `--`.
By the time I got it working, the "nud" routine was also handling the `$` field reference operator and function calls, and the parenthesized expression list normally seen with the array-subscript test expression `(expr, expr,...) in arrayname`.
The supposed Pratt parser was looking more similar to a [precedence](https://www.engr.mun.ca/~theo/Misc/exp_parsing.htm#climbing) [climbing](https://eli.thegreenplace.net/2012/08/02/parsing-expressions-by-precedence-climbing) parser, though it still has a Pratt-like main loop routine for expressions and still uses "left binding power" and "right binding power" concepts from Pratt parsing.
Consequently, I renamed the `nud` function to `primary()` and `led` function to `binary_op()`.

I hacked code into the expression routine `exprn()` to handle the `if (1 < 2 && x = 3)` case mentioned in the [previous section](./awk_parsing_is_tricky.html).
(This is also an issue for some similar cases, such as `1 && x = 3` and `2 < x = 3`.) 
It's not elegant, but it seems to work correctly.

### Statement termination

Another feature of awk is that statements normally end at the end of a line and do not require a semicolon to end the statement.
Some constructs require a semicolon in certain places: in `if (condition) statement₁; else statement₂`, where the `else` appears on the same line as statement₁ _and_ statement₁ is a single statement (i.e. not a compound statement in `{braces}`, and in `do statement; while (condition)` where the `while` appears on the same line as the statement and the statement is not enclosed in braces.
Multiple statements on the same line must be separated with semicolons.

The awk(1) man page says "Statements are terminated by semicolons, newlines or right braces."
Actually, all statements must be terminated with a newline or semicolon, _except_ the last statement before a closing right brace in brace-enclosed statement list.
The POSIX grammar has complications due to this, in that it has productions for `terminated_statement_list`, `unterminated_statement_list`, `terminated_statement`, `unterminated_statement`, and `terminatable_statement`.

The nawk/nnawk grammar avoids all this, via a lexical trick.
Since it is never wrong to put a semicolon before a closing brace, and it cannot change the meaning of a correct program, the nawk/nnawk lexical analyzer returns a faked semicolon before each closing brace.
Then it can use a much simpler grammar structure.
But you can detect this trick, because it can also make an incorrect program correct!

Try `awk 'BEGIN{while(n-->0);}'` with any awk and it will do nothing, silently.
But `awk 'BEGIN{while(n-->0)}'` (note missing semicolon empty statement) works in nawk/nnawk but gets a syntax error in gawk, goawk, bbawk, and wak.
(It also works in mawk, indicating that mawk uses the same lexical hack.)

I initially intended to do the same, but it seemed too hacky.
Instead, when I encounter an `else` or the `while` of a `do ... while` statement, I look at the previous token to see if it's any of newline, semicolon, or closing brace.
This has the equivalent effect and also avoids overly complicating the grammar and parsing logic.

### Some other parsing-related problems

#### print and printf

The `print` and `printf` statements have some issues.
These can be followed by one or more expressions, which may or may not be enclosed in parentheses.
So, you may encounter
```
print   # by itself, means print $0, the entire record
print 3, 4      # prints 3 4    # case 2
print(3, 4)     # same effect   # case 3
print(1+2), 4   # same effect   # case 4
```
When all you have is `print (` the parser cannot know if it's case 3 or 4 above, so it cannot know to look for an expression (case 4) or a list of expressions (case 3).
Also, when it sees a list of expressions (case 3), it still cannot decide it's a list to be printed, or the beginning of a subscripted `in` expression, as in `print(3, 4) in a`.

Any of the above print statements can be followed by redirection, as `print 3, 4 > "some_file"`.
This would make `print "def" > "abc"` ambiguous, so the language spec says that any comparison operator in a print statement must be inside parentheses, as in `print("def" > "abc")`, or `print("def" > "abc"), 123 > "outfile"`.
Actually, only the original nawk/nnawk enforces that against operators other than `>`.
Other awks allow `print "abc" < "def" > "outfile"`, for example.

I had a nasty kluge of switches and state variables in place to deal with all this, but now have it simplified somewhat.
In the `getlbp()` function (that returns the Pratt left binding power), I have logic to say if the parser is inside a print/printf statement, and not inside parentheses, and has a `>` token, return 0, which will end any expression.
Then the `>` will be correctly treated as a redirection operator.

The `primary()` function returns the number of expressions in a parentheses-enclosed statement list, or 0 (or -1 for a potential lvalue, that is used to help with the `(1 && a = 2)` problem mentioned in the previous [previous section](./awk_parsing_is_tricky.html)).

The `print_stmt()` function calls the expression parser as `exprn(CALLED_BY_PRINT)`, where `CALLED_BY_PRINT` is a "magic" value that flags the `exprn()` function to see if the initial `primary()` call returns a statement list (>= 2) followed immediately a token that can end a print statement ('>', '>&#x200B;>', '|', ';', '}', or newline).
If so, it returns the expression count from `primary()` to `print_stmt()`, otherwise it continues parsing what is expected to be the first (or possibly only) expression for the print statement.

I believe this correctly handles the print/printf statement.


#### Data types and scope

There are two main data types in awk: scalar and array, maybe three if you count regex literals (`/regular expression here/`) as a separate type.
Scalars can be numeric (always floating point), string, or both, which might be considered sub-types.
You may consider some special values as _numeric string_ (defined in POSIX) or _uninitialized_ as sub-types as well.
Arrays are associative, "subscripted" by string values.
I often refer to them as maps; they are implemented with hash tables in `wak`.

The scope rules are pretty simple.
Function formal parameters (variables supplied in the function definition `function _name_(param1, param2,...)`) are local to the function, as far as name scoping is considered.
All other variables are global.

When a function is called with a scalar variable, the call is by value and changes made to it within the function have no effect outside the function.
But when a function is called with an array variable, changes made to the array are visible outside the function.
A function can be called with fewer arguments than it has defined as formal parameters.
The "extra" parameters will then be uninitialized local variables inside the function, and this is the only mechanism for defining local variables in a function.

This presents some problems for a one-pass parser.
When the parser first encounters a variable, the type (scalar or array) is noted depending on whether or not it is followed by a subscript, or if it appears after the `in` operator.

With this: `BEGIN{f(a); g(a)}` a one-pass parser can't know if `a` is scalar or array.
If it's an array, it needs to be set up before calling `f(a)` so awk can pass the array to `g()`.

In `BEGIN{f(a); g(a)}; function f(x){x[1]=3}; function g(x){print x[1]}`, goawk / gawk / mawk / nnawk (OneTrueAwk) and bbawk all work.

But worse, it is possible for a function to be called with a scalar sometimes and an array (as the same parameter) other times.
For example:
```awk
BEGIN{f(1); f(0)}
function f(a, b){if (a) g(b); else {h(b); i(b)}}
function g(x){print x}
function h(x){x[1] = 37}
function i(x){print x[1]}
```
Here, gawk / nnawk / bbawk all print an empty line and then 37; mawk and goawk complain about type errors.
Posix says "The same name shall not be used within the same scope both as a scalar variable and as an array."
But I don't see how that can be determined during compilation of f(a, b) in a one-pass parser.
I'm curious how this works inside gawk, nnawk and bbawk but not going to look.

In compiling f(a, b), g(b) will pass a scalar and h(b) and i(b) will pass an array.
In my parser, if a variable appears only as a "bare" variable name in a function call, it is left "untyped" (actually gets a ZF_MAYBEMAP flag).
When it is pushed on the stack at runtime, an empty map is attached to it.
Then if it is used as a scalar, it is converted to scalar and if it is used as an array it is converted to an array.

### Error recovery

I am using D.A. Turner's error recovery technique to handle syntax errors and continue parsing.

Dick Grune's [Annotated Bibliography](https://dickgrune.com/Books/PTAPG_2nd_Edition/CompleteList.pdf) of papers on parsing includes this reference for Turner's method:

> Turner, D. A. Error diagnosis and recovery in one pass compilers. Inform. Process.
Lett., 6:113–115, 1977.
> Proposes an extremely simple(minded) error recovery method for recursive descent parsers: when an error occurs, the parser enters a recovering state. While in this recovering state, error messages are inhibited. Apart from that, the parser proceeds until it requires a definite symbol. Then, symbols are skipped until this symbol is found or the end of the input is reached. Because this method can result in a lot of skipping, some fine-tuning can be applied.

