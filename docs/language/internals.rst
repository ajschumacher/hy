=========================
Internal Hy Documentation
=========================

.. note:: These bits are mostly useful for folks who hack on Hy itself,
    but can also be used for those delving deeper in macro programming.

.. _models:

Hy Models
=========

Introduction to Hy Models
-------------------------

Hy models are a very thin layer on top of regular Python objects,
representing Hy source code as data. Models only add source position
information, and a handful of methods to support clean manipulation of
Hy source code, for instance in macros. To achieve that goal, Hy models
are mixins of a base Python class and :ref:`HyObject`.

.. _hyobject:

HyObject
~~~~~~~~

``hy.models.HyObject`` is the base class of Hy models. It only
implements one method, ``replace``, which replaces the source position
of the current object with the one passed as argument. This allows us to
keep track of the original position of expressions that get modified by
macros, be that in the compiler or in pure hy macros.

``HyObject`` is not intended to be used directly to instantiate Hy
models, but only as a mixin for other classes.

Compound Models
---------------

Parenthesized and bracketed lists are parsed as compound models by the
Hy parser.

Hy uses pretty-printing reprs for its compound models by default.
If this is causing issues,
it can be turned off globally by setting ``hy.models.PRETTY`` to ``False``,
or temporarily by using the ``hy.models.pretty`` context manager.

Hy also attempts to color pretty reprs using ``clint.textui.colored``.
This module has a flag to disable coloring,
and a method ``clean`` to strip colored strings of their color tags.

.. _hysequence:

HySequence
~~~~~~~~~~

``hy.models.HySequence`` is the abstract base class of "iterable" Hy
models, such as HyExpression and HyList.

Adding a HySequence to another iterable object reuses the class of the
left-hand-side object, a useful behavior when you want to concatenate Hy
objects in a macro, for instance.


.. _hylist:

HyList
~~~~~~

``hy.models.HyList`` is a :ref:`HySequence` for bracketed ``[]``
lists, which, when used as a top-level expression, translate to Python
list literals in the compilation phase.


.. _hyexpression:

HyExpression
~~~~~~~~~~~~

``hy.models.HyExpression`` inherits :ref:`HySequence` for
parenthesized ``()`` expressions. The compilation result of those
expressions depends on the first element of the list: the compiler
dispatches expressions between compiler special-forms, user-defined
macros, and regular Python function calls.

.. _hydict:

HyDict
~~~~~~

``hy.models.HyDict`` inherits :ref:`HySequence` for curly-bracketed
``{}`` expressions, which compile down to a Python dictionary literal.

