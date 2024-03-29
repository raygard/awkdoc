---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
nav_order: 1
---

# An awk implementation

Rob Landley's [toybox](https://landley.net/toybox/) [project](https://github.com/landley/toybox) provides a variety of Linux command-line tools, similar to [busybox](https://busybox.net/).
I have written a compact but fairly complete `awk` implementation intended to integrate with toybox, but it can also build standalone.

This implementation is named `wak`, because all the good `awk` names are taken.
But when used in toybox, it's just `awk`, or `toybox awk`.

`wak` is coded in C99. It uses mostly C standard library, aka `libc`, but also needs some POSIX support, mostly for regular expressions.

It is not public domain but does have Landley's very liberal [0BSD](https://landley.net/toybox/license.html) license.

These pages describe my awk implementation, but are not complete.

The implementation is at [github](https://github.com/raygard/wak/).
