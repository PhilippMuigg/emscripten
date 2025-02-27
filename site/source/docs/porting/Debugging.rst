.. _Debugging:

=========
Debugging
=========

One of the main advantages of debugging cross-platform Emscripten code is that the same cross-platform source code can be debugged on either the native platform or using the web browser's increasingly powerful toolset — including debugger, profiler, and other tools.

Emscripten provides a lot of functionality and tools to aid debugging:

- :ref:`Compiler debug information flags <debugging-debug-information-g>` that allow you to preserve debug information in compiled code and even create source maps so that you can step through the native C++ source when debugging in the browser.
- :ref:`Debug mode <debugging-EMCC_DEBUG>`, which emits debug logs and stores intermediate build files for analysis.
- :ref:`Compiler settings <debugging-compilation-settings>` to enable runtime checking of memory accesses and common allocation errors.
- :ref:`debugging-manual-debugging` of Emscripten-generated code is also supported, which is in some ways even better than on native platforms.
- :ref:`debugging-autodebugger`, which automatically instruments LLVM bitcode to write out each store to memory.

This article describes the main tools and settings provided by Emscripten for debugging, along with a section explaining how to debug a number of :ref:`debugging-emscripten-specific-issues`.


.. _debugging-debug-information-g:

Debug information
=================

:ref:`Emcc <emccdoc>` strips out most of the debug information from :ref:`optimized builds <Optimizing-Code>` by default. Optimisation levels :ref:`-O1 <emcc-O1>` and above remove LLVM debug information, and also disable runtime :ref:`ASSERTIONS <debugging-ASSERTIONS>` checks. From optimization level :ref:`-O2 <emcc-O2>` the code is minified by the :term:`Closure Compiler` and becomes virtually unreadable.

The *emcc* :ref:`-g flag <emcc-g>` can be used to preserve debug information in the compiled output. By default, this option includes Clang / LLVM debug information in a DWARF format in the generated WebAssembly code and preserves white-space, function names and variable names in the generated JavaScript code.

The flag can also be specified with an integer level: :ref:`-g0 <emcc-g0>`, :ref:`-g1 <emcc-g1>`, :ref:`-g2 <emcc-g2>`, and :ref:`-g3 <emcc-g3>` (default level when setting ``-g``). Each level builds on the last to provide progressively more debug information in the compiled output. See :ref:`Compiler debug information flags <emcc-gN>` for more details.

The :ref:`-gsource-map <emcc-gsource-map>` option is similar to ``-g2`` but also generates source maps that allow you to view and debug the *C/C++ source code* in your browser's debugger. Source maps are not as powerful as DWARF which was mentioned earlier (they contain only source location info), but they are currently more widely supported.

.. note:: Some optimizations may be disabled when used in conjunction with the debug flags. For example, if you compile with ``-O3 -g`` some of the normal ``-O3`` optimizations will be disabled in order to provide the requested debugging information, such as name minification.

.. _debugging-EMCC_DEBUG:

Debug mode (EMCC_DEBUG)
=======================

The ``EMCC_DEBUG`` environment variable can be set to enable Emscripten's debug mode:

.. code-block:: bash

  # Linux or macOS
  EMCC_DEBUG=1 emcc test/hello_world.cpp -o hello.html

  # Windows
  set EMCC_DEBUG=1
  emcc test/hello_world.cpp -o hello.html
  set EMCC_DEBUG=0

With ``EMCC_DEBUG=1`` set, :ref:`emcc <emccdoc>` emits debug output and generates intermediate files for the compiler's various stages. ``EMCC_DEBUG=2`` additionally generates intermediate files for each JavaScript optimizer pass.

The debug logs and intermediate files are output to
**TEMP_DIR/emscripten_temp**, where ``TEMP_DIR`` is the OS default temporary
directory (e.g. **/tmp** on UNIX).

The debug logs can be analysed to profile and review the changes that were made in each step.

.. note:: The more limited amount of debug information can also be enabled by specifying the :ref:`verbose output <debugging-emcc-v>` compiler flag (``emcc -v``).


.. _debugging-compilation-settings:

Compiler settings
==================

Emscripten has a number of compiler settings that can be useful for debugging. These are set using the :ref:`emcc -s<emcc-s-option-value>` option, and will override any optimization flags. For example:

.. code-block:: bash

  emcc -O1 -sASSERTIONS test/hello_world

