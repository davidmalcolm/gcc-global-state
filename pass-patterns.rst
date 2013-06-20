Various kinds of pass-local state
=================================
I went through every pass in GCC, looking at what internal state it has.

The full details can be seen in Appendix 2, but they can be summarized by
classifying passes by their "state-management" characteristics:

* Single-instance passes vs multiple-instances passes.  For example,
  `pass_build_cgraph_edges` only appears once in the pass pipeline, whereas
  `pass_copy_prop` appears in 8 places (I believe this is the record).

* Passes that have their own source file vs shares their source file with
  other pass(es).

  For an example of passes sharing a source file, see
  `tree-vect-generic.c`: where two instances of `pass_lower_vector_ssa`
  and an instance of `pass_lower_vector` have shared state, which isn't
  visible to the rest of the compiler.

* Passes with no internal state.

  Examples include:

    * `stack-ptr-mod.c`: pass_stack_ptr_mod
    * `tree-ssa-ifcombine.c`: pass_tree_ifcombine
    * `tree-ssa-loop-ch.c`: pass_ch
    * `tree-ssa-phiprop.c`: pass_phiprop

* Passes in which the internal state is already encapsulated by passing
  around a ptr to a struct.

  Examples include:

    * `gimple-low.c`: `pass_lower_cf`, which uses `(struct lower_data *)`
    * `tree-stdarg.c`: `pass_stdarg`, which uses `(struct stdarg_info *)`

* Passes where there are static variables in the underlying .c file, but
  in which the state is fully cleaned at the start/end of each invocation
  of the pass (i.e. for each function, for non-IPA passes).

  I've been calling this pattern *"per-invocation state"*

  There are numerous such passes; some examples are:

    * `compare-elim.c`: pass_compare_elim_after_reload
    * `mode-switching.c`: pass_mode_switching
    * `tree-loop-distribution.c`: pass_loop_distribution
    * `ree.c`: pass_ree
    * `regcprop.c`: pass_cprop_hardreg
    * `tracer.c`: pass_tracer
    * `tree-loop-distribution.c`: pass_loop_distribution
    * `tree-ssa-copy.c`: pass_copy_prop
    * `tree-ssa-math-opts.c` (all 4 passes)
    * `tree-ssa-reassoc.c`: pass_reassoc
    * `tree-ssa-sink.c`: pass_sink_code
    * `tree-ssa-strlen.c`: pass_strlen
    * `tree-ssa-uncprop.c`: pass_uncprop

  I posted a patch for tracer.c as
  http://gcc.gnu.org/ml/gcc-patches/2013-05/msg01318.html
  and the followup:
  http://gcc.gnu.org/ml/gcc-patches/2013-05/msg01351.html
  gives a general way of dealing with these.

  Richard Henderson posted a couple of other approaches as:
  http://gcc.gnu.org/ml/gcc-patches/2013-05/msg01395.html
  and:
  http://gcc.gnu.org/ml/gcc-patches/2013-05/msg01415.html

* Per-invocation state as above, but where the lifetime of the state is
  localized to a subset of the functions within the pass.

  An example is `tree-loop-distribution.c`: pass_loop_distribution,
  which has state that only lives within calls to `ldist_gen` and below,
  which is only a part of the pass

  This pattern can be dealt with like per-invocation state, but we
  can restrict where the state lives to keep in contained.  In the
  above example, we could have a `class ldist_gen_state` to emphasize
  that this state only lives during this part of the pass.

  Other examples:

  * `tree-if-conv.c`: pass_if_conversion

* Passes with one-time-initialized state, which is private to the pass.

  Any examples?

  I had thought that `tree-profile.c` (pass_ipa_tree_profile) was one:
  the first time in it creates tree nodes that will be shared by the
  manipulation of every function the pass touches, but which aren't
  used outside of the pass' code.  However the creation hook can be
  called from `profile.c` so we have to expose this poking of the state
  in case the time of initialization affects the results.

