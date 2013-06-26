.. _supportlibs:

Appendix 3: Support Libraries
-----------------------------

intl
^^^^
An in-tree copy of GNU libintl 0.12.1 taken from gettext.

Presumably this is used inside gcc using underscore to lookup localized
copies of messages.

dcigettext.c has::

  /* Lock variable to protect the global data in the gettext implementation.  */
  #ifdef _LIBC
  __libc_rwlock_define_initialized (, _nl_state_lock attribute_hidden)
  #endif

which appears to be used throughout, so presumably the internals are
threadsafe.


libbacktrace
^^^^^^^^^^^^
AFAICS it's a compile-time library for better-handling of crashes of the
compiler.  Not sure if it's thread-safe, but if you've crashed, all bets
are off, I guess.


libcpp
^^^^^^
C preprocessing as a library, used at compile-time.

Mostly keeps its state inside allocated structs, but does have some state:

In files.c::

  static struct pchf_data *pchf;
     /* set in _cpp_read_file_entries;
        used in check_file_against_entries */

In init.c `init_library` has::

  static int initialized = 0;

without a mutex.

lex.c has (for some architectures)::

  static search_line_fast_type search_line_fast;

which is a callback set up by init_vectorized_lexer without a mutex.

macro.c has some stats::

  unsigned num_expanded_macros_counter = 0;
  unsigned num_macro_tokens_counter = 0;

makeucnid.c is a build-time tool for generating a header from the
description of unicode (code points of valid identifiers)


libdecnumber
^^^^^^^^^^^^
Global state was reported by my custom pass as:

  * decumber.c::

      static Unit uarrone[1]={1};

  * "const uint8_t * mfctop"

TODO

libiberty
^^^^^^^^^
Kitchen sink of code, used at compile-time.

alloca.c has::

  static header *last_alloca_header = NULL;	/* -> last alloca header.  */

etc...

TODO

libquadmath
^^^^^^^^^^^
"Quad-Precision Math Library"

TODO

lto-plugin
^^^^^^^^^^
TODO

zlib
^^^^
Yet another copy of zlib

Build glue
^^^^^^^^^^
  * gnattools

  * libada

Run-time libraries
^^^^^^^^^^^^^^^^^^
The following are used at run-time by the output of the compiler, not
as for implementing the compiler itself, and thus aren't relevant to
state removal:

  * boehm-gc (used at run-time by Java and ObjectiveC)

  * libatomic (defines a `libat_`-prefixed API, which appears to be a
    run-time library (written by rth), used by libgo.  Also used as
    a fallback for C/C++ atomic operations that the hardware doesn't
    support

  * libffi (used at run-time by libgo and libjava)

  * libitm (Transactional Memory Library)

  * libgcc

  * libgfortran (presumably runtime support for fortran)

  * libgo (runtime support library for Go)

  * libgomp (runtime support library for OpenMP)

  * libjava (runtime library for gcj)

  * libmudflap (runtime library for mudflap)

  * libobjc (runtime library for Objective C)

  * libsanitizer (runtime library for asan and tsan)

  * libssp (runtime parts of stack protection e.g. `__strcpy_chk`)

  * libstdc++-v3 (C++ standard library)