The decision of using a list instead of a dict as the base class for
``HyDict`` allows easier manipulation of dicts in macros, with the added
benefit of allowing compound expressions as dict keys (as, for instance,
the :ref:`HyExpression` Python class isn't hashable).

Atomic Models
-------------

In the input stream, double-quoted strings, respecting the Python
notation for strings, are parsed as a single token, which is directly
parsed as a :ref:`HyString`.

An uninterrupted string of characters, excluding spaces, brackets,
quotes, double-quotes and comments, is parsed as an identifier.

Identifiers are resolved to atomic models during the parsing phase in
the following order:

 - :ref:`HyInteger <hy_numeric_models>`
 - :ref:`HyFloat <hy_numeric_models>`
 - :ref:`HyComplex <hy_numeric_models>` (if the atom isn't a bare ``j``)
 - :ref:`HyKeyword` (if the atom starts with ``:``)
 - :ref:`HySymbol`

.. _hystring:

HyString
~~~~~~~~

``hy.models.HyString`` represents string literals (including bracket strings),
which compile down to unicode string literals (``str``) in Python.

``HyString``\s are immutable.

Hy literal strings can span multiple lines, and are considered by the
parser as a single unit, respecting the Python escapes for unicode
strings.

``HyString``\s have an attribute ``brackets`` that stores the custom
delimiter used for a bracket string (e.g., ``"=="`` for ``#[==[hello
world]==]`` and the empty string for ``#[[hello world]]``).
``HyString``\s that are not produced by bracket strings have their
``brackets`` set to ``None``.

HyBytes
~~~~~~~

``hy.models.HyBytes`` is like ``HyString``, but for sequences of bytes.
It inherits from ``bytes``.

.. _hy_numeric_models:

Numeric Models
~~~~~~~~~~~~~~

``hy.models.HyInteger`` represents integer literals, using the ``int``
type.

``hy.models.HyFloat`` represents floating-point literals.

``hy.models.HyComplex`` represents complex literals.

Numeric models are parsed using the corresponding Python routine, and
valid numeric python literals will be turned into their Hy counterpart.

.. _hysymbol:

HySymbol
~~~~~~~~

``hy.models.HySymbol`` is the model used to represent symbols in the Hy
language. Like ``HyString``, it inherits from ``str`` (or ``unicode`` on Python
2).

Symbols are :ref:`mangled <mangling>` when they are compiled
to Python variable names.

.. _hykeyword:

HyKeyword
~~~~~~~~~

``hy.models.HyKeyword`` represents keywords in Hy. Keywords are
symbols starting with a ``:``. See :ref:`syntax-keywords`.

Hy Internal Theory
==================

.. _overview:

Overview
--------

The Hy internals work by acting as a front-end to Python bytecode, so
that Hy itself compiles down to Python Bytecode, allowing an unmodified
Python runtime to run Hy code, without even noticing it.

The way we do this is by translating Hy into an internal Python AST
datastructure, and building that AST down into Python bytecode using
modules from the Python standard library, so that we don't have to
duplicate all the work of the Python internals for every single Python
release.

Hy works in four stages. The following sections will cover each step of Hy
from source to runtime.

.. _lexing:

Steps 1 and 2: Tokenizing and Parsing
-------------------------------------

The first stage of compiling Hy is to lex the source into tokens that we can
deal with. We use a project called rply, which is a really nice (and fast)
parser, written in a subset of Python called rpython.

The lexing code is all defined in ``hy.lex.lexer``. This code is mostly just
defining the Hy grammar, and all the actual hard parts are taken care of by
rply -- we just define "callbacks" for rply in ``hy.lex.parser``, which takes
the tokens generated, and returns the Hy models.

You can think of the Hy models as the "AST" for Hy, it's what Macros operate
on (directly), and it's what the compiler uses when it compiles Hy down.

.. seealso::

   Section :ref:`models` for more information on Hy models and what they mean.

.. _compiling:

Step 3: Hy Compilation to Python AST
------------------------------------

This is where most of the magic in Hy happens. This is where we take Hy AST
(the models), and compile them into Python AST. A couple of funky things happen
here to work past a few problems in AST, and working in the compiler is some
of the most important work we do have.

The compiler is a bit complex, so don't feel bad if you don't grok it on the
first shot, it may take a bit of time to get right.

The main entry-point to the Compiler is ``HyASTCompiler.compile``. This method
is invoked, and the only real "public" method on the class (that is to say,
we don't really promise the API beyond that method).

In fact, even internally, we don't recurse directly hardly ever, we almost
always force the Hy tree through ``compile``, and will often do this with
sub-elements of an expression that we have. It's up to the Type-based dispatcher
to properly dispatch sub-elements.

All methods that preform a compilation are marked with the ``@builds()``
decorator. You can either pass the class of the Hy model that it compiles,
or you can use a string for expressions. I'll clear this up in a second.

First Stage Type-Dispatch
~~~~~~~~~~~~~~~~~~~~~~~~~

Let's start in the ``compile`` method. The first thing we do is check the
Type of the thing we're building. We look up to see if we have a method that
can build the ``type()`` that we have, and dispatch to the method that can
handle it. If we don't have any methods that can build that type, we raise
an internal ``Exception``.

For instance, if we have a ``HyString``, we have an almost 1-to-1 mapping of
Hy AST to Python AST. The ``compile_string`` method takes the ``HyString``, and
returns an ``ast.Str()`` that's populated with the correct line-numbers and
content.

Macro-Expand
~~~~~~~~~~~~

If we get a ``HyExpression``, we'll attempt to see if this is a known
Macro, and push to have it expanded by invoking ``hy.macros.macroexpand``, then
push the result back into ``HyASTCompiler.compile``.

Second Stage Expression-Dispatch
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The only special case is the ``HyExpression``, since we need to create different
AST depending on the special form in question. For instance, when we hit an
``(if True True False)``, we need to generate a ``ast.If``, and properly
compile the sub-nodes. This is where the ``@builds()`` with a String as an
argument comes in.

For the ``compile_expression`` (which is defined with an
``@builds(HyExpression)``) will dispatch based on the string of the first
argument. If, for some reason, the first argument is not a string, it will
properly handle that case as well (most likely by raising an ``Exception``).

If the String isn't known to Hy, it will default to create an ``ast.Call``,
which will try to do a runtime call (in Python, something like ``foo()``).

Issues Hit with Python AST
~~~~~~~~~~~~~~~~~~~~~~~~~~

Python AST is great; it's what's enabled us to write such a powerful project
on top of Python without having to fight Python too hard. Like anything, we've
had our fair share of issues, and here's a short list of the common ones you
might run into.

*Python differentiates between Statements and Expressions*.

This might not sound like a big deal -- in fact, to most Python programmers,
this will shortly become a "Well, yeah" moment.

In Python, doing something like:

``print for x in range(10): pass``, because ``print`` prints expressions, and
``for`` isn't an expression, it's a control flow statement. Things like
``1 + 1`` are Expressions, as is ``lambda x: 1 + x``, but other language
features, such as ``if``, ``for``, or ``while`` are statements.

Since they have no "value" to Python, this makes working in Hy hard, since
doing something like ``(print (if True True False))`` is not just common, it's
expected.

As a result, we reconfigure things using a ``Result`` object, where we offer
up any ``ast.stmt`` that need to get run, and a single ``ast.expr`` that can
be used to get the value of whatever was just run. Hy does this by forcing
assignment to things while running.

As example, the Hy::

    (print (if True True False))

Will turn into::

    if True:
        _temp_name_here = True
    else:
        _temp_name_here = False

    print(_temp_name_here)


OK, that was a bit of a lie, since we actually turn that statement
into::

    print(True if True else False)

By forcing things into an ``ast.expr`` if we can, but the general idea holds.


Step 4: Python Bytecode Output and Runtime
------------------------------------------

After we have a Python AST tree that's complete, we can try and compile it to
Python bytecode by pushing it through ``eval``. From here on out, we're no
longer in control, and Python is taking care of everything. This is why things
like Python tracebacks, pdb and django apps work.


Hy Macros
=========

.. _using-gensym:

Using gensym for Safer Macros
-----------------------------

When writing macros, one must be careful to avoid capturing external variables
or using variable names that might conflict with user code.

We will use an example macro ``nif`` (see http://letoverlambda.com/index.cl/guest/chap3.html#sec_5
for a more complete description.) ``nif`` is an example, something like a numeric ``if``,
where based on the expression, one of the 3 forms is called depending on if the
expression is positive, zero or negative.

A first pass might be something like:

.. code-block:: hy

   (defmacro nif [expr pos-form zero-form neg-form]
     `(do
       (setv obscure-name ~expr)
       (cond [(pos? obscure-name) ~pos-form]
             [(zero? obscure-name) ~zero-form]
             [(neg? obscure-name) ~neg-form])))

where ``obscure-name`` is an attempt to pick some variable name as not to
conflict with other code. But of course, while well-intentioned,
this is no guarantee.

The method :ref:`gensym` is designed to generate a new, unique symbol for just
such an occasion. A much better version of ``nif`` would be:

.. code-block:: hy

   (defmacro nif [expr pos-form zero-form neg-form]
     (setv g (gensym))
     `(do
        (setv ~g ~expr)
        (cond [(pos? ~g) ~pos-form]
              [(zero? ~g) ~zero-form]
              [(neg? ~g) ~neg-form])))

This is an easy case, since there is only one symbol. But if there is
a need for several gensym's there is a second macro :ref:`with-gensyms` that
basically expands to a ``setv`` form:

.. code-block:: hy

   (with-gensyms [a b c]
     ...)

expands to:

.. code-block:: hy

   (do
     (setv a (gensym)
           b (gensym)
           c (gensym))
     ...)

so our re-written ``nif`` would look like:

.. code-block:: hy

   (defmacro nif [expr pos-form zero-form neg-form]
     (with-gensyms [g]
       `(do
          (setv ~g ~expr)
          (cond [(pos? ~g) ~pos-form]
                [(zero? ~g) ~zero-form]
                [(neg? ~g) ~neg-form]))))

Finally, though we can make a new macro that does all this for us. :ref:`defmacro/g!`
will take all symbols that begin with ``g!`` and automatically call ``gensym`` with the
remainder of the symbol. So ``g!a`` would become ``(gensym "a")``.

Our final version of ``nif``, built with ``defmacro/g!`` becomes:

.. code-block:: hy

   (defmacro/g! nif [expr pos-form zero-form neg-form]
     `(do
        (setv ~g!res ~expr)
        (cond [(pos? ~g!res) ~pos-form]
              [(zero? ~g!res) ~zero-form]
              [(neg? ~g!res) ~neg-form])))



Checking Macro Arguments and Raising Exceptions
-----------------------------------------------



Hy Compiler Built-Ins
=====================

.. TODO: Write this.