* Passes with one-time-initialized state (which could perhaps be shared
  with other contexts?)

* Passes where the state may persist from invocation to invocation (e.g.
  stats)

* Passes with non-static state, visible to other parts of the compile
  (reginfo.c?)

* Passes with GTY(()) state.  See e.g. `tree-vect-generic.c`

* Passes that exists merely to cleanup other (global) state
  (e.g. `pass_ipa_free_lang_data`, `pass_release_ssa_names`)

* Source files with complicated interactions of state that don't easily
  fit into the above patterns.

  Examples:

    * `tree-mudflap.c` (where other parts of the compiler call into
      an API that shares state with the pass)

    * `tree-ssa-uninit.c`: pass_late_warn_uninitialized exposes its
      state via `ssa_undefined_value_p`

The approach I've proposed (tackling tracer.c) covers per-pass state
when there's only ever a single instance of the pass within a universe,
but I haven't yet posted how I plan to deal with per-pass state that's
shared between multiple pass instances.   For example, there are two
instances of "pass_vrp", which share the various states within
tree-vrp.c

One plan for dealing with these in a gcc-as-a-library setting is that
when the passes are created, the factory function is passed in a
pointer to the first instance of that pass within the current universe::

  extern opt_pass *
  make_pass_vrp (universe &uni, opt_pass *first_instance);

This pointer will be NULL for the first "pass_vrp" instance, and
subsequent instances will get the pointer to the first.  There's a
contract in the API between the manager and the passes that
first_instance will, if non-NULL, be an instance of the same subclass of
opt_pass that the function returns, so that make_pass_vrp can safely
cast it to the correct opt_pass subclass, and the details of the
opt_pass subclasses can stay encapsulated away inside their
individual .c files.

Another is similar, but instead passes have a clone method::

  class opt_pass
  {
  public:
    ...
    virtual opt_pass * clone() = 0;
    ...
  };

with this in tree-vrp.c::

  class pass_vrp : public gimple_opt_pass
  {
  public:
    pass_vrp(context &ctxt, pass_vrp *first_instance)
      : gimple_opt_pass(/*...snip...*/)

    /*...snip...*/

   opt_pass * clone() { return new pass_vrp (ctxt, this); }

    /*...snip...*/
  };

  extern opt_pass *
  make_pass_vrp (context &ctxt);
  /* this function makes the initial instance of the pass */


Then the first_instance gets responsibility for managing the pass state
(e.g. with a pass_vrp_state field), and all other instances can access
it - thus we have shared state, but the state is "local" to the universe::

  Universe A:                        Universe B:
  ===========                        ===========
  pass_vrp_0:A                       pass_vrp_0:B
              ↘                                  ↘
               pass_vrp_state:A                   pass_vrp_state:B
              ↗                                  ↗
  pass_vrp_1:A                       pass_vrp_1:B

(there are unicode arrow chars in the above "ascii" art, in case they're
not visible)

Once passes are C++ classes (automated), we could convert passes one at
a time to this model::

  /* State shared between multiple instances of pass_foo.  */
  class foo_state
  {
     /* Functions become MAYBE_STATIC methods of foo_state as necessary
        making most of them private, apart from the hooks called by
        the pass execution callback.  */

     /* Data become MAYBE_STATIC private fields of foo_state.  */
  };

  /* An instance of a pass (either the "main" one, or a "secondary"),
     with a reference to shared state.  */
  class pass_foo : public gimple_pass
  {
  protected:
     pass_foo(context &ctxt,
              foo_state &shared_state)

     /* Create secondary pass, sharing state with this one.
        All such clones will share state.  */
     opt_pass *clone() { return new pass_foo(ctxt, shared_state); }

  private:
     foo_state &shared_state;
  };

  /* The first pass to be created in a context "owns" the state.  */
  class main_pass_foo : public pass_foo
  {
  public:
     main_pass_foo(context &ctxt)
       : pass_foo(ctxt, shared_state)
     {}

  private:
     MAYBE_STATIC foo_state actual_state;
  };

  opt_pass *make_pass_foo (context &ctxt) { return main_pass_foo(ctxt); }

