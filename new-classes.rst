New Classes
-----------
Here's an idea of some of the new classes we might end up with.

This is a work in progress.

GC classes
^^^^^^^^^^

This class encapsulates the state of the GC-managed heap, so that you can
have more than one of them::

  class gc_heap
  {
     // TODO
  };

This allows us to pass a heap around without having to expose the entire
universe.

The following class is handy for GTY((user)) classes that are in an
inheritance hierarchy::

  class gc_base
  {
  public:
    /* GTY((user)) methods.

       These are pure virtual in the base class to ensure that you
       implement them.

       Subclasses that add extra GC-managed data should overide
       and chain up to their base class' implementation,
       adding walking of their extra fields.  */

    virtual void gt_ggc_mx () = 0;
    virtual void gt_pch_nx () = 0;
    virtual void gt_pch_nx_with_op (gt_pointer_operator op, void *cookie) = 0;
   };

The above virtual functions should only be used for the case where
inheritance is occurring (I'm only using this for pass instances) - the
rest of the time the correct hooks for the types should be invoked
directly, using overloaded `gt_` functions.

There might be a need for base classes:

  * `gc_managed` : allocated in a gc heap, and can own GC-references

  * `gc_owner` : can own GC-references, but not allocated in the gc-heap
    (e.g. universe instances, pass state where there was a GTY, etc)

Frontend Classes
^^^^^^^^^^^^^^^^
These exist in order to encapsulate the various "global_trees" fields::

  class GTY((user)) frontend
  {

  protected:
    frontend(gc_heap& heap);

  protected:
    MAYBE_STATIC gc_heap& heap_;

    MAYBE_STATIC tree global_trees[TI_MAX];
    MAYBE_STATIC tree built_in_attributes[(int) ATTR_LAST];
    MAYBE_STATIC tree builtin_types[(int) BT_LAST + 1];
    MAYBE_STATIC struct line_maps *line_table;

  };

  /* State and code shared between the C, ObjC and C++ frontends.  */
  class GTY((user)) c_family_frontend : public frontend
  {
  protected:
    c_family_frontend(gc_heap& heap);

  protected:
    MAYBE_STATIC tree c_global_trees[CTI_MAX];
  };

  /* An instance of the C++ frontend.  */
  class GTY((user)) cp_frontend : public c_family_frontend
  {
  private:
    MAYBE_STATIC tree cp_global_trees[CPTI_MAX];
  };

Pass classes
^^^^^^^^^^^^

We introduce some helper structs so that various property and todo flags
can be self-documenting; these are synactic sugar for wrapping `unsigned int`::

  struct required;
  struct provided;
  struct destroyed;
  struct start;
  struct finish;

and these for the appropriate bundles of types::

  /* Sets of properties input and output from this pass.  */
  struct pass_properties;

  /* Flags indicating common sets things to do before and after a pass.  */
  struct pass_todo_flags;

so that we can replace this::

  struct gimple_opt_pass pass_vrp =
  {
   {
    GIMPLE_PASS,
    "vrp",                               /* name */
    OPTGROUP_NONE,                       /* optinfo_flags */
    gate_vrp,                            /* gate */
    execute_vrp,                         /* execute */
    NULL,                                /* sub */
    NULL,                                /* next */
    0,                                   /* static_pass_number */
    TV_TREE_VRP,                         /* tv_id */
    PROP_ssa,                            /* properties_required */
    0,                                   /* properties_provided */
    0,                                   /* properties_destroyed */
    0,                                   /* todo_flags_start */
    TODO_cleanup_cfg
      | TODO_update_ssa
      | TODO_verify_ssa
      | TODO_verify_flow                 /* todo_flags_finish */
   }
  };

with::

  class pass_vrp : public gimple_opt_pass
  {
  public:
    pass_vrp(context &ctxt)
      : gimple_opt_pass(ctxt,
                        "vrp",
                        OPTGROUP_NONE,
                        TV_TREE_VRP,
                        pass_properties(required(PROP_ssa),
                                        provided(0),
                                        destroyed(0)),
                        pass_todo_flags(start(0),
                                        finish(TODO_cleanup_cfg
                                               | TODO_update_ssa
                                               | TODO_verify_ssa
                                               | TODO_verify_flow)))
  {}

  /* snip */

without needing comments on the fields.

`struct opt_pass` becomes a base class::

  /* Describe one pass; this is the common part shared across different pass
     types.  */
  class GTY((user)) opt_pass : public gc_base
  {
  public:
    virtual ~opt_pass () { }
  
    /* Public Methods */
  
    /* GTY((user)) methods.
       opt_pass subclasses with additional GC-managed data should overide
       these, chain up to the base class implementation, then walk their
       extra fields.  */
    virtual void gt_ggc_mx ();
    virtual void gt_pch_nx ();
    virtual void gt_pch_nx_with_op (gt_pointer_operator op, void *cookie);
  
    /* Ensure that instances are allocated in the GC-managed heap.  */
    void *operator new (size_t sz);
  
    /* This pass and all sub-passes are executed only if
       the function returns true.  */
    virtual bool has_gate () { return false; }
    virtual bool gate () { return true; }
  
    /* This is the code to run. The return value contains
       TODOs to execute in addition to those in TODO_flags_finish.   */
    virtual bool has_execute () = 0;
    virtual unsigned int impl_execute () = 0;
  
  protected:
    opt_pass(context &ctxt,
             enum opt_pass_type type,
             const char *name,
             unsigned int optinfo_flags,
             timevar_id_t tv_id,
             const pass_properties &props,
             const pass_todo_flags &todo_flags);
  
  /* We should eventually make these fields private: */
  public:
    context &ctxt_;
  
    /* Optimization pass type.  */
    enum opt_pass_type type;
  
    /* Terse name of the pass used as a fragment of the dump file
       name.  If the name starts with a star, no dump happens. */
    const char *name;
  
    /* The -fopt-info optimization group flags as defined in dumpfile.h. */
    unsigned int optinfo_flags;
  
    /* A list of sub-passes to run, dependent on gate predicate.  */
    struct opt_pass *sub;
  
    /* Next in the list of passes to run, independent of gate predicate.  */
    struct opt_pass *next;
  
    /* Static pass number, used as a fragment of the dump file name.  */
    int static_pass_number;
  
    /* The timevar id associated with this pass.  */
    /* ??? Ideally would be dynamically assigned.  */
    timevar_id_t tv_id;
  
    /* Sets of properties input and output from this pass.  */
    unsigned int properties_required;
    unsigned int properties_provided;
    unsigned int properties_destroyed;
  
    /* Flags indicating common sets things to do before and after.  */
    unsigned int todo_flags_start;
    unsigned int todo_flags_finish;
  };
  
  extern void gt_ggc_mx (opt_pass *p);
  extern void gt_pch_nx (opt_pass *p);
  extern void gt_pch_nx (opt_pass *p, gt_pointer_operator op, void *cookie);

There are three simple subclasses that don't add extra fields::

  /* Description of GIMPLE pass.  */
  class gimple_opt_pass : public opt_pass
  {
  public:
    gimple_opt_pass(context &ctxt,
                    const char *name,
                    unsigned int optinfo_flags,
                    timevar_id_t tv_id,
                    const pass_properties &props,
                    const pass_todo_flags &todo_flags)
      : opt_pass(ctxt,
                 GIMPLE_PASS,
                 name,
                 optinfo_flags,
                 tv_id,
                 props,
                 todo_flags)
    {}
  };
  
  /* Description of RTL pass.  */
  class rtl_opt_pass : public opt_pass
  {
  public:
    rtl_opt_pass(context &ctxt,
                 const char *name,
                 unsigned int optinfo_flags,
                 timevar_id_t tv_id,
                 const pass_properties &props,
                 const pass_todo_flags &todo_flags)
      : opt_pass(ctxt,
                 RTL_PASS,
                 name,
                 optinfo_flags,
                 tv_id,
                 props,
                 todo_flags)
    {}
  };
  
  /* Description of simple IPA pass.  Simple IPA passes have just one execute
     hook.  */
  class simple_ipa_opt_pass : public opt_pass
  {
  public:
    simple_ipa_opt_pass(context &ctxt,
                        const char *name,
                        unsigned int optinfo_flags,
                        timevar_id_t tv_id,
                        const pass_properties &props,
                        const pass_todo_flags &todo_flags)
      : opt_pass(ctxt,
                 SIMPLE_IPA_PASS,
                 name,
                 optinfo_flags,
                 tv_id,
                 props,
                 todo_flags)
    {}
  };

The other kind of IPA opt pass is more complicated::

  struct varpool_node;
  struct cgraph_node;
  struct lto_symtab_encoder_d;
  
  /* Description of IPA pass with generate summary, write, execute, read and
     transform stages.  */
  class ipa_opt_pass_d : public opt_pass
  {
  public:
    ipa_opt_pass_d(context &ctxt,
                   const char *name,
                   unsigned int optinfo_flags,
                   timevar_id_t tv_id,
                   const pass_properties &props,
                   const pass_todo_flags &todo_flags,
                   unsigned int function_transform_todo_flags_start)
      : opt_pass(ctxt,
                 IPA_PASS,
                 name,
                 optinfo_flags,
                 tv_id,
                 props,
                 todo_flags),
        function_transform_todo_flags_start(function_transform_todo_flags_start)
    {}
  
    /* IPA passes can analyze function body and variable initializers
        using this hook and produce summary.  */
    virtual bool has_generate_summary () = 0;
    virtual void impl_generate_summary () = 0;
  
    /* This hook is used to serialize IPA summaries on disk.  */
    virtual bool has_write_summary () = 0;
    virtual void impl_write_summary () = 0;
  
    /* This hook is used to deserialize IPA summaries from disk.  */
    virtual bool has_read_summary () = 0;
    virtual void impl_read_summary () = 0;
  
    /* This hook is used to serialize IPA optimization summaries on disk.  */
    virtual bool has_write_optimization_summary () = 0;
    virtual void impl_write_optimization_summary () = 0;
  
    /* This hook is used to deserialize IPA summaries from disk.  */
    virtual bool has_read_optimization_summary () = 0;
    virtual void impl_read_optimization_summary () = 0;
  
    /* Hook to convert gimple stmt uids into true gimple statements.  The second
       parameter is an array of statements indexed by their uid. */
    virtual bool has_stmt_fixup () = 0;
    virtual void impl_stmt_fixup (struct cgraph_node *, gimple *) = 0;
  
    virtual bool has_function_transform () = 0;
    virtual unsigned int impl_function_transform (struct cgraph_node *) = 0;
  
    virtual bool has_variable_transform () = 0;
    virtual void impl_variable_transform (struct varpool_node *) = 0;
  
  /* We should eventually make this field private: */
  public:
    /* Results of interprocedural propagation of an IPA pass is applied to
       function body via this hook.  */
    unsigned int function_transform_todo_flags_start;
  };

Middle-end classes
^^^^^^^^^^^^^^^^^^

Callgraph::

   class GTY((user)) callgraph
   {
   public:
      callgraph(universe &uni);

    /* Public methods: */

    /* In cgraph.c: */
    MAYBE_STATIC  void dump (FILE *) const;
    MAYBE_STATIC  void dump_cgraph_node (FILE *, struct cgraph_node *) const;

    MAYBE_STATIC  void remove_edge (struct cgraph_edge *);

    MAYBE_STATIC  void remove_node (struct cgraph_node *);

    MAYBE_STATIC  struct cgraph_edge *
    create_edge (struct cgraph_node *,
                 struct cgraph_node *,
                 gimple, gcov_type, int);

    /* etc */

    /* In cgraphunit.c: */
    MAYBE_STATIC  void finalize_function (tree, bool);
    MAYBE_STATIC  void finalize_compilation_unit ();
    MAYBE_STATIC  void compile ();
    MAYBE_STATIC  bool process_new_functions ();
    /* etc */

    /* In cgraphclones.c  */
    MAYBE_STATIC  struct cgraph_edge *
    clone_edge (struct cgraph_edge *,
               struct cgraph_node *, gimple,
               unsigned, gcov_type, int, bool);

    MAYBE_STATIC  struct cgraph_node *
    clone_node (struct cgraph_node *, tree, gcov_type,
                int, bool, vec<cgraph_edge_p>,
                bool);
    /* etc */

  private:
    /* Private fields */

    /* Number of nodes in existence.  */
    MAYBE_STATIC  int n_nodes;

    /* Maximal uid used in cgraph nodes.  */
    MAYBE_STATIC  int node_max_uid;

    /* Maximal uid used in cgraph edges.  */
    MAYBE_STATIC  int edge_max_uid;

    /* What state callgraph is in right now.  */
    enum cgraph_state state;

    /* etc */
  };


Backend classes
^^^^^^^^^^^^^^^

TODO; ideas include::

  class backend
  {
  public:
     MAYBE_STATIC rtx const_int_rtx_[MAX_SAVED_CONST_INT * 2 + 1];
     /* with gty hooks in the vfunc */

  };

  class recog
  {
  public:
    MAYBE_STATIC int which_alternative;
    MAYBE_STATIC struct recog_data_d recog_data;
  };


