Understanding jaxprs
====================

Updated: February 14, 2020 (for commit 9e6fe64).

(Note: the code examples in this file can be seed also in
``jax/tests/api_test::JaxprTest.testExamplesJaxprDoc``.)

Conceptually, one can think of JAX transformations as first tracing the Python
function to be transformed into a small and well-behaved intermediate form,
the jaxpr, that is then transformed accordingly, and ultimately compiled and executed.
One of the reasons JAX can pack so much power into such a small software package
is that it starts with a familiar and flexible programming interface (Python with NumPy)
and it uses the actual Python interpreter to do most of the heavy lifting to distill the
essence of the computation into a simple statically-typed expression language
with limited higher-order features: the jaxpr language.

Not all Python programs can be processed this way, but it turns out that many
scientific computing and machine learning programs do have this property.

Before we proceed, it is important to point out that not all JAX transformations
materialize a jaxpr as described above; some, e.g., differentiation,
will apply transformations incrementally during tracing.
Nevertheless, if one wants to understand how JAX works internally, or to
make use of the result of JAX tracing, it is useful to understand jaxpr.

A jaxpr instance represents a function with one of more typed parameters (input variables)
and one or more typed results. The results depend only on the input
variables; there are no free variables captured from enclosing scopes.
The inputs and outputs have types, which in JAX are represented as abstract
values. There are two related representations in the code for jaxprs. The main
one is :py:class:`jax.core.TypedJaxpr` and is what you obtain when you
use :py:func:`jax.make_jaxpr` to inspect jaxprs. It has the following
fields:

  * ``jaxpr``: is the actual computation content of the actual function (described below).
  * ``literals`` is a list of constants. For various reasons, during tracing JAX
    will collect the non-scalar constants that arise and will replace them with
    variables, e.g., constants that appear in the Python program, or the result of
    constant folding such constants. The variables that stand for these constants
    are mentioned separately in the enclosed ``jaxpr``.
    When applying a ``TypedJaxpr`` to some actual
    arguments, one must pass first the ``literals`` followed by the actual arguments.
  * ``in_avals`` and ``out_avals`` are the types of the input variables
    (excluding the ones that correspond to the ``literals``), and of the output values.
    These types are called in JAX abstract values, e.g., ``ShapedArray(float32[10,10])``.

The most interesting part of the TypedJaxpr is the actual execution content,
represented as a :py:class:`jax.core.Jaxpr` as printed using the following
grammar::

   jaxpr ::= { lambda Var* ; Var+.
               let Eqn*
               in  [Expr+] }

where:
  * The parameter of the jaxpr are shown as two lists of variables separated by
    ``;``. The first set of variables are the ones that have been introduced
    to stand for constants that have been hoisted out. These are called the
    `constvars`. The second list of variables are the real input variables.
  * ``Eqn*`` is a list of equations, defining intermediate variables referring to
    intermediate expressions. Each equation defines one or more variables as the
    result of applying a primitive on some atomic expressions. Each equation uses only
    input variables and intermediate variables defined by previous equations.
  * ``Expr+``: is a list of output atomic expressions for the jaxpr.

Equations are printed as follows::

  Eqn  ::= let Var+ = Primitive [ Param* ] Expr+

where:
  * ``Var+”`` are one or more intermediate variables to be defined as the
    output of a primitive invocation (some primitives can return multiple values)
  * ``Expr+`` are one or more atomic expressions, each either a variable or a
    literal constant. A special form of an atomic expression is the `unit`
    expression, printed as ``*`` and standing for a value that is not needed
    in the rest of the computation and has been elided.
  * ``Param*`` are zero or more named parameters to the primitive, printed in
    square brackets. Each parameter is shown as ``Name = Value``.


Most jaxpr primitives are first-order (they take just one or more Expr as arguments)::

  Primitive := add | sub | sin | mul | ...


The jaxpr primitives are documented in the :py:mod:`jax.lax` module.

For example, here is the jaxpr produced for the function ``func1`` below::

    from jax import numpy as jnp
    def func1(first, second):
       temp = first + jnp.sin(second) * 3.
       return jnp.sum(temp)

    print(jax.make_jaxpr(func1)(jnp.zeros(8), jnp.ones(8)))
    { lambda ; a b.
      let c = sin b
          d = mul c 3.0
          e = add a d
          f = reduce_sum[ axes=(0,)
                          input_shape=(8,) ] e
      in f }

Here there are no constvars, ``a`` and ``b`` are the input variables
and they correspond respectively to
``first`` and ``second`` function parameters. The scalar literal ``3.0`` is kept
inline.
The ``reduce_sum`` primitive has named parameters ``axes`` and ``input_shape``, in
addition to the operand ``e``.

