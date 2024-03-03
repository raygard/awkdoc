---
title:  Writing an awk
layout: page
nav_order: 2
date:   2024-01-24 05:00:00 -0600
---

# Writing an awk

## Introduction

I have written a compact but fairly complete `awk` implementation intended to integrate with Rob Landley's [toybox](https://landley.net/toybox/) [project](https://github.com/landley/toybox), but it can also build standalone.

These pages document some aspects of this effort.
This implementation is named `wak`, because all the good `awk` names are taken.
But when used in toybox, it's just `awk`, or `toybox awk`.

It is not public domain but does have Landley's very liberal [0BSD](https://landley.net/toybox/license.html) license.


## Who is this written for?

This is written for anyone who wants to understand how `wak` works internally.
It's partly for my own use, to document what I've done so I can find it later.
It's also for anyone curious about how awk can be implemented, including some of the problems I encountered and how I dealt with them.

To understand this, it helps to know awk pretty well and know a bit about how a lexical analyzer (aka lexer / scanner / tokenizer) and a recursive-descent parser work.

There is a [POSIX specification](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/awk.html) for awk, and `wak` should conform to it with some exceptions (mostly) documented here.
The reader should probably be familiar with the POSIX spec to read this, or the `wak` code.

## Other implementations

There are several implementations of awk readily available, which I have been testing my implementation against.

