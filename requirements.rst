Requirements
============

Motivation
----------
Quoting the creator of LLVM:

  There are multiple reasons why pieces of GCC cannot be reused as
  libraries, including rampant use of global variables, weakly enforced
  invariants, poorly designed data structures, sprawling code base, and
  the use of macros that prevent the codebase from being compiled to
  support more than one front-end/target pair at a time. The hardest
  problems to fix, though, are the inherent architectural problems that
  stem from its early design and age. Specifically, GCC suffers from
  layering problems and leaky abstractions: the back end walks front-end
  ASTs to generate debug info, the front ends generate back-end data
  structures, and the entire compiler depends on global data structures
  set up by the command line interface.

  -- Chris Lattner (from
     http://www.drdobbs.com/architecture-and-design/the-design-of-llvm/240001128 ;
     also in http://www.aosabook.org/en/llvm.html )

He missed a few issues, for example, the fact that xmalloc() calls
exit() if it fails.  This is very ill-mannered to be doing from inside a
library (an app might have caches that it can evict, or run a garbage
collection, or whatnot), but we'll have to live with it for now (the
whole of the GTK stack has this behavior also, alas).

I'm focusing on the first item on this list: removing the "rampant use
of global variables".

Use case: Just-In-Time Compilation
----------------------------------
Consider a web browser, where each tab or window can have multiple
threads, say, a thread to render HTML, a thread to run JavaScript, etc.
The JavaScript code is to be compiled to machine code to get maximum
speed.  How is each tab to do this?   The author of the web browser
needs to be able to embed a compiler into the process, where each thread
might want to do some compiling, each independently of the other
threads.   The compilation will involve some optimization passes - some
of them the ones already implemented in GCC, but maybe some extra ones
that are specific to the JavaScript implementation.

Similar situations arise in other language interpreters that wish to
compile bytecodes to machine code.

Use case: unit tests for optimization passes
--------------------------------------------
Having GCC as a shared library offers the ability to create real unit tests
for optimization passes, where the test directly constructs a fragment
of Gimple or RTL, and works on that, asserting properties of the result,
without having the risk of other optimization passes making changes to the
code that invalidates the testing.   Multiple tests could be run within one
process, potentially speeding up such a test suite dramatically when
compared to the current test suite.

Use case: Static Analysis
-------------------------
My own primary interest is in using GCC's code to build a static analysis
tool for finding bugs in C/C++ code - in my case, for hardening the Fedora
distribution against security vulnerabilities - though what I build should
be usable by other Free Software operating systems.  I don't think
everything I want to do can be done as a plugin, so I'm interested in a
more modular GCC.

Scope of the problem
--------------------
There are about 3500 global variables in GCC, used in slightly over 100000
sites in the code.

See
http://gcc.gnu.org/ml/gcc/2013-05/msg00015.html
which orders the global variables based on the number of sites
referencing them during a build of stage1.

Detailed notes on each of the top 40 items in the list can be seen
in :ref:`Appendix 1 <topglobals>`.

Detailed notes on internal state for all GCC passes can be seen in
:ref:`Appendix 2 <passes>`.

Detailed notes on internal state within the support libraries can be seen
in :ref:`Appendix 3 <supportlibs>`. (i.e. everything that's not in the
"gcc" subdirectory).

This document doesn't cover binutils (e.g. the GNU assembler) or gdb.
Note that to fully implement an in-process embedded JIT using GCC would
require also tackling binutils, since GCC emits assembler.

No Change in Other Functionality
--------------------------------
Although these changes enable a future gcc-as-a-library, I intend for
my patches to make no outwardly-observable changes to GCC's behavior:
all tests should continue to have the same results.

No change of license
--------------------
GCC's code will continue to be under its existing license.

Performance
-----------
People have expressed concerns about the performance impact of these
changes.