Note that JAX traces through Python-level control-flow and higher-order functions
when it extracts the jaxpr. This means that just because a Python program contains
functions and control-flow, the resulting jaxpr does not have
to contain control-flow or higher-order features.
For example, when tracing the function ``func3`` JAX will inline the call to
``inner`` and the conditional ``if second.shape[0] > 4``, and will produce the same
jaxpr as before::

    def func2(inner, first, second):
      temp = first + inner(second) * 3.
      return jnp.sum(temp)

    def inner(second):
      if second.shape[0] > 4:
        return jnp.sin(second)
      else:
        assert False

    def func3(first, second):
      return func2(inner, first, second)

    print(api.make_jaxpr(func2)(jnp.zeros(8), jnp.ones(8)))
    { lambda ; a b.
      let c = sin b
          d = mul c 3.0
          e = add a d
          f = reduce_sum[ axes=(0,)
                          input_shape=(8,) ] e
      in f }

Handling PyTrees
----------------

In jaxpr there are no tuple types; instead primitives take multiple inputs
and produce multiple outputs. When processing a function that has structured
inputs or outputs, JAX will flatten those and in jaxpr they will appear as lists
of inputs and outputs. For more details, please see the documentation for
PyTrees (:doc:`notebooks/JAX_pytrees`).

For example, the following code produces an identical jaxpr to what we saw
before (with two input vars, one for each element of the input tuple)::


    def func4(arg):  # Arg is a pair
      temp = arg[0] + jnp.sin(arg[1]) * 3.
      return jnp.sum(temp)

    print(api.make_jaxpr(func4)((jnp.zeros(8), jnp.ones(8))))
    { lambda a b.
      let c = sin b
          d = mul c 3.0
          e = add a d
          f = reduce_sum[ axes=(0,)
                          input_shape=(8,) ] e
      in f }



Constant Vars
--------------

ConstVars arise when the computation ontains array constants, either
from the Python program, or from constant-folding. For example, the function
``func6`` below::

    def func5(first, second):
      temp = first + jnp.sin(second) * 3. - jnp.ones(8)
      return temp

    def func6(first):
      return func5(first, jnp.ones(8))

    print(api.make_jaxpr(func6)(jnp.ones(8)))


JAX produces the following jaxpr::

    { lambda b d a.
      let c = add a b
          e = sub c d
      in e }

When tracing ``func6``, the function ``func5`` is invoked with a constant value
(``onp.ones(8)``) for the second argument. As a result, the sub-expression
``jnp.sin(second) * 3.`` is constant-folded.
There are two ConstVars, ``b`` (standing for ``jnp.sin(second) * 3.``) and ``d``
(standing for ``jnp.ones(8)``). Unfortunately, it is not easy to tell from the
jaxpr notation what constants the constant variables stand for.

Higher-order primitives
-----------------------

jaxpr includes several higher-order primitives. They are more complicated because
they include sub-jaxprs.

Cond
^^^^

JAX traces through normal Python conditionals. To capture a conditional expression
for dynamic execution, one must use the :py:func:`jax.lax.cond` constructor
with the following signature::

  lax.cond(pred : bool, true_op: A, true_body: A -> B, false_op: C, false_body: C -> B) -> B

For example::


    def func7(arg):
      return lax.cond(arg >= 0.,
                      arg,
                      lambda xtrue: xtrue + 3.,
                      arg,
                      lambda xfalse: xfalse - 3.)

    print(api.make_jaxpr(func7)(5.))
    { lambda  ; a.
      let b = ge a 0.0
          c = cond[ false_jaxpr={ lambda  ; a.
                                  let b = sub a 3.0
                                  in b }
                    linear=(False, False)
                    true_jaxpr={ lambda  ; a.
                                 let b = add a 3.0
                                 in b } ] b a a
      in c }


The cond primitive has a number of parameters:

  * `true_jaxpr` and `false_jaxpr` are jaxprs that correspond to the true
    and false branch functionals. In this example, those functionals take each
    one input variable, corresponding to ``xtrue`` and ``xfalse`` respectively.
  * `linear` is a tuple of booleans that is used internally by the auto-differentiation
    machinery to encode which of the input parameters are used linearly in the
    conditional.

