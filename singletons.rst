Singletons and performance
==========================
The plan calls for *lots* of new classes that will be singletons
in a non-shared-library build.

A concern about generalizing the code to support multiple states is
the increased register pressure of passing a context pointer around
everywhere.

I experimented with various approaches involving heavy use of the
preprocessor, but I'm leaning towards adding a compiler optimization to
handle it.

A singleton-removal optimization
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
The lowest-boilerplate way of optimizing singletons is to create a new
optimization pass for GCC:  optimized handling of singletons that have
been marked as such using a new attribute::

  class __attribute__((singleton("the_foo")) foo
  {
  };

The "singleton" attribute tells the C++ compiler that the struct/class
so-marked will only ever have a single instance, a global variable with
the given decl.  The compiler should issue an error if this contract is
violated.  Note that the_foo might be an instance of a *subclass* of foo.

Then all methods of the marked class lose their implicit "this"
parameters (changing ABI, I know), removing them from all callsites
also.  Instead, this local is implicitly injected into the
implementation of each method call::

   foo *this = &the_foo;

So we'd have something like this::

  class SINGLETON_IN_STATIC_BUILD("the_uni") universe
  {
  };

For a library build where universe instances are dynamically-created, the
macro expands away to whitespace, but for a non-library build, this
would expand to the attribute.

This thus:

  * saves the register pressure of passing around around the this ptr
    everywhere when there's only ever one instance

  * allows devirtualization of vfuncs: we *know* the exact subclass of
    the_foo, so any calls to foo or its subclasses must be the_foo.

  * other optimizations?  (e.g. "exploding" a global struct into global
    vars for its fields)

I think this could be used in quite a few places e.g. for universe, for
the pass manager, for the callgraph, for global_options.

I'm also thinking long-term the various tables of hooks should probably
become C++ objects with vtables, so that we can naturally generalize
them to be singletons in the static-build case, but potentially have
several in the gcc-as-library case.

I have a crude (unposted) patch for this already working in the GCC C++
frontend: it implicitly adds the "static" keyword throughout any classes
that have the singleton attribute.  This gives the saving of register
pressure, but doesn't allow virtual functions.

Note that you can only remove `(foo*)` params from code where you're
guaranteed that all users saw the attribute e.g. from methods of foo and
its subclasses.  You can't remove them from other functions since such
functions might be declared in a translation unit that doesn't see the full
class declaration, and thus doesn't see the attribute.

gengtype may need a little tweaking to help find the GTY markers in
class declarations::

  class GTY((user)) SINGLETON_IN_STATIC_BUILD(the_uni) universe
  {
  }; // class universe

Other ways to optimize singletons
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Another way to mitigate the function->class move is the static-vs-non-static
trick from the tracer.c thread
http://gcc.gnu.org/ml/gcc-patches/2013-05/msg01351.html::


  #if GLOBAL_STATE
  /* When using global state, all methods and fields of state classes
     become "static", so that there is effectively a single global
     instance of the state, and there is no implicit "this->" being passed
     around.  */
  # define MAYBE_STATIC static
  #else
  /* When using on-stack state, all methods and fields of state classes
     lose the "static", so that there can be multiple instances of the
     state with an implicit "this->" everywhere the state is used.  */
  # define MAYBE_STATIC
  #endif

and then using this within a pass to encapsulate state, either as a
singleton, or with multiple instances::

  class tracer_state
  {
  public:
    tracer_state();
  
    MAYBE_STATIC bool tail_duplicate ();
  
  private:
  
    MAYBE_STATIC edge find_best_successor (basic_block);
    MAYBE_STATIC edge find_best_predecessor (basic_block);
    MAYBE_STATIC int find_trace (basic_block, basic_block *);
    MAYBE_STATIC void mark_bb_seen (basic_block bb);
    MAYBE_STATIC bool bb_seen_p (basic_block bb);
  
  private:
  
    /* Minimal outgoing edge probability considered for superblock
       formation.  */
    MAYBE_STATIC int probability_cutoff;
    MAYBE_STATIC int branch_ratio_cutoff;
  
    /* A bit BB->index is set if BB has already been seen, i.e. it is
       connected to some trace already.  */
    MAYBE_STATIC sbitmap bb_seen;

  }; // tracer_state

Hence we can put a tracer_state on the stack in an execute hook, and it
will be empty in a GLOBAL_STATE build, with all the fields being
effectively globals.

Such classes that are local to a source file should be placed into an
anonymous namespace in order to take advantage of target-specific
optimizations that can be done on purely-local functions::

  namespace {

  class tracer_state
  {
     /* etc */
  }; // tracer_state

  } // anon namespace

Alternatively, Richard Henderson identified another pattern in
http://gcc.gnu.org/ml/gcc-patches/2013-05/msg01415.html ::

  namespace {

  class pass_state
  {
    private:
      int x, y, z;

    public:
      constexpr pass_state()
        : x(0), y(0), z(0)
      { }

      void doit();

    private:
      void a();
      void b();
      void c();
  };

  // ...

  } // anon namespace

  #ifdef GLOBAL_STATE
  static pass_state ps;
  #endif

  void execute_hook()
  {
  #ifndef GLOBAL_STATE
    pass_state ps;
  #endif
    ps.doit();
  }

where the compiler's IPA constant propagation sees that the initial "this"
argument is passed a constant value, letting it propagate and eliminate.

Presumably this only works for the case of state that's in one file and
effectively a local.  For state that persists between invocations (and thus
needs references to it stored somewhere), we need another approach (e.g.
the MAYBE_STATIC approach described above).

"constexpr" was introduced in C++11, so presumably we would need to wrap
it in a macro.

Are there other approaches?

FWIW I favor putting extra space between the MAYBE_STATIC and the decl,
breaking things up a little makes it easier for me to read the code::

  class callgraph
  {
  public:
    /* Number of nodes in existence.  */
    MAYBE_STATIC  int n_nodes;

    /* Maximal uid used in cgraph nodes.  */
    MAYBE_STATIC  int node_max_uid;

    /* Maximal uid used in cgraph edges.  */
    MAYBE_STATIC  int edge_max_uid;
  };

vs::

  class callgraph
  {
  public:
    /* Number of nodes in existence.  */
    MAYBE_STATIC int n_nodes;

    /* Maximal uid used in cgraph nodes.  */
    MAYBE_STATIC int node_max_uid;

    /* Maximal uid used in cgraph edges.  */
    MAYBE_STATIC int edge_max_uid;
  };



Elimination of singleton lookups
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Given this code::

   unsigned int
   pass_foo::execute_hook(void)
   {
      /* Get the universe as "this->ctxt_" */
      FILE *dump_file = ctxt_.dump_file_;

where `dump_file_` is a MAYBE_STATIC field of a context, I'm assuming
that in a GLOBAL_STATE build the optimizer can
identify that the `ctxt_` isn't used, and optimize away the lookups
as equivalent to::

   unsigned int
   pass_foo::execute_hook(void)
   {
      context &unused = this->ctxt;
      FILE *dump_file = context::dump_file_;

and simply do::

   unsigned int
   pass_foo::execute_hook(void)
   {
      FILE *dump_file = context::dump_file_;

Similarly, consider chains of singletons, e.g.::

  class context
  {
  public:
    MAYBE_STATIC  callgraph cgraph_;
  };

  class callgraph
  {
  public:
    MAYBE_STATIC  int node_max_uid;
  };

and this statement::

  foo ((/*this->*/ctxt_.cgraph_->node_max_uid);

where `ctxt_` is MAYBE_STATIC, this is effectively::

  context& tmpA = this->ctxt_;
  callgraph *tmpB = tmpA.cgraph_;
  int tmpC = tmpB->node_max_uid;
  foo (tmpC);

and static on the fields in a global state build means that this is::

  context& tmpA = this->ctxt_;
  callgraph *tmpB = context::cgraph_;
  int tmpC = callgraph::node_max_uid;

and thus tmpA and tmpB are unused, so this is effectively just::

  int tmpC = callgraph::node_max_uid;
  foo (tmpC);

Other aspects
^^^^^^^^^^^^^
TODO: experience in gdb for each variant?
TODO: experience in valgrind for each variant?
TODO: what about GC-owned objects and the (lack of) stack roots?

Plan
^^^^
I'm thinking that if the singleton attribute is acceptable we should use it
throughout: it avoids lots of ugly preprocessor hackery.

Otherwise we should use a dual approach:

  * rth's approach for "per-invocation" state

  * the MAYBE_STATIC approach for state that needs to be referenced
    by a pass or by the universe/context object.