The original [awk](https://github.com/onetrueawk/awk), written by Al Aho, Peter Weinberger, and Brian Kernighan, is still being maintained and even updated recently with new features.
Kernighan (bwk) calls it the One True Awk, and it's often referred to as `nawk` (new awk, as compared with the pre-1985 version, sometimes called old awk, which lacked many essential features).

Around September 2023, `nawk` was updated with some new features, primarily around UTF-8 handling and the ability to read CSV (comma-separated values) files.
I have used this later version in my testing, and refer to it here as `nnawk` (new new awk).

Many Linux distros include [gawk](https://www.gnu.org/software/gawk/) (GNU awk, maintained for decades by Arnold Robbins).
Some distros (like Raspberry Pi's Raspian) have [mawk](https://invisible-island.net/mawk/) (Mike Brennan's awk, now maintained by Thomas Dickey).

Busybox has an [awk implementation](https://git.busybox.net/busybox/tree/editors/awk.c) by Dmitry Zakharov, written in 2002 and currently maintained.

More recently, Ben Hoyt wrote [goawk](https://github.com/benhoyt/goawk), an implementation in Go language.

See also [https://en.wikipedia.org/wiki/AWK](https://en.wikipedia.org/wiki/AWK).

## Documents

The 1985 version of `nawk` is documented in *The AWK Programming Language* (1988), by Aho, Kernighan, and Weinberger.
I'll refer to that edition as the *AWK book*.
Since `nnawk` (aka One True Awk or OTA) has been enhanced in 2023 by Kernighan, the book has been issued in a [second edition](https://awk.dev) available at [Amazon](https://www.amazon.com/Programming-Language-Addison-Wesley-Professional-Computing/dp/0138269726/).
I'll refer to that as the *AWK book 2nd Ed*.

In addition to *The AWK Programming Language* (1st and 2nd editions) and the POSIX spec mentioned above, the [gawk manual](https://www.gnu.org/software/gawk/manual/) by Arnold Robbins, titled *GAWK: Effective AWK Programming* is a great resource.
It documents gawk in detail, including "standard" awk and gawk-only extensions, and it also explains how to use awk.
The explanations of awk's "dark corners" and many details omitted from the AWK book helped immensely in developing `wak`.

## What makes a proper awk?

None of the existing implementations follow POSIX to the letter, and no two of them behave exactly the same.
As I understand it, the POSIX awk spec is largely based on the description in the AWK book, but there are some differences.
The POSIX spec is more precise than the description in the AWK book, but not as detailed as, for example, the language spec for C.
The grammar in the POSIX spec reflects the actual `nawk` implementation more so than the book does, but is not based directly on the yacc/Bison grammar used by `nawk`.
Where POSIX differs from `nawk`, it seems most implementers tend to follow the actual behavior of `nawk` rather than POSIX.

`wak` intends to follow the [POSIX spec](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/awk.html) as well, but there are departures.
For example, `wak` does not handle locales except to the extent that toybox does.

## Some restrictions imposed by toybox

Toybox is licensed [0BSD](https://landley.net/toybox/license.html), Rob Landley's approach to have copyrighted code that is as free and close to public domain as possible; see the link for more information.
Rob does not accept code that has more restrictions than the 0BSD license, so he can't adopt any other current awk implementations.
`wak` uses the 0BSD license.

Rob is adamant that it not have any outside dependencies except "standard" libc (plus some POSIX elements such as regular expression functions), so using yacc/Bison or lex/flex is out, and toybox awk will have to use a hand-written lexical analyzer and parser.

In order to avoid any "contamination" of `wak` code by other implementations, I did not look at the code for other implementations.
I did look at the yacc/Bison grammar for `nawk`, only to attempt to understand the actual language accepted by the original awk.

## Program structure

The design is a classic "scan, compile to internal format, interpret" interpreter.

The compiler is a recursive descent parser with code generation rolled directly into it.
It uses a Pratt/precedence-climbing approach for parsing expressions.
It is a one-pass design, with a few "fixups" needed to deal with awk allowing functions to be called before being defined, and also to handle a few difficulties with the awk grammar.

The internal format is a "virtual machine" where the instructions are each a "word" (`int`).
For no special reason, I call my generated format "zcode".
The zcode machine is a straight stack machine.

### Lexical analysis

The current `wak` lexical analyzer, `scan.c`, is about 300 LOC (counting non-blank, non-comment lines).
It's a fairly typical scanner design, with functions to get tokens (numbers, strings, variables, keywords, regular expressions), look up keywords, etc.
It also handles reading the program text given on the command line or stepping through the files of awk code specified on the command line.

One interesting detail is the handling of regular expressions (hereafter just "regex").
The language allows literal regex specified as `/ ... regex here .../`.
The `/` symbol also indicates division, and `/=` is the divide-and-assign operator, just as in C.
To disambiguate, the POSIX spec says:
> When an input sequence begins with a <slash> character in any syntactic context where the token '/' or DIV_ASSIGN could appear as the next token in a valid program, the longer of those two tokens that can be recognized shall be recognized. In any other syntactic context where the token ERE could appear as the next token in a valid program, the token ERE shall be recognized.

(DIV_ASSIGN is the '/=' token, and ERE means a POSIX Extended Regular Expression token.)

I asked Arnold Robbins and Ozan Yigit (the current OneTrueAwk maintainer) if I am correct that a '/' or '/=' token can appear only after a number, string, variable, `getline` keyword, right parenthesis ')', right bracket ']', or an increment or decrement operator '++' or '--'.
I intended to have the scanner assume a '/' character after any of these means divide, and a '/' in any other context indicates a regex literal.

Neither gave me a definitive answer to that, but Ozan did not think it a good idea to have the scanner make the decision.
He said "letting the parser inform the scanner what to collect is a reasonable thing to do" and that "both ota and gawk do exactly that for collecting regular expressions."
That implied to me that the parser would have to tell the scanner when it is expecting a regex, which seemed to me to be trickier than having the scanner do as I suggested.
Perhaps I was wrong, but I did not want to look at the code for OTA (nawk) or gawk to see just how that is done.
I have taken the approach I described, and have not run into a problem yet with valid awk code being scanned or parsed wrongly as a result.

### Internal format and `zcode` machine

Each awk value is a `zvalue` `struct`, that can be a scalar, literal regular expression (regex for short), or an array.
A scalar can have a numeric value (C `double`) or a string value or both.
A string (`zstring`) is a `struct` that is reference counted and can hold an arbitrary number of bytes.
An array is a pointer to a `zmap` `struct` that holds a pointer to a hash table.
(I usually refer to the awk array structure as a map.)
A regex is a pointer to a `regex_t` compiled POSIX regex.

Because the string, array, and regex values are mutually exclusive within an awk variable, I use a union to hold them.
Here are the zvalue and zstring structures:
{: .lh-tight }
```c
// zvalue: the main awk value type!
// Can be number or string or both, or else map (array) or regex
struct zvalue {
  unsigned flags;
  double num;
  union { // anonymous union not in C99
    struct zstring *vst;
    struct zmap *map;
    regex_t *rx;
  };
};

// zstring: flexible string type.
// Capacity must be > size because we insert a NUL byte.
struct zstring {
  int refcnt;
  unsigned size;
  unsigned capacity;
  char str[];   // C99 flexible array member
};
```

The anonymous union in the `struct zvalue` is not valid C99, but gcc accepts it, giving a warning with -Wpedantic.
I may fix this, but maybe it just clutters up the code with extra `.z` characters for no really good reason.
This is the only non-standard aspect of `wak` code, as far as I know.
With all gcc warning options turn on, it compiles with no other warnings.

The stack, several compiler tables, and also the actual value structures of the arrays are held in expanding sequential list structures called `zlist`:
```
// zlist: expanding sequential list
struct zlist {
  char *base, *limit, *avail;
  size_t size;
};
```
The `size` member holds the size of each entry in the list (also referred to as slots).
`base` points to the base of the list; `limit` points to just past the end of the list, and `avail` points to the first unused slot.
The list is reallocated to be 50% larger as needed as it fills up.

The map (awk array) structure is the `zmap` hash table:

```
// zmap: Mapping data type for arrays; a hash table. Values in hash are either
// 0 (unused), -1 (marked deleted), or one plus the number of the zmap slot
// containing a key/value pair. The zlist slot entries are numbered from 0 to
// count-1, so need to add one to distinguish from unused.  The probe sequence
// is borrowed from Python dict, using the "perturb" idea to mix in upper bits
// of the original hash value.
struct zmap {
  unsigned mask;  // tablesize - 1; tablesize is 2 ** n
  int *hash;      // (mask + 1) elements
  int limit;      // 80% of table size ((mask+1)*8/10)
  int count;      // number of occupied slots in hash
  int deleted;    // number of deleted slots
  struct zlist slot;     // expanding list of zmap_slot elements
};
```

The `zlist slot` member holds the actual values of the hash table:

```
// Elements of the hash table (key/value pairs)
struct zmap_slot {
  int hash;       // store hash key to speed hash table expansion
  struct zstring *key;
  struct zvalue val;
};
```

The `zmap` `hash` member points to 2<sup>n</sup> `int` values that in turn index to `zmapslot` `structs`, each of which holds a `zvalue`, a key as a `zstring`, and its hash (to speed rehashing, a la Python).
The hash table grows by doubling as needed, starting at 8 slots and uses a probe sequence borrowed from Python: `(*probe = (*probe * 5 + 1 + (perturb >>= PSHIFT)) & m->mask;)`.
See comments in the [Python dict source code](https://github.com/python/cpython/blob/main/Objects/dictobject.c) for details; `wak` uses a simplified version of this approach.