Some important settings are:

  -
    .. _debugging-ASSERTIONS:

    ``ASSERTIONS=1`` is used to enable runtime checks for common memory allocation errors (e.g. writing more memory than was allocated). It also defines how Emscripten should handle errors in program flow. The value can be set to ``ASSERTIONS=2`` in order to run additional tests.

    ``ASSERTIONS=1`` is enabled by default. Assertions are turned off for optimized code (:ref:`-O1 <emcc-O1>` and above).

  -
    .. _debugging-SAFE-HEAP:

    ``SAFE_HEAP=1`` adds additional memory access checks, and will give clear errors for problems like dereferencing 0 and memory alignment issues.

    You can also set ``SAFE_HEAP_LOG`` to log ``SAFE_HEAP`` operations.

  -
    .. _debugging-STACK_OVERFLOW_CHECK:

    Passing the ``STACK_OVERFLOW_CHECK=1`` linker flag adds a runtime magic
    token value at the end of the stack, which is checked in certain locations
    to verify that the user code does not accidentally write past the end of the
    stack. While overrunning the Emscripten stack is not a security issue for
    JavaScript (which is unaffected), writing past the stack causes memory
    corruption in global data and dynamically allocated memory sections in the
    Emscripten HEAP, which makes the application fail in unexpected ways. The
    value ``STACK_OVERFLOW_CHECK=2`` enables slightly more detailed stack guard
    checks, which can give a more precise callstack at the expense of some
    performance. Default value is 1 if ``ASSERTIONS=1`` is set, and disabled
    otherwise.

  -
    .. _debugging-DEMANGLE_SUPPORT:

    ``DEMANGLE_SUPPORT=1`` links in code to automatically demangle stack traces, that is, emit human-readable C++ function names instead of ``_ZN..`` ones.

A number of other useful debug settings are defined in `src/settings.js <https://github.com/emscripten-core/emscripten/blob/main/src/settings.js>`_. For more information, search that file for the keywords "check" and "debug".

.. _debugging-sanitizers:

Sanitizers
==========

Emscripten also supports some of Clang's sanitizers, such as :ref:`sanitizer_ubsan` and :ref:`sanitizer_asan`.

.. _debugging-emcc-v:

emcc verbose output
===================

Compiling with the :ref:`emcc -v <emcc-verbose>` will cause Emscripten to output
the sub-command that it runs as well as passes ``-v`` to Clang.

.. _debugging-manual-debugging:

Manual print debugging
======================

You can also manually instrument the source code with ``printf()`` statements, then compile and run the code to investigate issues. Note that ``printf()`` is line-buffered, make sure to add ``\n`` to see output in the console.

