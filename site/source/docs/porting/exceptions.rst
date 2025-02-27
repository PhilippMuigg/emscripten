.. _exceptions:

======================
C++ Exceptions Support
======================

By default, exception catching is disabled in Emscripten. For example, if you
compile the following program:

.. code-block:: cpp

    #include <stdio.h>

    int main() {
      try {
        puts("throw...");
        throw 1;
        puts("(never reached)");
      } catch(...) {
        puts("catch!");
      }
      return 0;
    }

The first ``throw`` will abort the program and you'll see something like this in
the output:

.. code-block:: text

  throw...
  exception thrown: 5246024 - Exception catching is disabled, this exception cannot be caught. Compile with -sNO_DISABLE_EXCEPTION_CATCHING or -sEXCEPTION_CATCHING_ALLOWED=[..] to catch.

If you want to opt-in, you have two following options.


.. _javascript-based-exception-support:

JavaScript-based Exception Support
==================================

First, you can enable exceptions via Emscripten's JavaScript-based support. To
enable it, pass ``-fexceptions`` at both compile time and link time.

When you rebuild the example above with this flag, the output will change to:

.. code-block:: text

  throw...
  catch!

Note that this option has relatively high overhead, but it will work on all
JavaScript engines with WebAssembly support. You can reduce the overhead by
specifying a list of allowed functions in which exceptions are enabled, see the
``EXCEPTION_CATCHING_ALLOWED`` setting.


.. _webassembly-exception-handling-based-support:

WebAssembly Exception Handling-based Support
============================================

Alternatively, you can opt-in to the `native WebAssembly exception handling
<https://github.com/WebAssembly/exception-handling/blob/master/proposals/exception-handling/Exceptions.md>`_
proposal. To enable it, pass ``-fwasm-exceptions`` at both compile time and link
time.

Rebuilding the example with this flag will result in the same output as with
``-fexceptions`` above:

.. code-block:: text

  throw...
  catch!

This option leverages a new feature that brings built-in instructions for
throwing and catching exceptions to WebAssembly. As a result, it can reduce code
size and performance overhead compared to the JavaScript-based implementation.
This option is currently supported in several major web browsers, but `may not
be supported in all WebAssembly engines yet
<https://webassembly.org/roadmap/>`_.


Debugging Exceptions
====================

Stack Traces
------------

For :ref:`native Wasm exceptions
<webassembly-exception-handling-based-support>`, when :ref:`ASSERTIONS
<debugging-ASSERTIONS>` is enabled, uncaught exceptions will print stack traces
for debugging. :ref:`ASSERTIONS <debugging-ASSERTIONS>` is enabled by default
in :ref:`-O0 <emcc-O0>` and disabled in optimized builds (:ref:`-O1 <emcc-O1>`
and above). You can enable it by passing ``-sASSERTIONS`` to the ``emcc``
command line in optimized builds as well. To display Wasm function names in
stack traces, you also need :ref:`--profiling-funcs <emcc-profiling-funcs>`
(or :ref:`-g <emcc-g>` or :ref:`-gsource-map<emcc-gsource-map>`).

In JavaScript, you can also examine the stack traces using
`WebAssembly.Exception.prototype.stack
<https://developer.mozilla.org/en-US/docs/WebAssembly/JavaScript_interface/Exception/stack>`_
property. For example:

.. code-block:: javascript

  try {
    ... // some code that calls WebAssembly
  } catch (e) {
    // Do something with e.stack
    console.log(e.stack);
  }

Stack traces within Wasm code are not supported in :ref:`JavaScipt-based
exceptions <javascript-based-exception-support>`.

Exception Messages
------------------

You can also catch and examine the type and the message of C++ exceptions from
JavaScript. See :ref:`handling-c-exceptions-from-javascript`.


Using Exceptions and setjmp-longjmp Together
============================================

See :ref:`using-exceptions-and-setjmp-longjmp-together`.
