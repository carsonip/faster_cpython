*************
Optimizations
*************

Inline function calls
=====================

Example::

    def _get_sep(path):
        if isinstance(path, bytes):
            return b'/'
        else:
            return '/'

    def isabs(s):
        """Test whether a path is absolute"""
        sep = _get_sep(s)
        return s.startswith(sep)

Inline ``_get_sep()`` into ``isabs()`` and simplify the code for the ``str``
type::

    def isabs(s: str):
        return s.startswith('/')

It can be implemented as a simple call to the C function
``PyUnicode_Tailmatch()``.

Note: Inlining uses more memory and disk because the original function should
be kept. Except if the inlined function is unreachable (ex: "private
function"?).

Links:

* `Issue #10399 <http://bugs.python.org/issue10399>`_:
  AST Optimization: inlining of function calls


Move invariants out of the loop
===============================

Example::

    def func(obj, lines):
        for text in lines:
            print(obj.cleanup(text))

Become::

    def func(obj, lines):
        local_print = print
        obj_cleanup = obj.cleanup
        for text in lines:
            local_print(obj_cleanup(text))

Local variables are faster than global variables and the attribute lookup is
only done once.


C functions using only C types
==============================

Optimizations:

* Avoid reference counting
* Memory allocations on the heap
* Release the GIL

Example::

    def demo():
        s = 0
        for i in range(10):
            s += i
        return s

In specialized code, it may be possible to use basic C types like ``char`` or
``int`` instead of Python codes which can be allocated on the stack, instead of
allocating objects on the heap. ``i`` and ``s`` variables are integers in the
range ``[0; 45]`` and so a simple C type ``int`` (or even ``char``) can be
used::

    PyObject *demo(void)
    {
        int s, i;
        Py_BEGIN_ALLOW_THREADS
        s = 0;
        for(i=0; i<10; i++)
            s += i;
        Py_END_ALLOW_THREADS
        return PyLong_FromLong(s);
    }

Note: if the function is slow, we may need to check sometimes if a signal was
received.


Release the GIL
===============

Many methods of builtin types don't need the GIL. Example:
``"abc".startswith("def")``.


Replace calls to pure functions with the result
===============================================

Examples:

- ``len('abc')`` becomes ``3``
- ``"python2.7".startswith("python")`` becomes ``True``
- ``math.log(32) / math.log(2)`` becomes ``5.0``

Can be implemented in the AST optimizer.


Constant folding
================

Replace constants by their values. Simple example from pickle.py::

        MARK = b'('
        TUPLE = b't'

        def func():
            ...
            self.write(MARK + TUPLE)

The function becomes::

        def func():
            ...
            self.write(b'(t')

Can be implemented in an :ref:`AST optimizer <ast-optimizers>`.

See also `issue #1346238 <http://bugs.python.org/issue1346238>`_: A constant
folding optimization pass for the AST.


Peephole optimizer
==================

See :ref:`CPython peephole optimizer <cpython-peephole>`.


Unroll loops
============

Example::

    for i in range(4):
        print(i)

The loop body can be duplicated (twice in this example) to reduce the cost of a
loop::

    for i in range(0,4,2):
        print(i)
        print(i+1)
    i = 3

Or::

    print(0)
    print(1)
    print(2)
    print(3)
    i = 3



Remove dead code
================

- ``if DEBUG: print("debug")`` where ``DEBUG`` is known to be False


Load globals when the module is loaded
======================================

Load globals when the module is loaded? Ex: load "print" name when the module
is loaded.

Example::

    def hello():
        print("Hello World")

Become::

    local_print = print

    def hello():
        local_print("Hello World")

Useful if ``hello()`` is compiled to C code.


Don't create Python frames
==========================

Inlining and other optimizations don't create Python frames anymore. It can be
a serious issue to debug programs: tracebacks are an important feature of
Python.

At least in debug mode, frames should be created.

PyPy supports lazy creation of frames if an exception is raised.