(maybe "stateful_pass_foo" rather than just "main_pass_foo"?  better naming?)

This gives us state shared between all instances of a pass within a
context/universe, but separate to instances of that pass in other universes,
and hidden from the rest of the code.


Sometimes state needs to be shared between multiple kinds of pass within a
context/universe.

An example is `tree-vect-generic.c`, where the single-instanced
pass_lower_vector and pair of pass_lower_vector_ssa instances share
state within their respective universes::


  Universe A:                        Universe B:
  ===========                        ===========
  pass_lower_vector:A────────────╮   pass_lower_vector:B────────────╮
  pass_lower_vector_ssa_0:A────╮ │   pass_lower_vector_ssa_0:B────╮ │
  pass_lower_vector_ssa_1:A──╮ │ │   pass_lower_vector_ssa_1:B──╮ │ │
                             ↓ ↓ ↓                              ↓ ↓ ↓
              lower_vector_state:A               lower_vector_state:B

To handle this case, I'm considering two approaches:

  * a variant on the above scheme (pass_vrp), in which the first instance
    of any pass within the group to be created owns the state, and
    instances of other kinds of pass manually look up that instance via the
    pipeline object.

    Example: if pass_foo is created first, then pass_bar can share state
    with it like this::

      opt_pass *make_pass_bar (context &ctxt)
      {
        /* Locate the shared state my hardcoding a reference to a pass
           that already has it: */
        foo_pass *reference_pass = ctxt.pipeline->pass_bar_1;
        gcc_assert (reference_pass);
        foo_state &shared_state = reference_pass->get_shared_state ();
        return new pass_bar (ctxt, shared_state);
      }

    An issue with this approach is that it relies on the reference pass
    being created before any instances of pass_bar, so if the passes get
    reordered there's extra work.  Though we could workaround that
    by creating passes in two phases: creating the passes, then wiring
    up the hierarchy.

  * Putting a reference to the shared state into the universe/context object
    and having the passes locate it there (either at creation, or when they
    run)

    An issue with this is that the universe object gains state classes for
    various specific passes, which seems a little clunky.

Note that in both cases, the GLOBAL_STATE build has empty state objects:
the MAYBE_STATIC means that everything is being done with globals.


GTY pass data
^^^^^^^^^^^^^
Some pass state includes GTY(()) data.  For example `asan.c` has::

  static GTY(()) tree asan_ctor_statements;

which is effectively a local within asan_finish_file, but is currently
exposed as above to ensure it gets marked in case a GC happens within
that function.

Passes have hooks for interacting with the GC - a way to solve the above
issue may be to place such objects into a pass state class (as above),
and to ensure that the pass's GC hooks visit the relevant data (perhaps
by adding GTY hooks to the state class - although it will typically not
be GC-allocated, merely have the ability to own GC-references).


Pass management
---------------
There will be a new `class pipeline` encapsulating pass management.

http://gcc.gnu.org/ml/gcc-patches/2013-04/msg00182.html

Passes become C++ classes
^^^^^^^^^^^^^^^^^^^^^^^^^

See the notes below under "Pass classes" to see what they look like.

Passes "know" which universe they are in
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Passes are constructed with a `universe&`, making this information easily
accessible in the gate and execute hooks.

Remaining work
^^^^^^^^^^^^^^
The big issues remaining here are:

  * integrating with PCH
  * buy-in for having dynamically-allocated passes even in a "static
    build":

     * several hundred extra mallocs at start-up of less than 100 bytes
       each.  Potentially this can be worked around by using placement
       syntax, but is the extra ugliness worth the supposed speedup?
     * debuggability - having to go through the pass manager to get at
       data


