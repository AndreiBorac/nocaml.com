# QuickStart

### Purpose and Assumptions

This quick start tutorial is aimed at bringing the reader up to speed
with `Nocaml` such that the reader will be able to write `Nocaml`
programs, write any necessary builtins in `C`, and set up a `Nocaml`
runtime environment and invoke `Nocaml` code from `C`.

It is assumed that the reader has at least a rudimentary grasp of the
essentials of `OCaml`, `Ruby` and `C`.

### Brief tour of `Nocaml`

Let's start by cracking open `Nocaml`'s `stdlib.lisp`, which is, of
course, written in `Nocaml`. Specifically, let's take a look at the
`not` function:

```
(defun not (x)
  (case x
        ((False) true)
        ((True) false)))
```

As you can see, the syntax is vaguely reminiscent of `Lisp`. In this
example, `True` and `False` are type constructors, while the lowercase
variants `true` and `false` happen to be "primordial" objects that are
statically allocated outside the `Nocaml` heap. Note that `case` is
analogous to `OCaml`'s `match ... with` construct, except it isn't as
powerful - it only supports destructing a single level.

That wasn't a very exciting function though. Let's take a look at
`reverse-rec`:

```
(defun reverse-rec (lista listb)
  (case lista
        ((ListFini)
         listb)
        ((ListCons head tail)
         (reverse-rec tail (ListCons head listb)))))
```

Now here's a function that's actually recursive. `ListFini` and
`ListCons` are type constructors. As you can see, destructing
`ListCons` yields two variables, here given the names `head` and
`tail`. In the last line, the constructor `ListCons` is used to create
a new object. The result of `reverse-rec a b` should be `(concat
(reverse a) b)`, except it can't be implemented that way because
`concat` requires `reverse-rec` for its implementation.

### Where types, builtins and primordials come from

Types and builtins are declared in "companion" `Ruby` files like
`stdlib.rb`. Such files can also define primordials. Such files
consist of invocations of the following functions:

* `wombat_enable_ocaml` enables `OCaml` code generation for strong
  static typechecking. this should almost always be on, and `stdlib`
  indeed turns it on

* `wombat_register_primordial` registers a primordial object,
  accepting its name and `C` definition

* `wombat_register_builtin` registers a builtin, accepting its name
  and arity

* `wombat_register_constructor` registers a constructor, accepting its
  name and arity

* `wombat_register_constructor` registers a native constructor,
  accepting its name and arity. Native constructors cannot be
  destructed (`case`d) in `Nocaml`, and they are always initialized to
  zero when created

* `wombat_ocaml` registers `OCaml` code to emit during `OCaml` code
  generation. `Nocaml` has no facility for creating corresponding
  `OCaml` types or builtin function signatures so these must be coded
  by hand at present

Some additional notes that will assist the reader in understanding
`stdlib.rb`:

* Blobs will be described in another section. Do not concern yourself
  overly with the blob-related statements at present. Also, the
  blob-related statements only have to be implemented once, in
  `stdlib` -- a typical user of `Nocaml` will only be interested in
  defining custom types, primordials and builtins

* Notice how primordials are plainly defined as `static const
  uintptr_t` arrays in `C`

