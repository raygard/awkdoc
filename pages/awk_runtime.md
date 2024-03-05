---
title:  Runtime internals
layout: page
nav_order: 5
date:   2024-01-24 05:00:00 -0600
---

# Runtime internals

Much more needed here...

## Function definition, call, return, stack frame design

A function definition `function f(a, b, c,...) { ... }` generates:

```
tkfunction  function_number
(code for function body)
tknumber    uninitvalue
tkreturn    number_of_params
```

As each parameter is parsed, it is added to a table of local variables for this function.
The tkreturn is added to return an uninitialized value if the code falls off the end of the function.

When a `return` keyword is encountered, the expression following it is compiled and its value is left on the stack.
If no expression follows the return, then a `tknumber 0` is compiled to push a zero on the stack.


A function call `f(a, b, c,...)` generates:

```
opprepcall  function_number
(code to push args)
tkfunc      number_of_args
```

At runtime, these work as follows:

The runtime keeps an index into the stack called `parmbase` (a local variable of the main interpreter function `interpx()`, initially 0) that points into the current call stack frame as follows:

```
return_value    parmbase-4
return_addr     parmbase-3
prev_parmbase   parmbase-2
arg_count       parmbase-1
function_number parmbase
arg1            parmbase+1
arg2            parmbase+2
...
```

A function call executes as follows:

`opprepcall` pushes placeholder 0 values for the return value, return address, previous parmbase, and argument count.
Then it pushes the function number (from the `opprepcall function_number` code sequence).

The arguments are pushed onto the stack after that.

`tkfunc` retrieves the number of arguments (arg count) from the `tkfunc number_of_args` code sequence, calculates where the parmbase should be by subtracting the arg count from the current stack top index, then fills in the return address (offset of next zcode word in the zcode table) and the arg count in the stack frame.
Finally, the `tkfunc` op pushes the arg count on the stack and sets the `ip` (zcode instruction pointer) to go to the `tkfunction` op of the function definition.

`tkfunction` finds the local variable table for the function and gets the number or parameters.
It then pops the arg count (number of actual arguments) from the stack, calculates the new parmbase (stack top index minus arg count), stores the previous parmbase in the stack frame, and sets `parmbase` to the new parmbase.
Next, it loops to drop excess calling arguments if more args have been supplied in the call than there are params defined for the function. (NOTE: This is an error and should be caught at compile time! FIXME)
Then, if the number of supplied args is less than the number of defined params, additional "args" are pushed to be used as local variables by the function.
This is where the "maybe map" variables may have to be created, as explained elsewhere [FIXME WHERE?].

When the function returns, either via a `return` keyword or "falling off the end" of the function (where a `tkreturn` op has been compiled), the `tkreturn` op picks up the param count (from the `tkreturn number_of_params` code sequence) [NOTE this is unused; remove it? FIXME], gets the arg count from the stack frame, and copies the return value from the stack into the return value slot in the stack frame, and drops the return value from the stack.
At this point, the stack should have on it just the arguments, including the locals created beyond the args supplied.
The `tkreturn` op loops through the locals not supplied by the caller, releasing any map (array) data and dropping the local "arg" from the stack.
Now, only the args actually supplied by the caller remain on the stack, and these are dropped.
Finally, the `ip` instruction pointer is set from the return address in the stack frame, the previous `parmbase` value is restored from the stack frame, and the runtime continues with the next instruction.

