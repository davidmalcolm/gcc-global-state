Various kinds of pass-local state
=================================
I went through every pass in GCC, looking at what internal state each one
has.

The full details can be seen in :ref:`Appendix 2 <passes>`, but they can
be summarized by classifying passes by their "state-management"
characteristics:

* Single-instance passes vs multiple-instances passes.  For example,
  `pass_build_cgraph_edges` only appears once in the pass pipeline, whereas
  `pass_copy_prop` appears in 8 places (I believe this one holds the record).

* Passes that have their own source file vs those that share their source
  file with other pass(es).

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

* Passes that exist merely to cleanup other (global) state
  (e.g. `pass_ipa_free_lang_data`, `pass_release_ssa_names`)

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

* Shared pass state.   There are two instances of "pass_vrp", which share
  the various states within tree-vrp.c

  In a shared-library build, we need to ensure that these instances share
  state *within* a context/universe, but not with other contexts/universes::

    Universe A:                        Universe B:
    ===========                        ===========
    pass_vrp_0:A                       pass_vrp_0:B
                ↘                                  ↘
                 pass_vrp_state:A                   pass_vrp_state:B
                ↗                                  ↗
    pass_vrp_1:A                       pass_vrp_1:B

* Sometimes state needs to be shared between multiple kinds of pass within a
  context/universe.

  An example is `tree-vect-generic.c`, where the single-instanced
  pass_lower_vector and pair of pass_lower_vector_ssa instances share
  state, but only within their respective universes::

    Universe A:                       ║  Universe B:
    ===========                       ║  ===========
    pass_lower_vector:A────────────╮  ║  pass_lower_vector:B────────────╮
    pass_lower_vector_ssa_0:A────╮ │  ║  pass_lower_vector_ssa_0:B────╮ │
    pass_lower_vector_ssa_1:A──╮ │ │  ║  pass_lower_vector_ssa_1:B──╮ │ │
                               ↓ ↓ ↓  ║                             ↓ ↓ ↓
                lower_vector_state:A  ║              lower_vector_state:B

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

* Source files with complicated interactions of state that don't easily
  fit into the above patterns.

  Examples:

    * `tree-mudflap.c` (where other parts of the compiler call into
      an API that shares state with the pass)

    * `tree-ssa-uninit.c`: pass_late_warn_uninitialized exposes its
      state via `ssa_undefined_value_p`


Proposed implementation
-----------------------
There will be a new `class pipeline` encapsulating pass management.

http://gcc.gnu.org/ml/gcc-patches/2013-04/msg00182.html

Passes will become C++ classes.

Passes "know" which universe they are in: they will be constructed with
a `universe&`, stored as a field, making this information easily accessible
in the gate and execute hooks.

For each of the above state-management patterns, we move the state into
a new C++ class, converting functions to methods as necessary.

These classes will be singletons in the static build vs multiple instances
in the shared-library build.

Per-invocation state with no GTY markings
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
If we can use the new `force_static` attribute, these become fairly simple:
use the singleton optimization described elsewhere.

We put an instance of the state on the stack in the execute callback of
the pass.

In the shared-library build, this is "real" state, whereas in a non-shared
build this is a dummy empty object, and can be optimized away in favor of
global state.

We can't have on-stack GC roots, so if there are GTY markings, we need to
use one of the approaches below.

An example, from `tracer.c`::

  namespace {

  class MAYBE_SINGLETON tracer_state
  {
  public:
    tracer_state();

    bool tail_duplicate ();

  private:

    edge find_best_successor (basic_block);
    edge find_best_predecessor (basic_block);
    int find_trace (basic_block, basic_block *);
    void mark_bb_seen (basic_block bb);
    bool bb_seen_p (basic_block bb);

  private:

    /* Minimal outgoing edge probability considered for superblock formation.  */
    int probability_cutoff;
    int branch_ratio_cutoff;

    /* A bit BB->index is set if BB has already been seen, i.e. it is
       connected to some trace already.  */
    sbitmap bb_seen;
  }; // tracer_state

  } // anon namespace

  /* If we're actually using global state, we need definitions of the
     global fields. *
  #if USING_IMPLICIT_STATIC
  int tracer_state::probability_cutoff;
  int tracer_state::branch_ratio_cutoff;
  sbitmap tracer_state::bb_seen;
  #endif

  /* This is the top-level point within this pass' execution where state
     exists.  */
  bool
  tracer_state::tail_duplicate ()
  {
    /* ... snip .. */

    /* In a shared-library build, the state is on the stack.
       In a non-shared build, this object is empty and redundant and should
       be optimized away.  */
    tracer_state state;

    changed = state.tail_duplicate ();
    /* (this is a synonym of tracer_state::tail_duplicate () in a
        non-shared build) */

    /* ... snip .. */
  }

