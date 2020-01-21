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
