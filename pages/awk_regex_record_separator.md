---
title:  Awk -- implementing regex RS
layout: page
nav_order: 6
date:   2024-01-24 05:00:00 -0600
---

# Writing an awk

## awk -- implementing regex RS

The POSIX spec (and the original AWK book) only accepts a single character as the record separator, the default being `<newline>`.
And POSIX says "If RS contains more than one character, the results are unspecified."
I noticed that all the awk implementations I have available have implemented the capability of using a regular expression for RS, the record separator string.
This includes nawk (the original Kernighan "One True Awk" in the 2022 version) and nnawk (the latest version), gawk, mawk, goawk, and Busybox awk (bbawk).
I believe mawk was the first to implement this feature; it was added to nawk in 2019.
Maybe it will be in the next release of the POSIX spec for awk.

So I decided to try adding it to my version of awk.
This proved a bit trickier than it looked at first.

I had noticed that Busybox awk fails the `rebuf.awk` test in the gawk test set.
The problem in the bbawk output looked like the same sort of problem that the rebuf.awk test (a bug reported in 2002) originally revealed in gawk.
Looking at the nature of the output, where the regex `RS="ti1\n(dwv,)?"` was not being matched on some records, causing "dwv," to appear spuriously in the output, I guessed that the problem was that the data at the end of the buffer contained only a partial match for the regex.
That is, if the buffer ended with "ti1\n", the RS would match, but the "dwv," would be at the start of the next buffer.
But when I tried to implement it, I found my code was getting similar results.
I had logic such that if I got a match ending exactly at the end of the buffer, then expand the buffer, read more data, and try again.
The problem was that I could get a partial match _near_ the end of buffer, and have the same problem.
For example, if the buffer ends with "ti1\ndw", the RS will match the "til\n".
So I have to expand the buffer whenever the match is near the end of the buffer.
How near, though?

I wrote a program to stress test the awk versions on hand.
I generated 8500 lines of data that looks like:
```
1wxyz
2wxyyz
3wxyyyz
...
```
And a test:
```
BEGIN{ RS = "w(x[^z]*z\n)?" }
1
```
This did stress them.
Correct behavior would be to print 1, 2, ... 8500 lines with just the numbers on them.
nnawk did the best, running for 8180 lines before printing a couple of anomalous lines, then ran OK for the rest of the input.
mawk printed 8639 lines, with the first 717 correct.
goawk printed 9078 lines, with the first 355 correct.
wak printed 11048 lines, with the first 1170 correct.
gawk printed 12007 lines, with the first 40 correct.
bbawk printed 16828 lines, with the first 25 correct.

wak 1170
bbawk 25
goawk 355
gawk 40
mawk 717
nawk 8180

   8501 nnawk
   8639 mawk
   9078 goawk
  11048 wak
  12007 gawk
  16828 bbawk







