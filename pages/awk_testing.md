---
title:  Testing awk
layout: page
nav_order: 7
date:   2024-01-24 05:00:00 -0600
---

## Testing awk

I've been testing `wak` against the other awk implementations I've been able to obtain.
I have recent versions of `nnawk` (Kernighan's One True Awk, the original Unix awk updated), `gawk`, `mawk`, `goawk`, and `bbawk` (busybox awk).

As of this writing, the versions are:

nnawk: awk version 20231228 (compiled 2024-01-23)    
gawk: GNU Awk 5.3.0, API 4.0, PMA Avon 8-g1 (source 2023-11-02; compiled 2023-11-19)    
mawk: mawk 1.3.4 20231102 (compiled 2023-11-16)    
goawk: V1.25.0  (compiled 2024-01-23)    
bbawk: 2023-12-31 (compiled 2024-01-23)    

I'm sure there are plenty of bugs.
Brian Kernighan has until recently been maintaining the original Unix awk from the start almost 40 years ago and has a "FIXES" file with over 200 entries from 1987 to 2023, and continuing to the present in a separate file for the "second edition" of One True Awk.
If Kernighan has been fixing bugs for 35+ years, I doubt I can make a bug-free awk.

Some "bugs" are incompatible interpretations of awk compared with what other implementations do with certain features.
No two versions of awk (original awk, gawk, mawk, goawk, Busybox awk, etc.) agree completely.

Please report bugs to raygard at gmail.com.

### Testing strategy

I have used the test files that come with existing awk implementations, plus some I've written.
The original One True Awk comes with a folder `testdir` of about 315 files.
Kernighan's README.TESTS file says there are about 60 small tests `p.*` from the first two chapters of *The AWK Programming Language* (1st ed.) that are basic stuff; about 160 small tests `t.*` that are "a random sampling of awk constructions collected over the years.  Not organized, but they touch almost everything."; about 20 `tt.*` files that are timing tests.

The `testdir` folder has also about 30 `T.*` files that are "more systematic tests of specific language features", but unfortunately these are shell scripts that can test a single awk program to see if it computes correct output as compared with known good data built into the scripts.
This makes it difficult to use to compare my implementation against all the others in one pass, but I can run the scripts on `wak` separately.

`gawk` also comes with a folder of about 1475 files, and most of these are sets of `foo.awk`, `foo.in`, and `foo.ok` files.
In each case, the `foo.awk` file is run with `foo.in` input and the result can be compared with `foo.ok`.
Some are standalone tests that do not need an input file, so there is sometimes no `foo.in` file.

I have a not-very-neat test driver `test_awk.py` that I can use to run a batch of tests, such as all `t.*` in `testdir`, at one time against several awk implementations, and see how they compare.
In the case of `testdir`'s `p.*` and `t.*` files, they are intended to use certain input files (`test.countries` and `test.data`), and the outputs are compared via MD5 hashes.
Each unique output is saved for later examination.
For the `gawk`-style tests, the program can compare the output against the `foo.ok` file and give a pass/fail result.
If there is a non-zero return code or an exception, that is noted on the `test_awk.py` output.
The output looks like this:
``` txt
====versions====                 nnawk    gawk    mawk   goawk   bbawk   tbawk   muwak
                                  ====    ====    ====    ====    ====    ====    ====
Test delarpm2.awk              8841567 dd8e2e5   nnawk c992867    GAWK   GOAWK   GOAWK
Test dfacheck1.awk             0000000 03a19ad   nnawk   nnawk    GAWK !3a19ad   TBAWK
ERR: tbawk: awk: file tests/gawktests/dfacheck1.awk line 1: warning: '\<' -- unknown regex escape

ERR: muwak: muwak: file tests/gawktests/dfacheck1.awk line 1: warning: '\<' -- unknown regex escape


Test double1.awk               819e6db 8c7dbdf    GAWK 351564a d97b6e5   BBAWK   BBAWK
Test double2.awk               4941b67 4dbdb44    GAWK 7a665a2 acfcfe0 0124355   TBAWK
Test dtdgport.awk              a916caa   nnawk   nnawk   nnawk !!00000   nnawk   nnawk
RET: bbawk: 1
ERR: bbawk: awk: tests/gawktests/dtdgport.awk:37: %*x formats are not supported
```
The hex values are the first 7 digits of the MD5 of the output file.
If the output is an empty file, the MD5 is replaced with all zeroes to make it easier to spot.
If any stderr output occurs, the first digit is replaced with a '!'; if a non-zero return code occurs, the second digit is replaced with '!'.
In either case, the stderr output and return code are printed.
These (non-pass-fail, non-timing) reports always display a hash value of the output for the first column (i.e. the first awk version tested).
In subsequent columns, if the hash is different from the first column, that hash is listed; but if hash matches a hash from a previous column then the awk version is listed, and if it differs from the first column it is up-cased.

So for example, for `delarpm2.awk`, `gawk` has a different output from `nnawk`, `mawk` matches `nnawk`, `goawk` has yet another different output, `bbawk` matches `gawk`, and both `tbawk` (toybox `awk` -- my awk for toybox) and `muwak` (my awk compiled with `musl` libc) match `goawk`.
For `dfacheck1.awk`, `nnawk` produced no output, `gawk` gave some output, `mawk` and `goawk` also produced no output, `bbawk` matched `gawk`, `tbawk` produced the same output as `gawk` but had stderr output, and `muwak` matched `tbawk`, including having stderr output.

The `gawk` tests were originally intended to be run via the supplied `Makefile`, and some of them use special `gawk` options, environment setup, etc., so that when the `foo.awk` file is run by `test_awk.py` it may not produce correct `foo.ok` output even from `gawk`.
Because of this, I sifted the output from all the `gawk` tests against all the awk versions into several parts and moved the tests into corresponding folders: `gawktests\allfail` has tests that fail for all versions, including `gawk`; `gawktests\allpass` has tests that pass for all versions; `gawktests\gawkonly` has tests that pass for `gawk` and fail for all others (usually because they use gawk-only features); and `gawktests` has all the remaining tests.