* There are two sets of `OCaml` code. The first set (`wombat_*`) is
  for validating everything except the constraint that there be no
  partial function application. The second set (`wombatx_*` validates
  that no partial function application takes place. This requires the
  trivial transformation of, for each function, wrapping `N` arguments
  into a `vectorN` object. Thus every function is converted into a
  single-argument form, and partial application becomes
  impossible.

### Understanding blobs

In `Nocaml`, the memory representation of each object begins with a
"low" tag that contains the ID of a wombat or native
constructor. Alternatively, the tag can be a "high" tag whose high bit
is set. In this case, the tag declares a "blob" whose length in bytes
is given by the lower `W-1` bits of the tag (where `W` is the bit
length of a machine word).

When providing a facility to create a `Blob`, it was desired that
`wombat` remain agnostic with respect to any defined types. Therefore,
instead of allowing the creation of a `Blob` out of an `Integer`
object, `wombat` allows the creation of a `Blob` out of any object
whatsoever. The given object is passed to the special `wombat_measure`
function to be defined by the `C` code. This function is defined by
`stdlib` in an implementation that asserts that its argument is
`Integer`.

### Understanding allocation and mutation

In `Nocaml`, allocation of `Nocaml` objects always takes place (A) by
driver `C` code when no `Nocaml` code is executing or (B) in `Nocaml`
code. Builtins cannot allocate. Loosening this restriction is
technically possible but would add overhead and complexity.

Thus, if a builtin were otherwise to require allocation, the
allocation is first performed in `Nocaml` and passed into the
builtin. For example, addition is **not**:

```
(let ((c (int-add a b))) ...)
```

but is rather:

```
(let ((c (int-add a b (Integer)))) ...)
```

The policy is to allow a limited form of mutation, making `Nocaml` not
technically pure. The specific concession made in `stdlib` is to check
that the object being mutated is freshly allocated at the front of the
heap. This allows small, local, harmless mutations while prohibiting
the types of mutations that can cause confusion in large programs.

### A quick look at builtins

Let's open `stdlib.c` and check out a builtin:

```
WOMBAT_BUILTIN static uintptr_t* wombat_builtin_blob_minus_length(WombatExternal* wombat_external WOMBAT_UNUSED, uintptr_t* wombat_context WOMBAT_UNUSED, uintptr_t* cella, uintptr_t* cellb)
{
  STDLIB_CHECK_TOP(cellb);
  uintptr_t bit = (1UL << ((8*sizeof(uintptr_t))-1));
  assert((cella[0] & bit) != 0);
  assert((cellb[0] == WOMBAT_NATIVE_CONSTRUCTOR_Integer));
  cellb[1] = (cella[0] - bit);
  return cellb;
}
```

This builtin gets the length of a blob. The `_minus_` in its name
corresponds to a literal `-` character. `STDLIB_CHECK_TOP` is a macro
that checks that an object is at the top/front of the heap. The first
assert checks that the `cella` is a blob. The second checks that
`cellb` is an `Integer`. The next line sets the payload word of the
`Integer` object to the length of the blob.

`WOMBAT_BUILTIN` is a macro that expands to an attribute informing
`gcc` and `clang` that this function should not be subject to
inlining.

`WOMBAT_UNUSED` is a macro that expands to an attribute informing
`gcc` and `clang` that this parameter is not necessarily used. In this
case, `wombat_external` is used by `STDLIB_CHECK_TOP` to determine the
heap pointer.

In no cases would `wombat_context` apply to a builtin (it is there
because it is required to keep function signatures uniform within
`wombat` -- keep in mind that builtins can be passed around as
variables within `Nocaml`).

### Diving into the driver

The `driver.c` for the sample quicksort program illustrates how to set
up a `Nocaml` environment, allocate `Nocaml` data structures from `C`,
and call into `Nocaml` code from `C`.

The `Nocaml` specifics start thus:

```
#define NELM (1536*1024*1024/8)
static uintptr_t memory[NELM];
static uintptr_t bitmap[((NELM+((8*sizeof(uintptr_t))-1))/(8*sizeof(uintptr_t)))];
static uintptr_t ctrmap[(sizeof(bitmap)/sizeof(uintptr_t))];
#define STKMAPLEN 32768
static uintptr_t stkmap[STKMAPLEN];
```

In this example, `memory` allocates `1.5GiB` of heap (on 64-bit
systems -- this is not scaled for 32-bit because we are primarily
concerned with being able to sort a specific number of machine
words). The collector requires three additional areas of memory. The
first is the "bitmap" which must have one bit per word, rounded up to
an even number of words. The second is the "counter map", which must
have the same size as the bitmap. The third is the "stack map" which
must be large enough to accomodate all the pointers into the heap that
are stored on the stack. Usually the stack map need not be very large
at all, but in this case it is large to allow running code that has
been compiled with `-g` (which does not eliminate tail recursion).

Next is the setup of a `CollectorExternal` object on the stack:

```
    CollectorExternal external;
    memset((&external), 0, sizeof(external));
    external.heap.bas = memory;
    external.heap.end = memory + NELM;
    external.heap.ptr = external.heap.end;
    external.bmap.bas = bitmap;
    external.cmap.bas = ctrmap;
    external.smap.bas = stkmap;
    external.smap.end = stkmap + STKMAPLEN;
```

There is nothing remarkable about the above. The next line shows how
to turn a primordial into an ordinary `uintptr_t*` `wombat` object,
basically by casting away `const`:

```
    uintptr_t* rest = ((uintptr_t*)(wombat_primordial_list_minus_fini));
```

Next there is a loop:

```
    for (uintptr_t i = 0; i < QSELMS; i++) {
      uintptr_t v = ((uintptr_t)(rand()));
      rest = wombat_constructor_3(&(external.wombat), WOMBAT_CONSTRUCTOR_ListCons, wombat_constructor_2(&(external.wombat), WOMBAT_NATIVE_CONSTRUCTOR_Integer, ((uintptr_t*)(v))), rest);
    }
```

This constructs a list by prepending an element each iteration. There
are two constructor invocations per iteration, one for the list link
and one for the `Integer`.

Finally, the invocation:

```
    rest = collector_invoke((&external), ((void*)(wombat_defun_main_minus_quicksort_minus_ntimes)), NULL, rest, n, NULL, NULL, NULL, NULL);
```

In `Nocaml`, invocations are always made with 6 arguments (any
additional arguments are "don't cares"). After invocation, `Nocaml`
will conduct a cycle of compacting garbage collection, keeping only
the result of function invocation. All objects to be kept must be
included in the result.

After the quicksort example, there is an example using blobs. An input
blob containing the string "hello" is replicated 100 times, resulting
in a blob of length 500, that is converted to a string in the `C`
code.