If you have a good idea of the problem line you can add ``print(new Error().stack)`` to the JavaScript to get a stack trace at that point. Also available is :js:func:`stackTrace`, which emits a stack trace and also tries to demangle C++ function names if ``DEMANGLE_SUPPORT`` is enabled (if you don't want or need C++ demangling in a specific stack trace, you can call :js:func:`jsStackTrace`).

Debug printouts can even execute arbitrary JavaScript. For example::

  function _addAndPrint($left, $right) {
    $left = $left | 0;
    $right = $right | 0;
    //---
    if ($left < $right) console.log('l<r at ' + stackTrace());
    //---
    _printAnInteger($left + $right | 0);
  }


.. _handling-c-exceptions-from-javascript:

Handling C++ exceptions from JavaScript
=======================================

C++ exceptions are thrown from WebAssembly using exception pointers, which means
that try/catch/finally blocks in JavaScript will only receive a number, which
represents a pointer into linear memory. In order to get the exception message,
the user will need to create some WASM code which will extract the meaning from
the exception. In the example code below we created a function that receives the
address of a ``std::exception``, and by casting the pointer
returns the ``what`` function call result.

.. code-block:: cpp

  #include <emscripten/bind.h>

  std::string getExceptionMessage(intptr_t exceptionPtr) {
    return std::string(reinterpret_cast<std::exception *>(exceptionPtr)->what());
  }

  EMSCRIPTEN_BINDINGS(Bindings) {
    emscripten::function("getExceptionMessage", &getExceptionMessage);
  };

This requires using the linker flags ``-lembind -sEXPORT_EXCEPTION_HANDLING_HELPERS``.
Once such a function has been created, exception handling code in javascript
can call it when receiving an exception from WASM. Here the function is used
in order to log the thrown exception.

.. code-block:: javascript

  try {
    ... // some code that calls WebAssembly
  } catch (exception) {
    console.error(Module.getExceptionMessage(exception));
  } finally {
    ...
  }

It's important to notice that this example code will work only for thrown
statically allocated exceptions. If your code throws other objects, such as
strings or dynamically allocated exceptions, the handling code will need to
take that into account. For example, if your code needs to handle both native
C++ exceptions and JavaScript exceptions you could use the following code to
distinguish between them:

.. code-block:: javascript

  function getExceptionMessage(exception) {
    return typeof exception === 'number'
      ? Module.getExceptionMessage(exception)
      : exception;
  }

.. _debugging-emscripten-specific-issues:

Emscripten-specific issues
==========================

Memory Alignment Issues
-----------------------

The :ref:`Emscripten memory representation <emscripten-memory-model>` is compatible with C and C++. However, when undefined behavior is involved you may see differences with native architectures, and also differences between Emscripten's output for asm.js and WebAssembly:

- In asm.js, loads and stores must be aligned, and performing a normal load or store on an unaligned address can fail silently (access the wrong address). If the compiler knows a load or store is unaligned, it can emulate it in a way that works but is slow.
- In WebAssembly, unaligned loads and stores will work. Each one is annotated with its expected alignment. If the actual alignment does not match, it will still work, but may be slow on some CPU architectures.

.. tip:: :ref:`SAFE_HEAP <debugging-SAFE-HEAP>` can be used to reveal memory alignment issues.

Generally it is best to avoid unaligned reads and writes — often they occur as the result of undefined behavior, as mentioned above. In some cases, however, they are unavoidable — for example if the code to be ported reads an ``int`` from a packed structure in some pre-existing data format. In that case, to make things work properly in asm.js, and be fast in WebAssembly, you must be sure that the compiler knows the load or store is unaligned. To do so you can:

- Manually read individual bytes and reconstruct the full value
- Use the :c:type:`emscripten_align* <emscripten_align1_short>` typedefs, which define unaligned versions of the basic types (``short``, ``int``, ``float``, ``double``). All operations on those types are not fully aligned (use the ``1`` variants in most cases, which mean no alignment whatsoever).


Function Pointer Issues
-----------------------

If you get an ``abort()`` from a function pointer call to ``nullFunc`` or ``b0`` or ``b1`` (possibly with an error message saying "incorrect function pointer"), the problem is that the function pointer was not found in the expected function pointer table when called.

.. note:: ``nullFunc`` is the function used to populate empty index entries in the function pointer tables (``b0`` and ``b1`` are shorter names used for ``nullFunc`` in more optimized builds).  A function pointer to an invalid index will call this function, which simply calls ``abort()``.

There are several possible causes:

- Your code is calling a function pointer that has been cast from another type (this is undefined behavior but it does happen in real-world code). In optimized Emscripten output, each function pointer type is stored in a separate table based on its original signature, so you *must* call a function pointer with that same signature to get the right behavior (see :ref:`portability-function-pointer-issues` in the code portability section for more information).
- Your code is calling a method on a ``NULL`` pointer or dereferencing 0. This sort of bug can be caused by any sort of coding error, but manifests as a function pointer error because the function can't be found in the expected table at runtime.

In order to debug these sorts of issues:

- Compile with ``-Werror``. This turns warnings into errors, which can be useful as some cases of undefined behavior would otherwise show warnings.
- Use ``-sASSERTIONS=2`` to get some useful information about the function pointer being called, and its type.
- Look at the browser stack trace to see where the error occurs and which function should have been called.
- Enable clang warnings on dangerous function pointer casts using ``-Wcast-function-type``.
- Build with :ref:`SAFE_HEAP=1 <debugging-SAFE-HEAP>`.
- :ref:`Sanitizers` can help here, in particular UBSan.

Another function pointer issue is when the wrong function is called. :ref:`SAFE_HEAP=1 <debugging-SAFE-HEAP>` can help with this as it detects some possible errors with function table accesses.


Infinite loops
--------------

Infinite loops cause your page to hang. After a period the browser will notify the user that the page is stuck and offer to halt or close it.

If your code hits an infinite loop, one easy way to find the problem code is to use a *JavaScript profiler*. In the Firefox profiler, if the code enters an infinite loop you will see a block of code doing the same thing repeatedly near the end of the profile.

.. note:: The :ref:`emscripten-runtime-environment-main-loop` may need to be re-coded if your application uses an infinite main loop.

.. _debugging-profiling:

Profiling
=========

Speed
-----

To profile your code for speed, build with :ref:`profiling info <emcc-profiling>`,
then run the code in the browser's devtools profiler. You should then be able to
see in which functions is most of the time spent.

.. _debugging-profiling-memory:

Memory
------

The browser's memory profiling tools generally only understand
allocations at the JavaScript level. From that perspective, the entire linear
memory that the emscripten-compiled application uses is a single big allocation
(of a ``WebAssembly.Memory``). The devtools will not show information about
usage inside that object, so you need other tools for that, which we will now
describe.

Emscripten supports
`mallinfo() <https://man7.org/linux/man-pages/man3/mallinfo.3.html>`_, which lets
you get information from ``dlmalloc`` about current allocations. For example
usage, see
`the test <https://github.com/emscripten-core/emscripten/blob/9bb322f8a7ee89d6ac67e828b9c7a7022ddf8de2/tests/mallinfo.cpp>`_.

Emscripten also has a ``--memoryprofiler`` option that displays memory usage
in a visual manner, letting you see how fragmented it is and so forth. To use
it, you can do something like

.. code-block:: bash

  emcc test/hello_world.c --memoryprofiler -o page.html

Note that you need to emit HTML as in that example, as the memory profiler
output is rendered onto the page. To view it, load ``page.html`` in your
browser (remember to use a :ref:`local webserver <faq-local-webserver>`). The display
auto-updates, so you can open the devtools console and run a command like
``_malloc(1024 * 1024)``. That will allocate 1MB of memory, which will then show
up on the memory profiler display.

.. _debugging-autodebugger:

AutoDebugger
============

The *AutoDebugger* is the 'nuclear option' for debugging Emscripten code.

.. warning:: This option is primarily intended for Emscripten core developers.

The *AutoDebugger* will rewrite the output so it prints out each store to memory. This is useful because you can compare the output for different compiler settings in order to detect regressions.

The *AutoDebugger* can potentially find **any** problem in the generated code, so it is strictly more powerful than the ``CHECK_*`` settings and ``SAFE_HEAP``. One use of the *AutoDebugger* is to quickly emit lots of logging output, which can then be reviewed for odd behavior. The *AutoDebugger* is also particularly useful for :ref:`debugging regressions <debugging-autodebugger-regressions>`.

The *AutoDebugger* has some limitations:

-  It generates a lot of output. Using *diff* can be very helpful for identifying changes.
-  It prints out simple numerical values rather than pointer addresses (because pointer addresses change between runs, and hence can't be compared). This is a limitation because sometimes inspection of addresses can show errors where the pointer address is 0 or impossibly large. It is possible to modify the tool to print out addresses as integers in ``tools/autodebugger.py``.

To run the *AutoDebugger*, compile with the environment variable ``EMCC_AUTODEBUG=1`` set. For example:

.. code-block:: bash

  # Linux or macOS
  EMCC_AUTODEBUG=1 emcc test/hello_world.cpp -o hello.html

  # Windows
  set EMCC_AUTODEBUG=1
  emcc test/hello_world.cpp -o hello.html
  set EMCC_AUTODEBUG=0


.. _debugging-autodebugger-regressions:

AutoDebugger Regression Workflow
---------------------------------

Use the following workflow to find regressions with the *AutoDebugger*:

- Compile the working code with ``EMCC_AUTODEBUG=1`` set in the environment.
- Compile the code using ``EMCC_AUTODEBUG=1`` in the environment again, but this time with the settings that cause the regression. Following this step we have one build before the regression and one after.
- Run both versions of the compiled code and save their output.
- Compare the output using a *diff* tool.

Any difference between the outputs is likely to be caused by the bug.

.. note::
    You may want to use ``-sDETERMINISTIC`` which will ensure that timing
    and other issues don't cause false positives.


Useful Links
============

- `Blogpost about reading compiler output <http://mozakai.blogspot.com/2014/06/looking-through-emscripten-output.html>`_.
- `GDC 2014: Getting started with asm.js and Emscripten <https://web.archive.org/web/20140325222509/http://people.mozilla.org/~lwagner/gdc-pres/gdc-2014.html#/20>`_ (Debugging slides).

Need help?
==========

The :ref:`Emscripten Test Suite <emscripten-test-suite>` contains good examples of almost all functionality offered by Emscripten. If you have a problem, it is a good idea to search the suite to determine whether test code with similar behavior is able to run.

If you've tried the ideas here and you need more help, please :ref:`contact`.
