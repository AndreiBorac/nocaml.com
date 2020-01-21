# Nocaml

`Nocaml` is a compiled, nearly pure, functional programming language
with a lightweight runtime that includes a precise compacting garbage
collector. `Nocaml` is well-suited to use in embedded systems by
virtue of having a very small and space-efficient runtime.

`Nocaml`'s reference compiler is implemented in `Ruby`, and it can
convert a `Nocaml` program into `C` code for efficient
execution. `Nocaml` also supports strong static type checking by
conversion to `OCaml`.

### Performance

Since it compiles to `C`, `Nocaml` achieves superior performance to
typical interpreted runtimes for embedded systems. Granted, `Nocaml`
is currently somewhat slower than `OCaml` (when both are compiled to
native code). Below is a comparison of the performance relative to
`OCaml` on a basic non-mutating list-based `quicksort` program:

![barchart](https://raw.githubusercontent.com/AndreiBorac/nocaml.com/master/graph001.png)

The `OCaml` code was run with `OCAMLRUNPARAM=h=201326592,o=4000` to
completely remove `GC` memory pressure during the
computation. Instead, a `GC` cycle was forced at the end of the
`quicksort` calculation to mimic `Nocaml`'s behavior of `GC`ing before
returning to native code. For some reason, `Nocaml` exceeds `OCaml`
performance on the first data point. While the rest of the data points
are all slower than `OCaml`, `Nocaml` manages to stay above `70%` of
`OCaml` performance in all cases.

Future work to improve garbage collector performance in `Nocaml` is
planned.

### Language features

`Nocaml` supports:

* ML-like data-types with constructors (currently defined via `Ruby`
  code).

* a slight generalization of ML in which objects may be of any
  constructor of any type. If this is used, strong static type
  checking is no longer possible.

`Nocaml` doesn't support:

* partial function application (the absence of partial function
  applications is enforceable in the `OCaml`-based static type
  check). Naturally, it is possible to define functions that
  essentially perform partial function application, but they must be
  used explicitly not implicitly.

* fancy multi-level `match` as in `OCaml`. Instead one can switch over
  the possible constructors for one expression via a `case` statement.

* macros (but there are plans to allow macros implemented in `Ruby` as
  `AST` transformations).

### Platforms

`Nocaml` supports `x86_64` (`SYSV` calling convention) and 32-bit
`MIPS` (`O32` with `ABICALLS`). Support for additional platforms is
relatively easy to add -- there are just two simple functions that
need to be written in assembly.

### Principle of operation

You may be wondering, how is `Nocaml` able to do precise garbage
collection on a C program? The trick is that the C code output by
`Nocaml` manipulates only pointers, not arbitrary integer values. Thus
a stack location that has a value that appears to point into the heap
can be assumed to actually be a pointer and subject to adjustment if
the heap object it points to is relocated. Actually, "small" integer
values and pointers into static memory are also manipulated but these
are thankfully not problematic. The drawback to this approach is that
it is only possible to manipulate integers through builtins via
function calls that cannot be inlined.

### License and source code

`Nocaml` is made available under the terms of the `MIT` license. The
source code for the project is hosted on `Github`, click
[here](https://github.com/AndreiBorac/nocaml) to access it.

### More info

Check out the [FAQ](FAQ.md) and the [QuickStart](QuickStart.md) pages.