Some of the concerns have been:

  * increased register pressure from passing around a "this" pointer
    everywhere
  * increased memory usage if we store a context ptr in a commonly-used
    data structure (e.g. types)
  * increased startup time from extra mallocs (rather than static
    allocation)
  * performance hit from position-independent code relative to status quo

To be usable as a shared library, code needs to be built into
position-independent machine code, using one of `-fPIC` or `-fpic`, but
both of these have a performance impact relative to position-dependent
code.

Given this, I think the model is a configure-time switch to enable usage
as a shared library, which turns on one of `-fPIC` or `-fpic`
(TODO: which?).

The default would be the status quo: position-dependent code, creating
statically-linked executables.

This choice of shared/non-shared can then be expressed in a config #define
which can then affect the build: any changes which might affect performance
can be wrapped with a suitable #define, potentially affecting ABI (as seen
below), so you can't change the define without a full rebuild, better to
have two builddirs, one configured with, one without.

Hence testing this will require two bootstraps: a non-shared and a shared.

This potentially doubles the test matrix, but that's the cost of introducing
a configuration switch for this.

My workflow would probably be something like this::

   # (from a git clone in a "src" subdir)
   [david@surprise src]$ cd ..
   [david@surprise gcc-coding]$ mkdir build-static
   [david@surprise gcc-coding]$ pushd build-static
   # configure a traditional static build:
   [david@surprise build-static]$ ../src/configure
   # ...and do the 3-stage bootstrap:
   [david@surprise build-static]$ make -j4
   [david@surprise build-static]$ make -j4 check
   [david@surprise build-static]$ popd
   [david@surprise gcc-coding]$ mkdir build-shared
   [david@surprise gcc-coding]$ pushd build-shared
   # configure a shared-library-enabled build
   # (obviously not the final configure flag):
   [david@surprise build-shared]$ ../src/configure --enable-my-shared-stuff
   # ...and do the 3-stage bootstrap:
   [david@surprise build-shared]$ make -j4
   [david@surprise build-shared]$ make -j4 check
   [david@surprise build-shared]$ popd

What should the configure flag be called?
See http://gcc.gnu.org/ml/gcc-patches/2013-07/msg00093.html for some
notes on this.


Benchmarking
^^^^^^^^^^^^
Changes that might have a performance impact can be benchmarked to mitigate
risk.

I started a benchmarking suite here:
http://git.engineering.redhat.com/?p=users/dmalcolm/gcc-benchmarking.git;a=summary


Debuggability
-------------
It's important that the compiler is still debuggable.

TODO: add notes below on what the changes below do to the experience in gdb,
and to the experience in valgrind.


Ability to Backport
-------------------
All changes to the trunk impact the ability to backport other changes to
older branches.  To minimize increased pain of maintenance branches I will
attempt to minimize the textual differences of the changes.

For example, many of the proposed changes involve converting functions to
be methods of a class, with variables becoming fields.

In theory, field names should have trailing underscores, but we will not
add them when making these changes, to minimize the patch delta: the bodies
of most functions will be untouched.

Converting a function to a class method can be done with a patch of this
form to the implementation::

  --- foo.c
  +++ foo.c

    void
  + some_class::
    impl_foo (void)
    {

without disturbing the internals of the file..

This would change the internal prototypes more substantially::

  --- foo.c
  +++ foo.c

  - static void impl_foo (void);
  - static void impl_bar (void);
  +
  + class foo_state
  + {
  + public:
  +   void impl_foo (void);
  + private:
  +   void impl_bar (void);
  + }; // class foo_state

No New Requirements for Static Build
------------------------------------
The plan calls for the use of thread-local-storage in a few places for the
shared-library build, but such requirements must be confined to the
shared-library build; they must not be needed in the global-state build,
since some existing hosts may not support TLS.

GCC 4.9 schedule
----------------
One other concern is how all of this lines up with GCC 4.9's schedule.
These big internal reorganizations need to happen in stage 1 of the
schedule, right?  Not sure where that is calendar-wise, but my
hope is to get the big reorg changes in sooner rather than later.
