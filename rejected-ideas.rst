Appendix 4: Rejected Ideas
--------------------------

Passes are not yet to be invoked on a specific function
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
I had hoped that the execute callback of passes could gain a `function *`
parameter.  Initially this would be `cfun`, but this would give us a way of
eventually eliminating `cfun`.

Plan: don't do this for this milestone (see notes on cfun on the issues
with this).


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
