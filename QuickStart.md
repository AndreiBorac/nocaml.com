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