The above instance of the cond primitive takes 3 operands.
The first one (``b``) is the predicate, then ``a` is the ``true_op`` (``arg``, to be
passed to ``true_jaxpr``) and also ``a`` is the ``false_op``
(``arg``, to be passed to ``false_jaxpr``).

The following example shows a more complicated situation when the input
to the branch functionals is a tuple, and the `false` branch functional
contains a constant ``jnp.ones(1)`` that is hoisted as a `constvar`::

    def func8(arg1, arg2):  # arg2 is a pair
      return lax.cond(arg1 >= 0.,
                      arg2,
                      lambda xtrue: xtrue[0],
                      arg2,
                      lambda xfalse: jnp.ones(1) + xfalse[1])

    print(api.make_jaxpr(func8)(5., (jnp.zeros(1), 2.)))
    { lambda e ; a b c.
      let d = ge a 0.0
          f = cond[ false_jaxpr={ lambda  ; c a b.
                                  let d = add c b
                                  in d }
                    linear=(False, False, False, False, False)
                    true_jaxpr={ lambda  ; a b.
                                 let
                                 in a } ] d b c e b c
      in f }

The top-level jaxpr has one `constvar` ``e`` (corresponding to ``jnp.ones(1)`` from the
body of the ``false_jaxpr``) and three input variables ``a b c`` (corresponding to ``arg1``
and the two elements of ``arg2``; note that ``arg2`` has been flattened).
The ``true_jaxpr`` has two input variables (corresponding to the two elements of ``arg2``
that is passed to ``true_jaxpr``).
The ``false_jaxpr`` has three input variables (``c`` corresponding to the constant for
``jnp.ones(1)``, and ``a b`` for the two elements of ``arg2`` that are passed
to ``false_jaxpr``).

The actual operands to the cond primitive are: ``d b c e b c``, which correspond in order to:

  * 1 operand for the predicate,
  * 2 operands for ``true_jaxpr``, i.e., ``b`` and ``c``, which are input vars,
    corresponding to ``arg2`` for the top-level jaxpr,
  * 1 constant for ``false_jaxpr``, i.e., ``e``, which is a consvar for the top-level jaxpr,
  * 2 operands for ``true_jaxpr``, i.e., ``b`` and ``c``, which are the input vars
    corresponding to ``arg2`` for the top-level jaxpr.

While
^^^^^

Just like for conditionals, Python loops are inlined during tracing.
If you want to capture a loop for dynamic execution, you must use one of several
special operations, :py:func:`jax.lax.while_loop` (a primitive)
and :py:func:`jax.lax.fori_loop`
(a helper that generates a while_loop primitive)::

    lax.while_loop(cond_fun: (C -> bool), body_fun: (C -> C), init: C) -> C
    lax.fori_loop(start: int, end: int, body: (int -> C -> C), init: C) -> C


In the above signature, “C” stands for the type of a the loop “carry” value.
For example, here is an example fori loop::

    def func10(arg, n):
      ones = jnp.ones(arg.shape)  # A constant
      return lax.fori_loop(0, n,
                           lambda i, carry: carry + ones * 3. + arg,
                           arg + ones)

    print(api.make_jaxpr(func10)(onp.ones(16), 5))
    { lambda c d ; a b.
      let e = add a d
          f g h = while[ body_jaxpr={ lambda  ; e g a b c.
                                      let d = add a 1
                                          f = add c e
                                          h = add f g
                                      in (d, b, h) }
                         body_nconsts=2
                         cond_jaxpr={ lambda  ; a b c.
                                      let d = lt a b
                                      in d }
                         cond_nconsts=0 ] c a 0 b e
      in h }

The top-level jaxpr has two constvars: ``c`` (corresponding to ``ones * 3.`` from the body
of the loop) and ``d`` (corresponding to the use of ``ones`` in the initial carry).
There are also two input variables (``a`` corresponding to ``arg`` and ``b`` corresponding
to ``n``).
The loop carry consists of three values, as seen in the body of ``cond_jaxpr``
(corresponding to the iteration index, iteration end, and the accumulated value carry).
Note that ``body_jaxpr`` takes 5 input variables. The first two are actually
constvars: ``e`` corresponding to ``ones * 3`` and ``g`` corresponding to the
captures use of ``arg`` in the loop body.
The parameter ``body_nconsts = 2`` specifies that there are 2 constants for the
``body_jaxpr``.
The other 3 input variables for ``body_jaxpr`` correspond to the flattened carry values.

The while primitive takes 5 arguments: ``c a 0 b e``, as follows:

  * 0 constants for ``cond_jaxpr`` (since ``cond_nconsts`` is 0)
  * 2 constants for ``body_jaxpr`` (``c``, and ``a``)
  * 3 parameters for the initial value of carry

Scan
^^^^

JAX supports a special form of loop over the elements of an array (with
statically known shape). The fact that there are a fixed number of iterations
makes this form of looping easily reverse-differentiable. Such loops are constructed
with the :py:func:`jax.lax.scan` operator::

  lax.scan(body_fun: (C -> A -> (C, B)), init_carry: C, in_arr: Array[A]) -> (C, Array[B])

Here ``C`` is the type of the scan carry, ``A`` is the element type of the input array(s),
and ``B`` is the element type of the output array(s).

For the example consider the function ``func11`` below::

    def func11(arr, extra):
      ones = jnp.ones(arr.shape)  #  A constant
      def body(carry, aelems):
        # carry: running dot-product of the two arrays
        # aelems: a pair with corresponding elements from the two arrays
        ae1, ae2 = aelems
        return (carry + ae1 * ae2 + extra, carry)

      return lax.scan(body, 0., (arr, ones))

     print(api.make_jaxpr(func11)(onp.ones(16), 5.))
    { lambda c ; a b.
      let d e = scan[ forward=True
                      jaxpr={ lambda  ; a b c d e.
                              let f = mul c e
                                  g = add b f
                                  h = add g a
                              in (h, b) }
                      length=16
                      linear=(False, False, False, True, False)
                      num_carry=1
                      num_consts=1 ] b 0.0 a * c
      in (d, e) }

The top-level jaxpr has one constvar ``c`` corresponding to the ``ones`` constant,
and two input variables corresponding to the arguments ``arr`` and ``extra``.
The body of the scan has 5 input variables, of which:

  * one (``a``) is a constant (since ``num_consts = 1``), and stands for the
    captured variable ``extra`` used in the loop body,
  * one (``b``) is the value of the carry (since ``num_carry = 1``)
  * The remaining 3 are the input values. Notice that only ``c`` and ``e`` are used,
    and stand respectively for the array element from the first array passed to
    lax.scan (``arr``) and to the second array (``ones``). The input variables
    (``d``) seems to be an artifact of the translation.

The ``linear`` parameter describes for each of the input variables whether they
are guaranteed to be used linearly in the body. Here, only the unused input
variable is marked linear. Once the scan goes through linearization, more arguments
will be linear.

The scan primitive takes 5 arguments: ``b 0.0 a * c``, of which:

  * one is the free variable for the body
  * one is the initial value of the carry
  * The next 3 are the arrays over which the scan operates. The middle one is not used (*).

XLA_call
^^^^^^^^

The call primitive arises from JIT compilation, and it encapsulates
a sub-jaxpr along with parameters the specify the backend and the device the
computation should run. For example::

    def func12(arg):
      @api.jit
      def inner(x):
        return x + arg * jnp.ones(1)  # Include a constant in the inner function
      return arg + inner(arg - 2.)

    print(api.make_jaxpr(func12)(1.))
    { lambda b ; a.
      let c = sub a 2.0
          d = xla_call[ backend=None
                        call_jaxpr={ lambda  ; c b a.
                                     let d = mul b c
                                         e = add a d
                                     in e }
                        device=None
                        name=inner ] b a c
          e = add a d
      in e }

The top-level constvar ``b`` refers to the ``jnp.ones(1)`` constant, and
the top-level input variable `a` refers to the ``arg`` parameter of ``func12``.
The ``xla_call`` primitive stands for a call to the jitted ``inner`` function.
The primitive has the function body in the ``call_jaxpr`` parameter, a jaxpr
with 3 input parameters:

  * ``c`` is a constvar and stands for the ``ones`` constant,
  * ``b`` corresponds to the free variable ``arg`` captured in the ``inner`` function,
  * ``a`` corresponds to the ``inner`` parameter ``x``.

The primitive takes three arguments ``b a c``.

XLA_pmap
^^^^^^^^

If you use the :py:func:`jax.pmap` transformation, the function to be
mapped is captured using the ``xla_pmap`` primitive. Consider this
example::

    def func13(arr, extra):
      def inner(x):
        # use a free variable "extra" and a constant jnp.ones(1)
        return (x + extra + jnp.ones(1)) / lax.psum(x, axis_name='rows')
      return api.pmap(inner, axis_name='rows')(arr)

    print(api.make_jaxpr(func13)(jnp.ones((1, 3)), 5.))
    { lambda c ; a b.
      let d = xla_pmap[ axis_name=rows
                        axis_size=1
                        backend=None
                        call_jaxpr={ lambda  ; d b a.
                                     let c = add a b
                                         e = add c d
                                         f = psum[ axis_name=rows ] a
                                         g = div e f
                                     in g }
                        devices=None
                        global_axis_size=None
                        mapped_invars=(True, False, True)
                        name=inner ] c b a
      in d }

The top-level constvar ``c`` refers to the ``jnp.ones(1)`` constant.
The ``xla_pmap`` primitive specifies the name of the axis (parameter ``rows``)
and the body of the function to be mapped as the ``call_jaxpr`` parameter. The
value of this parameter is a Jaxpr with 3 input variables:

  * ``d`` stands for the constant ``jnp.ones(1)``,
  * ``b`` stands for the free variable ``extra``,
  * ``a`` stands for the parameter ``x`` of ``inner``.


The parameter ``mapped_invars`` specify which of the input variables should be
mapped and which should be broadcast. In our example, the value of ``extra``
is broadcast, the other input values are mapped.

