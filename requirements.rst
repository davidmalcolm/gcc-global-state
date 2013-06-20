Requirements
============

Motivation
----------
Quoting the creator of LLVM (from
http://www.drdobbs.com/architecture-and-design/the-design-of-llvm/240001128 ),

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

  -- Chris Lattner (May 29, 2012)

He missed a few issues, for example, the fact that xmalloc() calls
exit() if it fails.  This is very ill-mannered to be doing from inside a
library (an app might have caches that it can evict, or run a garbage
collection, or whatnot), but we'll have to live with it for now (the
whole of the GTK stack has this behavior also, alas).

I'm focusing on the first item on this list: removing the "rampant use
of global variables".

Scope of the problem
--------------------
There are about 3500 global variables in GCC, used in slightly over 100000
sites in the code.

See
http://gcc.gnu.org/ml/gcc/2013-05/msg00015.html
which orders the global variables based on the number of sites
referencing them during a build of stage1.

Detailed notes on each of the top 40 items in the list can be seen
in Appendix 1.

Detailed notes on internal state for all GCC passes can be seen in
Appendix 2.

Detailed notes on internal state within the support libraries will appear in
Appendix 3 (i.e. everything that's not in the "gcc" subdirectory).

This document doesn't cover binutils (e.g. the GNU assembler) or gdb.

No Change in Other Functionality
--------------------------------
Although these changes enable a future gcc-as-a-library, I intend for
my patches to make no outwardly-observable changes to GCC's behavior:
all tests should continue to have the same results.


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
position-independent machine code, using one of -fPIC/-fpic, but both of
these have a performance impact relative to position-dependent code.

Given this, I think the model is a configure-time switch to enable usage
as a shared library, which turns on one of -fPIC or -fpic (TODO: which?).

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

There are some more invasive changes, but I feel they need making
(e.g. removal of cfun macros).
