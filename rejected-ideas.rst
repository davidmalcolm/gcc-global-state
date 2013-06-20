Appendix 4: Rejected Ideas
--------------------------

Passes are not yet to be invoked on a specific function
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
I had hoped that the execute callback of passes could gain a `function *`
parameter.  Initially this would be `cfun`, but this would give us a way of
eventually eliminating `cfun`.

Plan: don't do this for this milestone (see notes on cfun on the issues
with this).

Rejected ideas relating to sharing pass state
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Rather than a clone method, I thought of adding a 2nd argument to the 
pass factory function.  When the passes are created, the factory function
would be passed in a pointer to the first instance of that pass within
the current universe::

  extern opt_pass *
  make_pass_vrp (universe &uni, opt_pass *first_instance);

This pointer will be NULL for the first "pass_vrp" instance, and
subsequent instances will get the pointer to the first.  There would have
been a contract in the API between the manager and the passes that
first_instance will, if non-NULL, be an instance of the same subclass of
opt_pass that the function returns, so that make_pass_vrp can safely
cast it to the correct opt_pass subclass, and the details of the
opt_pass subclasses can stay encapsulated away inside their
individual .c files.

I chose instead to have a clone method since the type-safety is slightly
easier, and because the GCC plugin API expects to receive passes, rather
than factory functions, so the clone approach means we don't have to touch
that API.

Another idea I had was that the first instance to be created would *embed*
the state, so that the first pas would be in a special subclass::

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

and the others would simply be in the base class::

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

This avoids a "new", but only in the shared library case, and is
complicated, hence we're not doing it this way.

For complicated shared state I considered trying to not put the state*
into the universe by having the states look for them in a designated
location.  For example: if pass_foo is created first, then pass_bar
could share state with it like this::

      opt_pass *make_pass_bar (context &ctxt)
      {
        /* Locate the shared state my hardcoding a reference to a pass
           that already has it: */
        foo_pass *reference_pass = ctxt.pipeline->pass_bar_1;
        gcc_assert (reference_pass);
        foo_state &shared_state = reference_pass->get_shared_state ();
        return new pass_bar (ctxt, shared_state);
      }

But this is clunky.  It's much simpler and less fragile to put a state
pointer into the universe.


Rejected idea: On-stack roots
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
I wrote up some ideas on how to do on-stack GC roots.

For the case where a gc_owner is on the stack, we may want a helper
class::

  class gc_stack_root
  {
  public:
    gc_stack_root(gc_heap &heap, gc_owner& obj) { heap.push_root (obj); }
    ~gc_stack_root() { heap.pop_root (); }
  };

so that you can write::

  #ifdef GLOBAL_STATE
  static pass_state ps;
  #endif

  void
  pass_foo::execute_hook()
  {
  #ifndef GLOBAL_STATE
    pass_state ps;
    gc_stack_root sroot(ctxt_.heap_, ps); // probably have a macro for this
  #endif
    ps.doit();
  }

and have implicit integration of the pass state with the GC in case a
collection happens within the scope.

Alternatively, the state class itself could have the push/pop property::

  #ifdef GLOBAL_STATE
  /* Empty: not used on stack in a global-state build: */
  #define MAYBE_STACK_ROOT
  #else
  /* Inherit from gc_stack_root in a shared-state build: */
  #define MAYBE_STACK_ROOT : public gc_stack_root
  class gc_stack_root : public gc_owner
  {
  public:
    gc_stack_root(gc_heap &heap) { heap.push_root (this); }
    ~gc_stack_root() { heap.pop_root (); }
  };
  #endif

  class GTY((user)) pass_state MAYBE_STACK_ROOT
  {
  };

  #ifdef GLOBAL_STATE
  static pass_state ps;
  #endif

  void
  pass_foo::execute_hook()
  {
  #ifndef GLOBAL_STATE
    pass_state ps(ctxt_.heap); // this implicitly gives you
                               // push/pop registrations of the pass
                               // state with the gc heap.
  #endif
    ps.doit();
  }

We're not doing this.

Rejected idea: storing universe refs in scopes
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
We're not storing universe& in types (like LLVM does), for memory-usage
reasons.

Every type already has a context, from tree.h::

  #define TYPE_CONTEXT(NODE) (TYPE_CHECK (NODE)->type_common.context)

  struct GTY(()) tree_type_common {
     ...
     tree context;
     ...
  };

so one idea I had was that such contexts could gain a universe*, or the
root context could gain one.

For the non-shared case you'd be doing work to access
the universe, then ignoring this - so universe-lookup could be done behind
a macro::

  /* Macro for getting a (universe &) from a type. */
  #if SHARED_BUILD
    #define GET_UNIVERSE(TYPE)  get_universe_from_type((TYPE))
  #else
    /* Access the global singleton: */
    #define GET_UNIVERSE(type)  (the_uni)
  #endif

Rejected, as it involves CPU work and some extra memory; we'll use TLS
instead.