If we can't use the "force_static optimization", we need other approaches.
I posted a patch for `tracer.c` as:
http://gcc.gnu.org/ml/gcc-patches/2013-05/msg01318.html
and the followup:
http://gcc.gnu.org/ml/gcc-patches/2013-05/msg01351.html
gives a general way of dealing with these.

Richard Henderson posted a couple of other approaches as:
http://gcc.gnu.org/ml/gcc-patches/2013-05/msg01395.html
and:
http://gcc.gnu.org/ml/gcc-patches/2013-05/msg01415.html

Essentially we put the class in an anonymous namespace, and have a global
singleton.   The optimizer should be smart enough to see that "this" is
always &the_singleton and copy-propagate.

In the shared-library build, we instead put the value on the stack in
the execute callback of the pass.

In my tests it wasn't clear that the optimizer was always smart enough
to eliminate the "this", which is why I favor the "force_static" approach.


Pass state with GTY markings
^^^^^^^^^^^^^^^^^^^^^^^^^^^^
If there are GTY markings, we need to add `GTY((user))` to the new class
and manually write the gty hooks (gengtype doesn't seem to be up to the
task in my experiments).

How the marking hook gets called depends on further aspects below.


State shared by pass instances
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
For the pass_vrp case::

    Universe A:                        Universe B:
    ===========                        ===========
    pass_vrp_0:A                       pass_vrp_0:B
                ↘                                  ↘
                 pass_vrp_state:A                   pass_vrp_state:B
                ↗                                  ↗
    pass_vrp_1:A                       pass_vrp_1:B

The plan for dealing with these in a gcc-as-a-library setting is that
the `opt_pass` base class gains a clone method::

   class opt_pass
   {
   public:
      // ...snip...
      
      virtual opt_pass *clone ();
   }; // class opt_pass

   /* Passes have to explicitly opt-in to be clonable,
      by implementing their own clone method.  */
   opt_pass*
   opt_pass::clone ()
   {
     internal_error ("pass %s does not support cloning", name);
   }

so that when clones are created, the passes can "wire up" the shared state
appropriately::

  namespace {

  class MAYBE_SINGLETON foo_state
  {
    // functions and data for the whole pass
    int some_field;
  }; // class foo_state

  } // anon namespace

  /* Global data for the non-shared build.  */
  #if USING_IMPLICIT_STATIC
  int foo_state::some_field;
  #endif

  /* Singleton instance for non-shared build.  This will be an
     empty dummy object in stages 2 and 3 (once "force_static" is
     usable).  */
  IF_GLOBAL_STATE(static foo_state the_foo;)

  class pass_foo : public gimple_pass
  {
  public:
    pass_foo(universe &uni, pass_state *state)
      : state_(state)
    { }

    /* Clone the pass, sharing state.  */
    opt_pass*
    opt_pass clone ()
    {
      return new pass_foo(uni, state);
    }

    /* The bulk of the work happens in the state;
       we only dereference once.  */
    unsigned int execute () { state_->execute (); }

  private:
    foo_state *state_;
  }; // class pass_foo

  /* Create first instance of pass, with its own state.  */
  opt_pass *
  make_pass_foo (universe &uni)
  {
    return new pass_foo(uni,
                        IF_GLOBAL_VS_SHARED(&the_foo_state,
                                            new foo_state));
  }

Then the first_instance gets responsibility for creating the pass state
and all the clones can share it, but the state is "local" to the universe,
whilst staying simple and efficient for the "global state" case.

If the state is GTY-marked, then the passes need to call the state's gty
hooks from their gty hooks.

More complicated arrangements
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
The singleton tricks from above are widely applicable.

For any kind of "state-wiring" more complicated than the above, we'll
simply put a reference to the shared state into the universe/context
object, and have the passes locate it there (either at pass creation,
or when they run).

For example::

   class universe
   {
   public:
       /* ... snip ... */

       /* State shared by many passes. */
       MAYBE_STATIC struct df_d *df_;
       MAYBE_STATIC redirect_edge_var_state *edge_vars_;

       /* Passes that have special state-handling needs.  */
       MAYBE STATIC mudflap_state *mudflap_;
       MAYBE STATIC lower_vector_state *lower_vector_;

   }; // class universe

In a global-state build these state instances will be singletons and thus
global variables.  In a shared-library build these state instances will be
allocated when the universe is constructed.

If the state is GTY-marked, then the universe needs to call the state's gty
hooks when the universe's gty hooks run.
