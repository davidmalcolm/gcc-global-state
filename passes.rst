.. _passes:

Appendix 2: Notes on specific passes
------------------------------------
These notes are based on r199560 (trunk, 2013-05-31)

Various stats on the core passes (not counting those in subdirs):

  * 219 pass structs of which:

    * 117 of `struct gimple_opt_pass`
    * 84 of `struct rtl_opt_pass`
    * 9 of `struct ipa_opt_pass`
    * 9 of `struct simple_ipa_opt_pass`

  * 57 pass structs with a NULL `gate` callback
  * 162 pass structs with a non-NULL `gate` callback
  * 17 pass structs with a NULL `execute` callback (specifically:
    `pass_loop2`, `pass_ipa_lto_gimple_out`, `pass_ipa_lto_finish_out`,
    `pass_all_early_optimizations`, `pass_all_optimizations`,
    `pass_all_optimizations_g`, `pass_rest_of_compilation`,
    `pass_postreload`, `pass_stack_regs`, `pass_tm_init`, the "no-mudflap"
    variants of `pass_mudflap_1` and `pass_mudflap_2`,
    `pass_update_address_taken`, `pass_tree_loop`, `pass_graphite`,
    `pass_build_alias`, `pass_build_ealias`)
  * 202 pass structs with a non-NULL `execute` callback

Locations of other passes:

  * `config/epiphany/mode-switch-use.c`
  * `config/epiphany/resolve-sw-modes.c`
  * `config/i386/i386.c`
  * `config/mips/mips.c`
  * `config/rl78/rl78.c`
  * `config/sparc/sparc.c`

I've grouped the passes by the source file that the struct describing the
pass appears in, since that often (but not always) identifies state that's
shared between passes.

`asan.c`: pass_asan and pass_asan_O0
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Two instances of pass_asan, single instance of pass_asan_O0; they are gimple
passes.

Global variables in `asan.c`::

  alias_set_type asan_shadow_set = -1;
     /* created by asan_init_shadow_ptr_types */

  static GTY(()) tree shadow_ptr_types[2];
     /* created by asan_init_shadow_ptr_types, which is called lazily */

  static alloc_pool asan_mem_ref_alloc_pool;
     /* lazily created by asan_mem_ref_get_alloc_pool
        freed by free_mem_ref_resources */

  static hash_table <asan_mem_ref_hasher> asan_mem_ref_ht;
     /* lazily created by get_mem_ref_hash_table
        freed by free_mem_ref_resources */

  static pretty_printer asan_pp;
  static bool asan_pp_initialized;
    /* these are set up by asan_pp_initialize */

  static GTY(()) tree asan_ctor_statements;
    /* effectively a local within asan_finish_file, but
       exposed to ensure it gets marked in case a GC happens
       inside there */

and `asan.h` exposes::

  /* Alias set for accessing the shadow memory.  */
  extern alias_set_type asan_shadow_set;

However, this doesn't seem to be used except in asan.c; can we make it
static instead?

TODO: make asan_shadow_set static?

Plan: introduce a `class asan_state` shared by the pass instances to hold
the above.


`auto-inc-dec.c`: pass_inc_dec
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single instance of pass_inc_dec

Guts of file are guarded by #ifdef AUTO_INC_DEC

One-time-initialized state::

  static bool initialized = false;
  static enum gen_form decision_table[INC_last][INC_last][FORM_last];

Other state::

  static rtx mem_tmp;

  static rtx *reg_next_use = NULL;
  static rtx *reg_next_inc_use = NULL;
  static rtx *reg_next_def = NULL;

`bb-reorder.c`: pass_reorder_blocks, pass_duplicate_computed_gotos, pass_partition_blocks
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Each of these are single-instanced RTL passes.

State::

  struct target_bb_reorder default_target_bb_reorder;
  #if SWITCHABLE_TARGET
  struct target_bb_reorder *this_target_bb_reorder = &default_target_bb_reorder;
  #endif
     /* used in macro uncond_jump_length */

  static int array_size;
    /* set whenever bbd is set: in copy_bb, reorder_basic_blocks */

  static bbro_basic_block_data *bbd;
    /* allocated in reorder_basic_blocks, resized in copy_bb,
       freed in reorder_basic_blocks */

  static int max_entry_frequency;
    /* set up in find_traces */

  static gcov_type max_entry_count;
    /* set up in find_traces */

`find_traces` is called by `reorder_basic_blocks`, which is called by the
execute hook of `pass_reorder_blocks`.

`duplicate_computed_gotos` uses uncond_jump_length, but none of the rest of
the state

`pass_partition_blocks` doesn't use any of the file's state

TODO

`bt-load.c`: pass_branch_target_load_optimize1, pass_branch_target_load_optimize2
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
"branch target register load optimizations"

Both are single-instanced RTL passes.

State in file::

  static int issue_rate;
     /* set by branch_target_load_optimize; used by migrate_btr_def */

  static struct obstack migrate_btrl_obstack;
     /* inited and freed in migrate_btr_defs */

  static HARD_REG_SET *btrs_live;
     /* allocated and freed in migrate_btr_defs */

  static HARD_REG_SET *btrs_live_at_end;
     /* allocated and freed in migrate_btr_defs */

  static HARD_REG_SET all_btrs;

  static int first_btr, last_btr;
    /* set up by migrate_btr_defs */

  static rtx *btr_reference_found;
    /* set by find_btr_reference; used by find_btr_use */

TODO

`cfgcleanup.c`: pass_jump and pass_jump2
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Both are single-instanced RTL passes.

Both call cleanup_cfg()

File state::

  /* Set to true when we are running first pass of try_optimize_cfg loop.  */
  static bool first_pass;

  /* Set to true if crossjumps occurred in the latest run of try_optimize_cfg.  */
  static bool crossjumps_occured;

  /* Set to true if we couldn't run an optimization due to stale liveness
     information; we should run df_analyze to enable more opportunities.  */
  static bool block_was_dirty;

`cfgexpand.c`: pass_expand
^^^^^^^^^^^^^^^^^^^^^^^^^^
This is the (single-instanced) pass that converts gimple to RTL;
it's an `RTL_PASS`.

Global vars::

  /* This variable holds information helping the rewriting of SSA trees
     into RTL.  */
  struct ssaexpand SA;

  /* This variable holds the currently expanded gimple statement for purposes
     of comminucating the profile info to the builtin expanders.  */
  gimple currently_expanding_gimple_stmt;

  /* We have an array of such objects while deciding allocation.  */
  static struct stack_var *stack_vars;
  static size_t stack_vars_alloc;
  static size_t stack_vars_num;
  static struct pointer_map_t *decl_to_stack_part;

  /* Conflict bitmaps go on this obstack.  This allows us to destroy
     all of them in one big sweep.  */
  static bitmap_obstack stack_var_bitmap_obstack;

  /* An array of indices such that stack_vars[stack_vars_sorted[i]].size
     is non-decreasing.  */
  static size_t *stack_vars_sorted;

  /* The phase of the stack frame.  This is the known misalignment of
     virtual_stack_vars_rtx from PREFERRED_STACK_BOUNDARY.  That is,
     (frame_offset+frame_phase) % PREFERRED_STACK_BOUNDARY == 0.  */
  static int frame_phase;

  /* Used during expand_used_vars to remember if we saw any decls for
     which we'd like to enable stack smashing protection.  */
  static bool has_protected_decls;

  /* Used during expand_used_vars.  Remember if we say a character buffer
     smaller than our cutoff threshold.  Used for -Wstack-protector.  */
  static bool has_short_buffer;

  /* Maps the blocks that do not contain tree labels to rtx labels.  */

  static struct pointer_map_t *lab_rtx_for_bb;

so perhaps a `class expand_state` is required?

TODO: To what extent is state shared between invocations of the pass?
(my guess is "not at all"; if so, can this go on-stack?)

`cfgrtl.c`: pass_free_cfg, pass_into_cfg_layout_mode, pass_outof_cfg_layout_mode
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
All three are single-instance RTL passes.

File state::

  static GTY(()) rtx cfg_layout_function_footer;
  static GTY(()) rtx cfg_layout_function_header;

  struct cfg_hooks rtl_cfg_hooks = {..};
  struct cfg_hooks cfg_layout_rtl_cfg_hooks = {...);



TODO

`cgraphbuild.c`
^^^^^^^^^^^^^^^

* pass_build_cgraph_edges (single, gimple)
* pass_rebuild_cgraph_edges (2 of them, gimple)
* pass_remove_cgraph_callee_edges (3 of them, gimple)

Appears to have no per-file state.


`combine.c`: pass_combine
^^^^^^^^^^^^^^^^^^^^^^^^^^
Single instance of pass_combine (rtl)

File state::

  static int combine_attempts;
    /* zeroed in combine_instructions */

  static int combine_merges;
    /* zeroed in combine_instructions */

  static int combine_extras;
    /* zeroed in combine_instructions */

  /* Number of instructions combined in this function.  */
  static int combine_successes;
    /* zeroed in combine_instructions */

  /* Totals over entire compilation.  */
  static int total_attempts, total_merges, total_extras, total_successes;
    /* these are *not* reinitialized */

  static rtx i2mod;
  static rtx i2mod_old_rhs;
  static rtx i2mod_new_rhs;
  static vec<reg_stat_type> reg_stat;
  static int mem_last_set;
  static int last_call_luid;
  static rtx subst_insn;
  static int subst_low_luid;
  static HARD_REG_SET newpat_used_regs;
  static rtx added_links_insn;
  static basic_block this_basic_block;
  static bool optimize_this_for_speed_p;
  static int max_uid_known;
  static int *uid_insn_cost;
  static struct insn_link **uid_log_links;

  /* Links for LOG_LINKS are allocated from this obstack.  */
  static struct obstack insn_link_obstack;

  static int label_tick;

  /* Reset to label_tick for each extended basic block in scanning order.  */
  static int label_tick_ebb_start;

  static enum machine_mode nonzero_bits_mode;
  static int nonzero_sign_valid;
  static struct undobuf undobuf;
  static int n_occurrences;

  /* Lines 12725- */
  static unsigned int reg_dead_regno, reg_dead_endregno;
  static int reg_dead_flag;




TODO

`combine-stack-adj.c`: pass_stack_adjustments
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single instance of pass_stack_adjustments

Appears to have no state

`compare-elim.c`: pass_compare_elim_after_reload
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single instance of pass_compare_elim_after_reload

State::

  static vec<comparison_struct_p> all_compares;

This appears to be cleaned up after each call, and be built by
`find_comparisons`.

Plan: add a reference to it to struct dom_walk_data, and make it a local
of the execute hook, passed by reference to find_comparison.

`config/epiphany/mode-switch-use.c`: pass_mode_switch_use
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Appears to have no internal (file) state

`config/epiphany/resolve-sw-modes.c`: pass_resolve_sw_modes
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Appears to have no internal (file) state

config/i386/i386.c: pass_insert_vzeroupper
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single-instanced RTL pass, added sometimes for i386 target.

TODO

config/mips/mips.c: pass_mips_machine_reorg2
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single-instanced RTL pass, added sometimes for mips target.

TODO

config/rl78/rl78.c: rl78_devirt_pass
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single-instanced RTL pass, added sometimes for rl78 target.

TODO; execute hook is a call to `rl78_reorg`


config/sparc/sparc.c: pass_work_around_errata
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single-instanced RTL pass, added sometimes for sparc target.

TODO


`cprop.c`: pass_rtl_cprop
^^^^^^^^^^^^^^^^^^^^^^^^^
3 instances of pass_rtl_cprop

File state::

  static struct obstack cprop_obstack;
  static struct hash_table_d set_hash_table;
  static rtx *implicit_sets;
  static int *implicit_set_indexes;
  static regset reg_set_bitmap;
  static int bytes_used;
  static int local_const_prop_count;
  static int local_copy_prop_count;
  static int global_const_prop_count;
  static int global_copy_prop_count;

  /* Lines 545- */
  static sbitmap *cprop_avloc;
  static sbitmap *cprop_kill;
  static sbitmap *cprop_avin;
  static sbitmap *cprop_avout;

  static rtx reg_use_table[MAX_USES];
  static unsigned reg_use_count;

  static int bypass_last_basic_block;

TODO

`cse.c`: pass_cse, pass_cse2, pass_cse_after_global_opts
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

All three are single-instanced rtl passes.

File state::

  static int max_qty;
  static int next_qty;

  static struct qty_table_elem *qty_table;
    /* allocated/freed in cse_extended_basic_block */

  #ifdef HAVE_cc0
  static rtx this_insn_cc0, prev_insn_cc0;
  static enum machine_mode this_insn_cc0_mode, prev_insn_cc0_mode;
  #endif
  static rtx this_insn;
  static bool optimize_this_for_speed_p;
  static struct reg_eqv_elem *reg_eqv_table;
  static struct cse_reg_info *cse_reg_info_table;
  static unsigned int cse_reg_info_table_size;
  static unsigned int cse_reg_info_table_first_uninitialized;
  static unsigned int cse_reg_info_timestamp;
  static HARD_REG_SET hard_regs_in_table;
  static bool cse_cfg_altered;
  static bool cse_jumps_altered;
  static bool recorded_label_ref;
  static int do_not_record;
  static int hash_arg_in_memory;
  static struct table_elt *table[HASH_SIZE];
  static struct table_elt *free_element_chain;
  static int constant_pool_entries_cost;
  static int constant_pool_entries_regcost;
  static bitmap cse_ebb_live_in, cse_ebb_live_out;
  static sbitmap cse_visited_basic_blocks;

TODO

`dce.c`: pass_ud_rtl_dce, pass_fast_rtl_dce
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Both are single-instance rtl passes.

File state::

  static bool df_in_progress = false;
    /* set in run_fast_df_dce */

  static bool can_alter_cfg = false;
    /* set in init_dce */

  static vec<rtx> worklist;
    /* released in rest_of_handle_ud_dce */

  static sbitmap marked;
    /* freed in fini_dce */

  static bitmap_obstack dce_blocks_bitmap_obstack;
    /* released in fini_dce when "fast" */

  static bitmap_obstack dce_tmp_bitmap_obstack;
    /* released in fini_dce when "fast" */

TODO

`df-core.c`: pass_df_initialize_opt, pass_df_initialize_no_opt, pass_df_finish
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

All three are single-instance rtl passes.

File state::

  struct bitmap_obstack reg_obstack;
  bitmap_obstack df_bitmap_obstack;
  struct df_d *df;

  /* Lines 1334- */
  static struct df_problem user_problem;
  static struct dataflow user_dflow;

  /* Line 1720 */
  static int *saved_cfg = NULL;

TODO

`dse.c`
^^^^^^^
"RTL dead store elimination"

* pass_rtl_dse1 (single, rtl)
* pass_rtl_dse2 (appears to not be used, rtl)

File state::

  static bitmap_obstack dse_bitmap_obstack;
    /* released in dse_step7 */

  static struct obstack dse_obstack;
    /* freed in dse_step7 */

  /* Scratch bitmap for cselib's cselib_expand_value_rtx.  */
  static bitmap scratch = NULL;
    /* freed in dse_step7 */

  static alloc_pool cse_store_info_pool;

  static alloc_pool rtx_store_info_pool;
    /* freed in dse_step7 */

  static alloc_pool read_info_pool;
    /* freed in dse_step7 */

  static alloc_pool insn_info_pool;
    /* freed in dse_step7 */

  static insn_info_t active_local_stores;
  static int active_local_stores_len;

  static alloc_pool bb_info_pool;
    /* freed in dse_step7 */

  static bb_info_t *bb_table;
    /* freed in dse_step7 */

  static alloc_pool rtx_group_info_pool;
    /* freed in dse_step7 */

  static int rtx_group_next_id;

  static vec<group_info_t> rtx_group_vec;
    /* released in dse_step7 */

  static alloc_pool deferred_change_pool;
    /* freed in dse_step7 */

  static deferred_change_t deferred_change_list = NULL;
  static group_info_t clear_alias_group;
  static htab_t clear_alias_mode_table;
  static bool stores_off_frame_dead_at_return;

  static int globally_deleted;
    /* inited in dse_step0 */

  static int locally_deleted;
    /* inited in dse_step0 */

  static int spill_deleted;
    /* inited in dse_step0 */

  static bitmap all_blocks;
    /* freed in dse_step7 */

  static bitmap kill_on_calls;
  static unsigned int current_position;

  static hash_table <invariant_group_base_hasher> rtx_group_table;
    /* disposed in dse_step7 */



TODO


`dwarf2cfi.c`: pass_dwarf2_frame
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single instance of pass_dwarf2_frame (rtl)

State::

  static vec<dw_trace_info> trace_info;
  static vec<dw_trace_info_ref> trace_work_list;
  static hash_table <trace_info_hasher> trace_index;

  cfi_vec cie_cfi_vec;
    /* exposed in dwarf2out.h; used in dwarf2out.c and dwarf2cfi.c */

  static GTY(()) dw_cfi_row *cie_cfi_row;
  static GTY(()) reg_saved_in_data *cie_return_save;
  static GTY(()) unsigned long dwarf2out_cfi_label_num;
  static rtx add_cfi_insn;
  static cfi_vec *add_cfi_vec;
  static dw_trace_info *cur_trace;
  static dw_cfi_row *cur_row;
  static dw_cfa_location *cur_cfa;
  static vec<queued_reg_save> queued_reg_saves;
  static bool any_cfis_emitted;
  static unsigned dw_stack_pointer_regnum;
  static unsigned dw_frame_pointer_regnum;

`except.c`
^^^^^^^^^^

* pass_set_nothrow_function_flags (single, rtl)
* pass_convert_to_eh_region_ranges (single, rtl)

File state::

  static GTY(()) int call_site_base;

  static GTY ((param_is (union tree_node)))
    htab_t type_to_runtime_map;
    /* created in init_eh; never freed */

  /* Describe the SjLj_Function_Context structure.  */
  static GTY(()) tree sjlj_fc_type_node;
    /* created in init_eh */

  static int sjlj_fc_call_site_ofs;
    /* set up in init_eh */

  static int sjlj_fc_data_ofs;
    /* set up in init_eh */

  static int sjlj_fc_personality_ofs;
    /* set up in init_eh */

  static int sjlj_fc_lsda_ofs;
    /* set up in init_eh */

  static int sjlj_fc_jbuf_ofs;
    /* set up in init_eh */

  /* Line 1056: */
  static vec<int> sjlj_lp_call_site_index;
    /* safe_grow_cleared/released in sjlj_build_landing_pads */


TODO

`final.c`
^^^^^^^^^

* pass_compute_alignments (single, rtl)
* pass_final (single, rtl)
* pass_shorten_branches (single, rtl)
* pass_clean_state (single, rtl)

File state::

  static rtx debug_insn;
  rtx current_output_insn;
  static int last_linenum;
  static int last_discriminator;
  static int discriminator;
  static int high_block_linenum;
  static int high_function_linenum;
  static const char *last_filename;
  static const char *override_filename;
  static int override_linenum;
  static bool force_source_line = false;
  rtx this_is_asm_operands;
  static unsigned int insn_noperands;
  static rtx last_ignored_compare = 0;
  static int insn_counter = 0;
  #ifdef HAVE_cc0
  CC_STATUS cc_status;
  CC_STATUS cc_prev_status;
  #endif
  static int block_depth;
  static int app_on;

  rtx final_sequence;
    /* exposed in output.h and used in various config subdirs */

  #ifdef ASSEMBLER_DIALECT
  static int dialect_number;
  #endif
  rtx current_insn_predicate;
  bool final_insns_dump_p;
  static bool need_profile_function;
  static int *insn_lengths;

  vec<int> insn_addresses_;
    /* exposed in insn-addr.h and via macros INSN_ADDRESSES etc */

  static int insn_lengths_max_uid;
  int insn_current_address;
  int insn_last_address;
  int insn_current_align;
  static rtx *uid_align;
  static int *uid_shuid;
  static struct label_alignment *label_align;
  static int min_labelno, max_labelno;

  debug_prefix_map *debug_prefix_maps;
    /* made this static in r199963 */




TODO

`function.c`
^^^^^^^^^^^^

* pass_instantiate_virtual_regs (single, rtl)
* pass_leaf_regs (single, rtl)
* pass_thread_prologue_and_epilogue (single, rtl)
* pass_match_asm_constraints (single, rtl)

File state::

  static GTY(()) int funcdef_no;

  struct machine_function * (*init_machine_status) (void);

  struct function *cfun = 0;

  static GTY((if_marked ("ggc_marked_p"), param_is (struct rtx_def)))
    htab_t prologue_insn_hash;
  static GTY((if_marked ("ggc_marked_p"), param_is (struct rtx_def)))
    htab_t epilogue_insn_hash;

  htab_t types_used_by_vars_hash = NULL;
  vec<tree, va_gc> *types_used_by_cur_var_decl;

  static vec<function_p> function_context_stack;

  static GTY((param_is(struct temp_slot_address_entry))) htab_t temp_slot_address_table;
  static size_t n_temp_slots_in_use;

  /* Lines 1344- */
  static int in_arg_offset;
  static int var_offset;
  static int dynamic_offset;
  static int out_arg_offset;
  static int cfa_offset;

  /* Line 4329: */
  static GTY(()) int next_block_index = 2;

  static bool in_dummy_function;

  static vec<function_p> cfun_stack;

  static GTY(()) rtx initial_trampoline;
    /* unused; removed this in r199962. */


TODO

`fwprop.c`: pass_rtl_fwprop, pass_rtl_fwprop_addr
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Both are single-instanced rtl passes

Both passes call `fwprop_init` / `fwprop_done`

File state::

  static int num_changes;
    /* zeroed by fwprop_init */

  static vec<df_ref> use_def_ref;
    /* created by build_single_def_use_links */
       released by fwprop_done */

  static vec<df_ref> reg_defs;
     /* created/released by build_single_def_use_links */

  static vec<df_ref> reg_defs_stack;
     /* created/released by build_single_def_use_links */

  static bitmap local_md;
    /* allocated/freed by build_single_def_use_links */

  static bitmap local_lr;
    /* allocated/freed by build_single_def_use_links */

  static df_ref *active_defs;
    /* allocated by fwprop_init;
       freed by fwprop_done */

  #ifdef ENABLE_CHECKING
  static sparseset active_defs_check;
  #endif
    /* allocated by fwprop_init;
       freed by fwprop_done */

Appears to be essentially per-invocation state.


`gcse.c`: pass_rtl_pre, pass_rtl_hoist
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Both are single-instanced rtl passes

File state::

  struct target_gcse default_target_gcse;
  #if SWITCHABLE_TARGET
  struct target_gcse *this_target_gcse = &default_target_gcse;
  #endif
  int flag_rerun_cse_after_global_opts;
  static struct obstack gcse_obstack;
  static struct hash_table_d expr_hash_table;
  static struct ls_expr * pre_ldst_mems = NULL;
  static hash_table <pre_ldst_expr_hasher> pre_ldst_table;

  static regset reg_set_bitmap;
    /* allocated by alloc_gcse_mem;
       freed by free_gcse_mem */

  static vec<rtx> *modify_mem_list;
    /* allocated by alloc_gcse_mem;
       freed by free_modify_mem_tables */

  static bitmap modify_mem_list_set;
    /* allocated by alloc_gcse_mem;
       freed by free_gcse_mem */

  static vec<modify_pair> *canon_modify_mem_list;
    /* allocated by alloc_gcse_mem;
       freed by free_modify_mem_tables */

  static bitmap blocks_with_calls;
    /* allocated by alloc_gcse_mem;
       freed by free_gcse_mem */

  static int bytes_used;
    /* zeroed in one_pre_gcse_pass */

  static int gcse_subst_count;
    /* zeroed in one_pre_gcse_pass */

  static int gcse_create_count;
    /* zeroed in one_pre_gcse_pass */

  static bool doing_code_hoisting_p = false;

  static sbitmap *ae_kill;
    /* allocated by alloc_pre_mem */

  static basic_block curr_bb;
  static int curr_reg_pressure[N_REG_CLASSES];

  /* Lines 751- */
  static struct reg_avail_info *reg_avail_info;
  static basic_block current_bb;

  static GTY(()) rtx test_insn;

  /* Lines 1785- */
  static sbitmap *transp;
    /* allocated by alloc_pre_mem;
       freed by free_pre_mem */

  static sbitmap *comp;
    /* allocated by alloc_pre_mem;
       freed by free_pre_mem */

  static sbitmap *antloc;
    /* allocated by alloc_pre_mem */

  static sbitmap *pre_optimal;
    /* zeroed by alloc_pre_mem;
       freed by free_pre_mem */

  static sbitmap *pre_redundant;
    /* zeroed by alloc_pre_mem;
       freed by free_pre_mem */

  static sbitmap *pre_insert_map;
    /* zeroed by alloc_pre_mem;
       freed by free_pre_mem */

  static sbitmap *pre_delete_map;
    /* zeroed by alloc_pre_mem;
       freed by free_pre_mem */

  /* Lines 2769- */
  static sbitmap *hoist_vbein;
  static sbitmap *hoist_vbeout;


Ideas:

  * `class gcse_mem` for alloc_gcse_mem, free_gcse_mem
  * `class pre_state`

TODO


`gimple-low.c`: pass_lower_cf
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single instance of pass_lower_cf

Appears to have no global state; a `(struct lower_data *)` is passed
around.

`gimple-ssa-strength-reduction.c`: pass_strength_reduction
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single-instanced gimple pass.

File state::

  static vec<slsr_cand_t> cand_vec;
    /* created in execute_strength_reduction */

  static struct pointer_map_t *stmt_cand_map;
    /* created in execute_strength_reduction */

  static struct obstack cand_obstack;
    /* inited in execute_strength_reduction */

  static struct obstack chain_obstack;
    /* inited in execute_strength_reduction */

  static incr_info_t incr_vec;
    /* allocated/freed in analyze_candidates_and_replace */

  static unsigned incr_vec_len;
    /* zeroed in analyze_candidates_and_replace */

  static bool address_arithmetic_p;
    /* inited in analyze_candidates_and_replace */

  static hash_table <cand_chain_hasher> base_cand_map;
    /* created in execute_strength_reduction */

TODO


`ifcvt.c`
^^^^^^^^^
* pass_rtl_ifcvt (single, rtl)
* pass_if_after_combine (single, rtl)
* pass_if_after_reload (single, rtl)

The execute hooks of all three passes call into if_convert.

File state::

  static int num_possible_if_blocks;
     /* zeroed in if_convert;
        dumped at end of if_convert */

  static int num_updated_if_blocks;
     /* zeroed in if_convert;
        dumped at end of if_convert */

  static int num_true_changes;
     /* zeroed in if_convert;
        dumped at end of if_convert */

  static int cond_exec_changed_p;
     /* inited in do {...} while loop in if_convert */

This all appears to effectively be local state with lifetime scoped by calls
to if_convert.

Plan: introduce a `class if_convert_state`


`init-regs.c`: pass_initialize_regs
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single instance of pass_initialize_regs

Appears to have no global state.

`ipa.c`
^^^^^^^
* pass_ipa_function_and_variable_visibility (single)
* pass_ipa_free_inline_summary (single)
* pass_ipa_whole_program_visibility (single)
* pass_ipa_profile (single)
* pass_ipa_cdtor_merge (single)

File state::

  /* Lines 1068- */
  vec<histogram_entry *> histogram;
    /* released by ipa_profile */

  static alloc_pool histogram_pool;
    /* created by ipa_profile_generate_summary
       and by ipa_profile_read_summary;
       freed by ipa_profile */

  /* Lines 1518- */
  static vec<tree> static_ctors;
    /* released by ipa_cdtor_merge */

  static vec<tree> static_dtors;
    /* released by ipa_cdtor_merge */

TODO

`ipa-cp.c`: pass_ipa_cp (single)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

File state::

  alloc_pool ipcp_values_pool;
  alloc_pool ipcp_sources_pool;
  alloc_pool ipcp_agg_lattice_pool;
  static gcov_type max_count;
  static long overall_size, max_new_size;
  static struct ipcp_value *values_topo;

  /* Line 2322: */
  static vec<cgraph_edge_p> next_edge_clone;


`ipa-inline-analysis.c`: pass_inline_parameters
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
2 instances of this pass

File state::

  static struct cgraph_node_hook_list *function_insertion_hook_holder;
  static struct cgraph_node_hook_list *node_removal_hook_holder;
  static struct cgraph_2node_hook_list *node_duplication_hook_holder;
  static struct cgraph_2edge_hook_list *edge_duplication_hook_holder;
  static struct cgraph_edge_hook_list *edge_removal_hook_holder;

  vec<inline_summary_t, va_gc> *inline_summary_vec;
  vec<inline_edge_summary_t> inline_edge_summary_vec;

  vec<int> node_growth_cache;
  vec<edge_growth_cache_entry> edge_growth_cache;

  static alloc_pool edge_predicate_pool;



TODO


`ipa-inline.c`: pass_early_inline, pass_ipa_inline
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Both are single-instanced

File state::

  static int overall_size;
  static gcov_type max_count;

TODO


`ipa-pure-const.c`
^^^^^^^^^^^^^^^^^^
* pass_ipa_pure_const (single, ipa_opt_pass_d)
* pass_local_pure_const (3 of these, gimple)

File state::

  static struct pointer_set_t *visited_nodes;
  static struct funct_state_d varying_state;
  static vec<funct_state> funct_state_vec;
  static struct cgraph_node_hook_list *function_insertion_hook_holder;
  static struct cgraph_2node_hook_list *node_duplication_hook_holder;
  static struct cgraph_node_hook_list *node_removal_hook_holder;

TODO


`ipa-reference.c`: pass_ipa_reference
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single-instanced ipa_opt_pass_d

File state::

  static splay_tree reference_vars_to_consider;
  static bitmap all_module_statics;
  static bitmap_obstack local_info_obstack;
  static bitmap_obstack optimization_summary_obstack;
  static struct cgraph_2node_hook_list *node_duplication_hook_holder;
  static struct cgraph_node_hook_list *node_removal_hook_holder;
  static vec<ipa_reference_vars_info_t> ipa_reference_vars_vector;
  static vec<ipa_reference_optimization_summary_t> ipa_reference_opt_sum_vector;

TODO


`ipa-split.c`
^^^^^^^^^^^^^
* pass_split_functions (single, gimple)
* pass_feedback_split_functions (single, gimple)

File state::

  static vec<bb_info> bb_info_vec;
    /* released at end of execute_split_functions */

  struct split_point best_split_point;
    /* zeroed within execute_split_functions */

  static bitmap forbidden_dominators;
    /* allocated/freed within execute_split_functions */

TODO

`ira.c`
^^^^^^^
* pass_ira (single)
* pass_reload (single)

TODO

`jump.c`: pass_cleanup_barriers
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
(single, rtl)

Appears to have no per-file state.


`loop-init.c`
^^^^^^^^^^^^^
* pass_loop2 (single, rtl)
* pass_rtl_loop_init (single, rtl)
* pass_rtl_loop_done (single, rtl)
* pass_rtl_move_loop_invariants (single, rtl)
* pass_rtl_unswitch (single, rtl)
* pass_rtl_unroll_and_peel_loops (single, rtl)
* pass_rtl_doloop (single, rtl)

Appears to have no per-file state.


`lower-subreg.c`
^^^^^^^^^^^^^^^^
* pass_lower_subreg (single, rtl)
* pass_lower_subreg2 (single, rtl)

File state::

  static bitmap decomposable_context;
    /* allocated/freed in decompose_multiword_subregs */

  static bitmap non_decomposable_context;
    /* allocated/freed in decompose_multiword_subregs */

  static bitmap subreg_context;
    /* allocated/freed in decompose_multiword_subregs */

  static vec<bitmap> reg_copy_graph;
    /* created/released in decompose_multiword_subregs */

  struct target_lower_subreg default_target_lower_subreg;
  #if SWITCHABLE_TARGET
  struct target_lower_subreg *this_target_lower_subreg
    = &default_target_lower_subreg;
  #endif
     /* memset to zero in init_lower_subreg */

TODO


`lto-streamer-out.c`
^^^^^^^^^^^^^^^^^^^^
* pass_ipa_lto_gimple_out (single)
* pass_ipa_lto_finish_out (single)

Appears to have no per-file state.


`mode-switching.c`: pass_mode_switching
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
There is normally only a single instance of pass_mode_switching; however
`epiphany.c` adds a target-specific second instance in `epiphany_init`.

Guarded by #ifdef OPTIMIZE_MODE_SWITCHING

File state::

  /* These bitmaps are used for the LCM algorithm.  */
  static sbitmap *antic;
  static sbitmap *transp;
  static sbitmap *comp;

These appear to be created/destroyed at start/end of the execute hook;
hence per-invocation state.


`modulo-sched.c`: pass_sms
^^^^^^^^^^^^^^^^^^^^^^^^^^
Single instance of pass_sms

Guarded by #ifdef INSN_SCHEDULING

File state::

  static struct common_sched_info_def sms_common_sched_info;
  static struct sched_deps_info_def sms_sched_deps_info;
  static struct haifa_sched_info sms_sched_info;

TODO: interaction with other scheduler code?

`omp-low.c`: pass_expand_omp, pass_lower_omp, pass_diagnose_omp_blocks
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single instance of each of these three passes.

Globals within this file::

  static splay_tree all_contexts;
     /* created/destroyed by execute_lower_omp, the execute hook of
        pass_lower_omp */
  static int taskreg_nesting_level;
  struct omp_region *root_omp_region;
  static bitmap task_shared_vars;

  static GTY(()) unsigned int tmp_ompfn_id_num;

  static GTY((param1_is (tree), param2_is (tree)))
    splay_tree critical_name_mutexes;

  static splay_tree all_labels;
    /* lifetime is bounded by diagnose_omp_structured_block_errors */

so perhaps a `class omp_low_state`, or perhaps break things up a bit?

`passes.c`
^^^^^^^^^^
* pass_early_local_passes (single)
* pass_all_early_optimizations (single)
* pass_all_optimizations (single)
* pass_all_optimizations_g (single)
* pass_rest_of_compilation (single)
* pass_postreload (single)

TODO

`postreload.c`: pass_postreload_cse
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single instance of pass_postreload_cse

Global state occurs most of the way down the file::

  static int reg_set_luid[FIRST_PSEUDO_REGISTER];
  static HOST_WIDE_INT reg_offset[FIRST_PSEUDO_REGISTER];
  static int reg_base_reg[FIRST_PSEUDO_REGISTER];
  static rtx reg_symbol_ref[FIRST_PSEUDO_REGISTER];
  static enum machine_mode reg_mode[FIRST_PSEUDO_REGISTER];
  static int move2add_luid;
  static int move2add_last_label_luid;

`postreload-gcse.c`: pass_gcse2
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single instance of pass_gcse2

File state::

  static struct {...} stats;
  static hash_table <expr_hasher> expr_table;
  static struct obstack expr_obstack;
  static struct obstack occr_obstack;
  static struct obstack unoccr_obstack;
  static int *reg_avail_info;
  static struct modifies_mem *modifies_mem_list;
  static struct obstack modifies_mem_obstack;
  static struct modifies_mem  *modifies_mem_obstack_bottom;
  static int *uid_cuid;

There's an alloc_mem/free_mem pair for much of this

`predict.c`
^^^^^^^^^^^
* pass_profile (single, gimple)
* pass_strip_predict_hints (2 of these, gimple)

File state::

  static sreal real_zero, real_one, real_almost_one, real_br_prob_base,
       real_inv_br_prob_base, real_one_half, real_bb_freq_max;
  static int real_values_initialized;

  static gcov_type min_count = -1;

  static struct pointer_map_t *bb_predictions;
     /* created/destroyed in tree_estimate_probability */

TODO

`recog.c`
^^^^^^^^^
* pass_peephole2 (single, rtl)
* pass_split_all_insns (normally single-instanced, but epiphany adds a
  second instance, rtl)
* pass_split_after_reload (single, rtl)
* pass_split_before_regstack (single, rtl)
* pass_split_before_sched2 (single, rtl)
* pass_split_for_shorten_branches (single, rtl)

TODO

`ree.c`: pass_ree
^^^^^^^^^^^^^^^^^
Single instance of pass_ree ("Redundant Zero-Extension Elimination")

State::

  static int max_insn_uid;
    /* set near the top of find_and_remove_re, which is called at the top
       of the execute hook */

Plan: introduce a state class and have a local of it in the execute hook.

`reg-stack.c`
^^^^^^^^^^^^^
* pass_stack_regs (single, rtl)
* pass_stack_regs_run (single, rtl)

File state::

  static vec<char> stack_regs_mentioned_data;
    /* created near end of reg_to_stack; released at top of reg_to_stack */

  static basic_block current_block;
  static bool starting_stack_p;
  static rtx
    FP_mode_reg[LAST_STACK_REG+1-FIRST_STACK_REG][(int) MAX_MACHINE_MODE];
  static rtx not_a_num;
  static rtx ix86_flags_rtx;
  static bool any_malformed_asm;

TODO


`regcprop.c`: pass_cprop_hardreg
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single instance of pass_cprop_hardreg

File state::

  static alloc_pool debug_insn_changes_pool;

This seems to only have lifetime within the call to
`copyprop_hardreg_forward`, the execute hook: per-invocation state.

`reginfo.c`: pass_reginfo_init
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single instance of pass_reginfo_init

Global state: many vars, not all of them static

`regmove.c`: pass_regmove
^^^^^^^^^^^^^^^^^^^^^^^^^
Single instance of pass_regmove

Global state (scattered through source file)::

  static int *regno_src_regno;
    /* allocated/freed in regmove_optimize */

  static basic_block *reg_set_in_bb;
    /* allocated first time reg_is_remote_constant_p is called;
       freed in regmove_optimize */

  static unsigned int max_reg_computed;
    /* set first time reg_is_remote_constant_p is called;
       and subsequently after call to regmove_optimize due to
         if (!reg_set_in_bb) */

Hence this appears to be per-invocation state.


`reorg.c`
^^^^^^^^^
* pass_delay_slots (single, rtl)
* pass_machine_reorg (single, rtl)

File state::

  static struct obstack unfilled_slots_obstack;
  static rtx *unfilled_firstobj;
  static rtx function_return_label;
  static rtx function_simple_return_label;

  static int *uid_to_ruid;
    /* allocated/freed in dbr_schedule */

  static int max_uid;
    /* set up in dbr_schedule */

  /* Lines 712- */
  static int num_insns_needing_delays[NUM_REORG_FUNCTIONS][MAX_REORG_PASSES];
  static int num_filled_delays[NUM_REORG_FUNCTIONS][MAX_DELAY_HISTOGRAM+1][MAX_REORG_PASSES];
  static int reorg_pass_number;


TODO


`regrename.c`: pass_regrename
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single instance of pass_regrename

About a dozen static vars.

A non-static var::

  /* If nonnull, the code calling into the register renamer requested
     information about insn operands, and we store it here.  */
  vec<insn_rr_info> insn_rr;

exposed in regrename.h::

  extern vec<insn_rr_info> insn_rr;

and used by config/c6x/c6x.c

`sched-rgn.c`
^^^^^^^^^^^^^
* pass_sched (single, rtl)
* pass_sched2 (single, rtl)

File state::

  static int nr_inter, nr_spec;

  static int is_cfg_nonregular (void);

  int nr_regions = 0;
    /* exposed in sched-int.h */

  region *rgn_table = NULL;
    /* exposed in sched-int.h */

  int *rgn_bb_table = NULL;
     /* exposed in sched-int.h */

  int *block_to_bb = NULL;
     /* exposed in sched-int.h */

  int *containing_rgn = NULL;
    /* exposed in sched-int.h */

  int *ebb_head = NULL;
    /* exposed in sched-int.h */

  static int min_spec_prob;

  int current_nr_blocks;
    /* exposed in sched-int.h */

  int current_blocks;
    /* exposed in sched-int.h */

  static basic_block *bblst_table;
  static int bblst_size, bblst_last;
  static char *bb_state_array = NULL;
  static state_t *bb_state = NULL;
  static edge *edgelst_table;
  static int edgelst_last;
  static sbitmap *dom;
    /* used by IS_DOMINATED */
  static int *prob;
  static int rgn_nr_edges;
  static edge *rgn_edges;
  static edgeset *pot_split;
  static edgeset *ancestor_edges;

  /* Lines 2059- */
  static int sched_target_n_insns;
  static int target_n_insns;
  static int sched_n_insns;

  /* Lines 2326- */
  static struct common_sched_info_def rgn_common_sched_info;
  static struct sched_deps_info_def rgn_sched_deps_info;
  static struct haifa_sched_info rgn_sched_info;

  /* Line 2568: */
  static struct deps_desc *bb_deps;

Idea:  introduce a `class sched` or somesuch to the backend class, as
MAYBE_STATIC.


TODO


`stack-ptr-mod.c`: pass_stack_ptr_mod
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single instance of pass_stack_ptr_mod

No global state.

`store-motion.c`: pass_rtl_store_motion
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single instance of pass_rtl_store_motion

Global state, apparently all static::

  static struct st_expr * store_motion_mems = NULL;
  static sbitmap *st_kill, *st_avloc, *st_antloc, *st_transp;
  static sbitmap *st_insert_map;
  static sbitmap *st_delete_map;
  static int num_stores;
  static struct edge_list *edge_list;
  static hash_table <st_expr_hasher> store_motion_mems_table;

`free_store_memory()` clears much(all?) of this;
`build_store_vectors` seems to be the counterpart.

`tracer.c`: pass_tracer
^^^^^^^^^^^^^^^^^^^^^^^
Single instance of pass_tracer

File state::

  static int probability_cutoff;
  static int branch_ratio_cutoff;
  sbitmap bb_seen;

These state variables are all per-invocation of the pass.

Patch posted as http://gcc.gnu.org/ml/gcc-patches/2013-05/msg01318.html


`trans-mem.c`
^^^^^^^^^^^^^
* pass_diagnose_tm_blocks (single, gimple)
* pass_lower_tm (single, gimple)
* pass_tm_init (single, gimple)
* pass_tm_mark (single, gimple)
* pass_tm_edges (single, gimple)
* pass_tm_memopt (single, gimple)
* pass_ipa_tm (single, simple-ipa)

File state::

  static hash_table <log_entry_hasher> tm_log;
     /* created by tm_log_init;
        disposed by tm_log_delete */

  static vec<tree> tm_log_save_addresses;
     /* created by tm_log_init;
        released by tm_log_delete */

  static hash_table <tm_mem_map_hasher> tm_new_mem_hash;
     /* created by tm_log_init;
        disposed by tm_log_delete */

  bool pending_edge_inserts_p;
     /* initialized by execute_tm_mark */

  static struct tm_region *all_tm_regions;
  static bitmap_obstack tm_obstack;

  static bitmap_obstack tm_memopt_obstack;

  static unsigned int tm_memopt_value_id;
  static hash_table <tm_memop_hasher> tm_memopt_value_numbers;





TODO


`tree.c`: pass_ipa_free_lang_data
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Single instance of pass_ipa_free_lang_data

Doesn't appear to have state itself so much as have responsibility for
poking at globals::

  /* Free resources that are used by FE but are not needed once they
     are done. */

`tree-call-cdce.c`: pass_call_cdce
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
"Conditional Dead Call Elimination"
Single-instanced gimple pass.

File appears to have no state.


`tree-cfg.c`
^^^^^^^^^^^^
* pass_build_cfg (single)
* pass_split_crit_edges (single)
* pass_warn_function_return (single)
* pass_warn_function_noreturn (single)
* pass_warn_unused_result (single)

File state::

  /*
   "Right now this table is set up and torn down at key points in the
    compilation process.  It would be nice if we could make the table
    more persistent.  The key is getting notification of changes to
    the CFG (particularly edge removal, creation and redirection)."  */
  static struct pointer_map_t *edge_to_cases;
    /* created by start_recording_case_labels;
       destroyed by end_recording_case_labels */

  static bitmap touched_switch_bbs;
  /* allocated by start_recording_case_labels;
     freed by end_recording_case_labels */

  static struct cfg_stats_d cfg_stats;
     /* cleared by build_gimple_cfg */

  static bool found_computed_goto;
    /* cleared by build_gimple_cfg */

  static hash_table <locus_discrim_hasher> discriminator_per_locus;
    /* created/disposed during one phase of build_gimple_cfg */

  static struct label_record {...} *label_for_bb;
    /* allocated/free at top/bottom of cleanup_dead_labels;
       used there and by main_block_label

       main_block_label used by cleanup_dead_labels_eh and
       cleanup_dead_labels
       */

TODO


`tree-cfgcleanup.c`: pass_merge_phi
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
(2 of these)

File state::

   bitmap cfgcleanup_altered_bbs;
      /* exposed in tree-flow.h and used by tree-cfg.c */

Plan: introduce a `class cfg_cleanup_state` and add it to context, with
MAYBE_STATIC.


`tree-complex.c`
^^^^^^^^^^^^^^^^
* pass_lower_complex (2 of these)
* pass_lower_complex_O0 (single)

They share `tree_lower_complex` as the execute hook.

File state::

  static vec<complex_lattice_t> complex_lattice_values;
     /* released at end of tree_lower_complex */

  static int_tree_htab_type complex_variable_components;
     /* disposed at end of tree_lower_complex */

  static vec<tree> complex_ssa_name_components;
     /* released at end of tree_lower_complex */

Given that, this looks like per-invocation state.

TODO


`tree-eh.c`
^^^^^^^^^^^
* pass_lower_eh (single)
* pass_refactor_eh (single)
* pass_lower_resx (single)
* pass_lower_eh_dispatch (single)
* pass_cleanup_eh (2 of these)

File state::

  static int using_eh_for_cleanups_p = 0;
  static hash_table <finally_tree_hasher> finally_tree;
  static gimple_seq eh_seq;
  static bitmap eh_region_may_contain_throw_map;

State that's passed around::

  struct leh_state *state
  struct leh_tf_state *tf

TODO


`tree-emutls.c`: pass_ipa_lower_emutls
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single-instanced simple-ipa pass

File state::

  static varpool_node_set tls_vars;
    /* created/freed at start/end of ipa_lower_emutls */

  static vec<varpool_node_ptr> control_vars;
    /* created/freed at start/end of ipa_lower_emutls */

  static vec<tree> access_vars;
    /* created/freed at start/end of ipa_lower_emutls */

  static tree emutls_object_type;
    /* created the first time get_emutls_object_type is called */

This is mostly per-invocation state, but the emutls_object_type
would be shared between invocations.  That said, this is a simple-ipa
pass, so AIUI it's only invoked once.


`tree-if-conv.c`: pass_if_conversion
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single instance of pass_if_conversion (gimple)

File state::

  static basic_block *ifc_bbs;

Appears to be cleaned up at the end of `tree_if_conversion`, hence this
appears to be a subset-of-per-invocation state.


`tree-into-ssa.c`: pass_build_ssa
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single-instanced gimple pass.

File state::

  static vec<tree> block_defs_stack;
     /* created/released within rewrite_blocks */

  static sbitmap old_ssa_names;
     /* allocated in init_update_ssa;
        freed in delete_update_ssa */

  static sbitmap new_ssa_names;
     /* allocated in init_update_ssa;
        freed in delete_update_ssa */

  sbitmap interesting_blocks;

  static bitmap names_to_release;
     /* initialized in init_update_ssa;
        freed in delete_update_ssa */

  static vec<gimple_vec> phis_to_rewrite;
    /* created in update_ssa
       appears to never be released
       appears to be resized as necessary */

  static bitmap blocks_with_phis_to_rewrite;
    /* freed in delete_update_ssa */

  static struct function *update_ssa_initialized_fn = NULL;
     /* initialized in init_update_ssa */

  static hash_table <var_info_hasher> var_infos;
  static vec<ssa_name_info_p> info_for_ssa_name;
  static unsigned current_info_for_ssa_name_age;

  static bitmap_obstack update_ssa_obstack;
     /* initialized in init_update_ssa */

  static bitmap blocks_to_update;
    /* freed in delete_update_ssa */

  static bitmap symbols_to_rename_set;
    /* freed in delete_update_ssa */

  static vec<tree> symbols_to_rename;

Done: made interesting_blocks static as r199911.

TODO: I'm guessing that this is all per-invocation state, but I should
finish the above analysis.

`var_infos` appears to be somewhat exposed via static `get_var_info` called
by static `get_common_info` called by `get_current_def` and
`set_current_def` which are called in 9 places in `tree-vect-loop-manip.c`.


`tree-loop-distribution.c`: pass_loop_distribution
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single-instanced gimple pass.

State::

  static bitmap remaining_stmts;
    /* allocated and freed in ldist_gen */

  static bitmap upstream_mem_writes;
    /* allocated and freed in ldist_gen */

Both are allocated and freed in `ldist_gen`, which is called by
`distribute_loop`, called in turn by `tree_loop_distribution`, the
execute hook.

Hence this is effectively per-invocation state

Plan: introduce a `class ldist_gen_state`, introduce a local
variable instance of it to `ldist_gen`, as a global in a GLOBAL_STATE
build.


`tree-mudflap.c` / `tree-nomudflap.c`
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
`pass_mudflap_1` and `pass_mudflap_2` are both single-instanced gimple passes.

The "real" implementation is in `tree-mudflap.c`; `tree-nomudflap.c` provides
an empty implementation for frontends that can't handle it (e.g. Fortran).

State in tree-mudflap.c consists of a number of tree nodes of the form::

  static GTY (()) tree mf_uintptr_type;

These appear to all be set up by `mudflap_init`, which has this internal state::

  static bool done = false;

There are also these::

  static GTY ((param_is (union tree_node))) htab_t marked_trees = NULL;

  static GTY (()) vec<tree, va_gc> *deferred_static_decls;

  static GTY (()) tree enqueued_call_stmt_chain;


mudflap has an external API; for example, this is called by `toplev.c`::

      if (flag_mudflap)
	mudflap_finish_file ();

hence we need to preserve state outside of the passes.

Plan: Create a `class mudflap_state` to hold the state within tree-mudflap.c
and move the variables into it as MAYBE_STATIC fields.  Have an instance
owned by the context.


`tree-nrv.c`
^^^^^^^^^^^^
Gimple passes language-independent return value optimizations):

  * pass_nrv (single)
  * pass_return_slot (single)

Appears to have no per-file state.


`tree-object-size.c`: pass_object_sizes
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
2 instances of pass_object_sizes, a gimple pass.

State::

   static unsigned HOST_WIDE_INT unknown[4] = { -1, -1, 0, 0 };
     /* made const as r199832. */

   static unsigned HOST_WIDE_INT *object_sizes[4];
      /* allocated first time into init_object_sizes;
         freed in fini_object_sizes */

  static bitmap computed[4];
      /* allocated first time into init_object_sizes;
         freed in fini_object_sizes */

   static unsigned HOST_WIDE_INT offset_limit;
      /* set up in init_offset_limit, called by init_object_sizes */

Given that `fini_object_sizes` is called at the end of the execute hook,
this appears to be per-invocation state.

tree.h exposes this API::

  /* In tree-object-size.c.  */
  extern void init_object_sizes (void);
  extern void fini_object_sizes (void);
  extern unsigned HOST_WIDE_INT compute_builtin_object_size (tree, int);

However, the only one that's called outside of `tree-object-size.c` is
`compute_builtin_object_size`, and that's only called by
`fold_builtin_object_size` in `builtins.c`.  This exists to handle::

  static tree
  fold_builtin_2 (location_t loc, tree fndecl, tree arg0, tree arg1, bool ignore)
  {
    tree type = TREE_TYPE (TREE_TYPE (fndecl));
    enum built_in_function fcode = DECL_FUNCTION_CODE (fndecl);

    switch (fcode)
      {
      /*...*/
      case BUILT_IN_OBJECT_SIZE:

in `fold_builtin_2`, used in `fold_builtin_n` to fold builtins with 2 args



TODO:

`tree-optimize.c`
^^^^^^^^^^^^^^^^^
Gimple passes:

  * pass_cleanup_cfg_post_optimizing (single)
  * pass_fixup_cfg (2 of these)

Appears to have no per-file state.


`tree-profile.c`: pass_ipa_tree_profile
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single-instanced simple_ipa pass.

File state::

  static GTY(()) tree gcov_type_node;
  static GTY(()) tree tree_interval_profiler_fn;
  static GTY(()) tree tree_pow2_profiler_fn;
  static GTY(()) tree tree_one_value_profiler_fn;
  static GTY(()) tree tree_indirect_call_profiler_fn;
  static GTY(()) tree tree_average_profiler_fn;
  static GTY(()) tree tree_ior_profiler_fn;
    /* all of the above are created the first time
       gimple_init_edge_profiler is called */

  static GTY(()) tree ic_void_ptr_var;
  static GTY(()) tree ic_gcov_type_ptr_var;
  static GTY(()) tree ptr_void;
    /* these are created by init_ic_make_global_vars,
       which is called the first time gimple_init_edge_profiler
     */

All of the above are created the first time gimple_init_edge_profiler is
called, and are never explicitly cleaned up (presumably the status quo
is that the pointers are never NULLed, so they are never collected by the
GC).

This state is shared between repeated invocations of the pass.

`gimple_init_edge_profiler` is used from outside this file in `profile.c`,
so perhaps this state needs to be made a per-context thing.
Alternatively, we could make it per-pass, and have
`gimple_init_edge_profiler` poke at it through::

  context->pass_manager->pass_ipa_tree_profile


`tree-sra.c`: pass_sra_early, pass_sra, pass_early_ipa_sra
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
"Scalar Replacement of Aggregates (SRA) converts some structure
references into scalar references, exposing them to the scalar
optimizers."

Single instances of each of pass_sra_early, pass_sra and pass_early_ipa_sra;
all three are gimple passes.

Various state in tree-sra.c, all static::

  static enum sra_mode sra_mode;
  static alloc_pool access_pool;
  static alloc_pool link_pool;
  static struct pointer_map_t *base_access_vec;
  static bitmap candidate_bitmap;
  static hash_table <uid_decl_hasher> candidates;
  static bitmap should_scalarize_away_bitmap, cannot_scalarize_away_bitmap;
  static struct obstack name_obstack;
  static struct access *work_queue_head;
  static int func_param_count;
  static bool encountered_apply_args;
  static bool encountered_recursive_call;
  static bool encountered_unchangable_recursive_call;
  static HOST_WIDE_INT *bb_dereferences;
  static bitmap final_bbs;
  static struct access no_accesses_representant;
  static struct {} sra_stats;

TODO

`tree-ssa.c`: pass_init_datastructures, pass_early_warn_uninitialized, pass_update_address_taken
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single instances of pass_init_datastructures, pass_early_warn_uninitialized.

pass_update_address_taken is apparently never used; patch posted as
http://gcc.gnu.org/ml/gcc-patches/2013-06/msg00205.html though Richard
Biener wants to keep the pass around for now.

File variables::

  static struct pointer_map_t *edge_var_maps;
    /* lazily created by redirect_edge_var_map_add
       used throughout the redirect_edge_var_map_* API
       destroyed by redirect_edge_var_map_destroy, called by delete_tree_ssa,
       which is called in two other places in the source tree.  */

The `redirect_edge_var_map_*` API is used in 5 other source files, so this looks
like it needs to be per-context state.

Plan:

  * introduce a `class redirect_edge_var_state` (or whatnot) and add to
    context, with usual MAYBE_STATIC on both the state member within the
    context and the fields of the state class itself.

  * convert the `redirect_edge_var_map_*` API to be methods of said class.

`tree-ssa-ccp.c`: pass_ccp and pass_fold_builtins
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
4 instances of pass_ccp; 2 of pass_fold_builtins

File state::

  static prop_value_t *const_val;
     /* allocated in ccp_initialize;
        freed in ccp_finalize;
        explicit uses in lines 229-836, but exposed by get_value API
        used in much of the file */

  static unsigned n_const_val;
     /* initialized in ccp_initialize */

i.e. an array.

TODO: how is this shared between the passes?


`tree-ssa-copy.c`: pass_copy_prop
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
8 instances of pass_copy_prop

State::

    static prop_value_t *copy_of;
    static unsigned n_copy_of;

though this one appears to be reinit-ed at execute time, so state doesn't
need preserving

`tree-ssa-copyrename.c`: pass_rename_ssa_copies
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
(5 of these)

TODO

`tree-ssa-dce.c`
^^^^^^^^^^^^^^^^
"Dead code elimination"

These are gimple passes:

* pass_dce (3 of these)
* pass_dce_loop (3 of these)
* pass_cd_dce (2 of these)

File state::

  static struct stmt_stats {...} stats;
  static vec<gimple> worklist;
  static sbitmap processed;
  static sbitmap last_stmt_necessary;
  static sbitmap bb_contains_live_stmts;
  static bitmap *control_dependence_map;
  static sbitmap visited_control_parents;

  static bitmap visited = NULL;
  static unsigned int longest_chain = 0;
  static unsigned int total_chain = 0;
  static unsigned int nr_walks = 0;
  static bool chain_ovfl = false;


TODO


`tree-ssa-dom.c`
^^^^^^^^^^^^^^^^
These are gimple passes:

  * pass_dominator (2 of these)
  * pass_phi_only_cprop (2 of these)

File state::

  static vec<expr_hash_elt_t> avail_exprs_stack;
    /* created/released at start/end of tree_ssa_dominator_optimize */

  static hash_table <expr_elt_hasher> avail_exprs;
    /* created/disposed at start/end of tree_ssa_dominator_optimize */

  static vec<tree> const_and_copies_stack;
    /* created/released at start/end of tree_ssa_dominator_optimize */

  static bool cfg_altered;
    /* inited near top of tree_ssa_dominator_optimize */

  static bitmap need_eh_cleanup;
    /* created/free at start/end of tree_ssa_dominator_optimize */

  static struct opt_stats_d opt_stats;
    /* zeroed at top of tree_ssa_dominator_optimize */

If I'm reading this right, all of the above state is per-invocation
state of `pass_dominator`.

TODO: verify absence of interactions with pass_phi_only_cprop


`tree-ssa-dse.c`: pass_dse
^^^^^^^^^^^^^^^^^^^^^^^^^^
2 instances of this gimple pass

File state::

  static bitmap need_eh_cleanup;
    /* allocated/freed at start/end of tree_ssa_dse, the execute hook */

This is per-invocation state.


`tree-ssa-forwprop.c`: pass_forwprop
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Four instances of this gimple pass.

File state::

  static bool cfg_changed;
     /* set to false at top of ssa_forward_propagate_and_combine
        (the execute hook)

        Shadowed by locals in:
          * remove_prop_source_from_use
          * forward_propagate_into_comparison
          * forward_propagate_into_gimple_cond

        Used as a global in:
          * tidy_after_forward_propagate_addr
          * simplify_gimple_switch_label_vec
          * ssa_forward_propagate_and_combine
        */

Appears to be treatable as per-invocation state; the shadowing
should probably be fixed.


`tree-ssa-ifcombine.c`: pass_tree_ifcombine
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single-instanced gimple pass.

Appears to have no internal state.


`tree-ssa-loop.c`: 18 different passes
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

There appears to be no state within *this* source file, but the bulk of
the implementation seems to be elsewhere - this source file just handles
wiring up other files as passes, with some simple gate/execute wrappers.

All 18 passes are gimple passes.

========================== ================ =  ================================
Pass                       Name             #  Execute hook
========================== ================ =  ================================
`pass_tree_loop`           `loop`           1  None; holder for the other
                                               passes

`pass_tree_loop_init`      `loopinit`       1  `tree_ssa_loop_init` in
                                               same file

`pass_lim`                 `lim`            3  `tree_ssa_loop_im`, calls
                                               into `tree_ssa_lim`
                                               defined in
                                               `tree-ssa-loop-im.c`, which
                                               has initialize/finalize
                                               pair

`pass_tree_unswitch`       `unswitch`       1  `tree_ssa_loop_unswitch`
                                               calls into
                                               `tree_ssa_unswitch_loops`
                                               defined in
                                               `tree-ssa-loop-unswitch.c`
                                               which seems to have no state

`pass_predcom`             `pcom`           1  `run_tree_predictive_commoning`
                                               which calls
                                               `tree_predictive_commoning`
                                               defined in `tree-predcom.c`
                                               Uses
                                               `initialize_original_copy_tables`
                                               and `free_original_copy_tables`
                                               from `cfg.c`; may need to
                                               encapsulate this as a class?

`pass_vectorize`           `vect`           1  `tree_vectorize` calls
                                               `vectorize_loops` defined
                                               in `tree-vectorizer.c`
                                               This pass guards one of the
                                               instances of pass_dce_loop

`pass_graphite`            `graphite0`      1  None; holder for graphite
                                               passes

`pass_graphite_transforms` `graphite`       1  `graphite_transforms` calls
                                               `graphite_transform_loops`
                                               defined in `graphite.c`

                                               State includes::

                                                 CloogState *cloog_state;
                                                 isl_ctx *the_isl_ctx;

                                               and these are used by other
                                               files.

`pass_check_data_deps`     `ckdd`           1  `check_data_deps` calls
                                               `tree_check_data_deps`
                                               defined in `tree-data-ref.c`

                                               State there appears to be just::

                                                 static struct datadep_stats
                                                 { /* */ } dependence_stats;

`pass_iv_canon`            `ivcanon`        1  `tree_ssa_loop_ivcanon` calls
                                               `canonicalize_induction_variables`
                                               defined in
                                               `tree-ssa-loop-ivcanon.c`
                                               which has state::

                                                 static vec<loop_p> loops_to_unloop;
                                                 static vec<int> loops_to_unloop_nunroll;

`pass_scev_cprop`          `sccp`           1  `scev_const_prop` defined in
                                               `tree-scalar-evolution.c` which has
                                               state shared with various other files
                                               e.g. `scalar_evolution_info` which is
                                               cleaned up by `scev_finalize ()`,
                                               called by 8 other source files.

`pass_record_bounds`       `*record_bounds` 1  `tree_ssa_loop_bounds` which calls
                                               `estimate_numbers_of_iterations`
                                               defined in
                                               `tree-ssa-loop-niter.c`
                                               (which appears to have no global
                                               state itself)
                                               and `scev_reset`

`pass_complete_unroll`     `cunroll`        1  `tree_complete_unroll` which calls
                                               `tree_unroll_loops_completely`
                                               defined in
                                               `tree-ssa-loop-ivcanon.c`
                                               which has state described above.

`pass_complete_unrolli`    `cunrolli`       1  `tree_complete_unroll_inner`
                                               which calls::

                                                 loop_optimizer_init
                                                 scev_initialize
                                                 tree_unroll_loops_completely
                                                 free_numbers_of_iterations_estimates
                                                 scev_finalize
                                                 loop_optimizer_finalize

`pass_parallelize_loops`   `parloops`       1  `tree_parallelize_loops` calls
                                               `parallelize_loops` defined in
                                               `tree-parloops.c` which has
                                               state (halfway down file)::

                                                 static GTY(()) bitmap parallelized_functions;

`pass_loop_prefetch`       `aprefetch`      1  `tree_ssa_loop_prefetch` calls
                                               `tree_ssa_prefetch_arrays`
                                               defined in
                                               `tree-ssa-loop-prefetch.c`
                                               which appears to have no state.

`pass_iv_optimize`         `ivopts`         1  `tree_ssa_loop_ivopts` calls
                                               `tree_ssa_iv_optimize`
                                               defined in
                                               `tree-ssa-loop-ivopts.c`
                                               Most state is wrapped in::

                                                 struct ivopts_data

                                               on the stack, but there's
                                               also::

                                                 static vec<tree> decl_rtl_to_reset;

                                               which is created/released in
                                               the init/finalize functions
                                               (was this an optimization,
                                               just missed, or awkward to get
                                               at since it's populated from a
                                               walk_tree callback that
                                               already uses its data ptr?)

`pass_tree_loop_done`      `loopdone`       1  `tree_ssa_loop_done` appears
                                               to be just cleanups for
                                               other state::

                                                 free_numbers_of_iterations_estimates ();
                                                 scev_finalize ();
                                                 loop_optimizer_finalize ();

========================== ================ =  ================================


`tree-ssa-loop-ch.c`: pass_ch
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single instance of pass_ch (gimple)

Appears to have no state

`tree-ssanames.c`: pass_release_ssa_names
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single instance of pass_release_ssa_names

pass_release_ssa_names seems to be just another cleanup pass::

  /* Return SSA names that are unused to GGC memory and compact the SSA
     version namespace.  This is used to keep footprint of compiler during
     interprocedural optimization.  */

`tree-ssa-math-opts.c`
^^^^^^^^^^^^^^^^^^^^^^
Four passes:

  * pass_cse_reciprocals
  * pass_cse_sincos
  * pass_optimize_bswap
  * pass_optimize_widening_mul

All 4 are single-instance.

State in the file::

  static struct {...} reciprocal_stats;
  static struct {...} sincos_stats;
  static struct {...} bswap_stats;
  static struct {...} widen_mul_stats;
    /* all of these are memset to zero at the beginning of the
       relevant execute hook */

  static struct occurrence *occ_head;
    /* set by register_division_in's call to insert_bb */

  static alloc_pool occ_pool;
    /* created and freed by execute_cse_reciprocals;
       used by occ_new and free_bb */

Plan: this is all per-invocation state, so introduce 4 state classes.


`tree-ssa-phiopt.c`
^^^^^^^^^^^^^^^^^^^
Gimple passes:

* pass_phiopt (3 of these)
* pass_cselim (single)

File state::

  static unsigned int nt_call_phase;
     /* zeroed by get_non_trapping */

  static struct pointer_set_t *nontrap_set;
     /* created by get_non_trapping, which returns it */

  static hash_table <ssa_names_hasher> seen_ssa_names;
     /* created and disposed by get_non_trapping */

The execute hooks of both passes are implemented using
`tree_ssa_phiopt_worker`, with:

  * `do_store_elim == false` for `tree_ssa_phiopt` (execute hook of
    `pass_phiopt`)

  * `do_store_elim == true` for `tree_ssa_cs_elim` (execute hook of
    `pass_cselim`)


`get_non_trapping` is called at the top of `tree_ssa_phiopt_worker` and
destroyed near the bottom, in both cases if `do_store_elim`.

Hence this appears to be per-invocation state of pass_cselim.


`tree-ssa-phiprop.c`: pass_phiprop
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
"Backward propagation of indirect loads through PHIs."

Single-instanced gimple pass.

File appears to have no state.


`tree-ssa-pre.c`
^^^^^^^^^^^^^^^^
Gimple passes:

  * pass_pre (single-instanced)

  * pass_fre (2 of these)

File state::

  static unsigned int next_expression_id;
  static vec<pre_expr> expressions;
  static hash_table <pre_expr_d> expression_to_id;
  static vec<unsigned> name_to_id;
  static alloc_pool pre_expr_pool;
  static vec<bitmap> value_expressions;
  static int *postorder;
  static int postorder_num;
  static struct {...} pre_stats;
  static bool do_partial_partial;
  static alloc_pool bitmap_set_pool;
  static bitmap_obstack grand_bitmap_obstack;
  static bitmap need_eh_cleanup;
  static bitmap need_ab_cleanup;
  static hash_table <expr_pred_trans_d> phi_translate_table;
  static sbitmap has_abnormal_preds;
  static sbitmap changed_blocks;
  static bitmap inserted_exprs;

  /* Local state for the eliminate domwalk.  */
  static vec<gimple> el_to_remove;
  static vec<gimple> el_to_update;
  static unsigned int el_todo;
  static vec<tree> el_avail;
  static vec<tree> el_avail_stack;

TODO


`tree-ssa-reassoc.c`: pass_reassoc
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Three instances of this gimple pass.

File state::

  static struct {...} reassociate_stats;
    /* cleared by init_reassoc */

  static alloc_pool operand_entry_pool;
    /* allocated by init_reassoc
       freed by fini_reassoc */

  static int next_operand_entry_id;
    /* inited by init_reassoc */

  static long *bb_rank;
    /* allocated by init_reassoc
       freed by fini_reassoc */

  static struct pointer_map_t *operand_rank;
    /* created by init_reassoc
       freed by fini_reassoc */

  static vec<tree> plus_negates;
    /* inited by init_reassoc
       released by fini_reassoc */

  static vec<oecount> cvec;
    /* created and released by undistribute_ops_list;
       used by oecount_hasher::hash and equal */

  static vec<repeat_factor> repeat_factor_vec;
    /* created and released by attempt_builtin_powi and only used
       there; can this be a local of that function? */


So most of this state is guarded by `init_reassoc` and `fini_reassoc`,
which are called in the execute hook, the rest is even more localized.

Plan: treat this as per-invocation state


`tree-ssa-sink.c`: pass_sink_code
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single instance of pass_sink_code, a gimple pass.

State::

  sink_stats
    /* this is initialized near the top of the execute hook,
       hence this is per-invocation state */

Plan: per-invocation state.


`tree-ssa-strlen.c`: pass_strlen
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single instance of this gimple pass.

State::

  static vec<int> ssa_ver_to_stridx;
     /* safe_grow_cleared at start of tree_ssa_strlen
        released at end of tree_ssa_strlen */
  static int max_stridx;
    /* set at start of tree_ssa_strlen */

  static alloc_pool strinfo_pool;
    /* created at start of tree_ssa_strlen
       freed at end of tree_ssa_strlen */

  static vec<strinfo, va_heap, vl_embed> *stridx_to_strinfo;
    /* created/freed within strlen_leave_block */

  static hash_table <stridxlist_hasher> decl_to_stridxlist_htab;
     /* lazily created by addr_stridxptr
        freed at end of tree_ssa_strlen */

  static struct obstack stridx_obstack;
     /* lazily created by addr_stridxptr
        freed at end of tree_ssa_strlen */

  struct laststmt_struct {...} laststmt;
     /* cleared at end of tree_ssa_strlen */

Based on the above, this mostly is per-invocation state, with
stridx_to_strinfo being even narrower than that.

Plan: treat above as per-invocaton state.


`tree-ssa-structalias.c`: pass_build_alias, pass_build_ealias, pass_ipa_pta
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
All are single instances. `pass_build_alias` and `pass_build_ealias` are
dummy gimple passes that do nothing except ensure that we execute
`TODO_rebuild_alias`.

`pass_ipa_pta` is an IPA pass.  It has a fair amount of per-file state
which appears to all be marked as static, with this exception::

  /* A map mapping call statements to per-stmt variables for uses
     and clobbers specific to the call.  */
  struct pointer_map_t *call_stmt_vars;

However it appears not to be used elsewhere.

Done:

  * made call_stmt_vars static (patch was
    http://gcc.gnu.org/ml/gcc-patches/2013-06/msg00194.html )

Plan:

 * assuming that I'm right in thinking this pass is called once, introduce
    a state class full of MAYBE_STATIC, and put the state on the stack in the
    execute hook; make functions in the file be member function of this state
    as necessary (again with MAYBE_STATIC)


`tree-ssa-uncprop.c`: pass_uncprop
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
2 instances of this gimple pass.

File state::

  static vec<tree> equiv_stack;
  static hash_table <val_ssa_equiv_hasher> val_ssa_equiv;
    /* both created near start of tree_ssa_uncprop
       both freed up near end of  tree_ssa_uncprop
       (this is the execution hook) */

This thus appears to be per-invocation state.


`tree-ssa-uninit.c`: pass_late_warn_uninitialized
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
2 instances of this gimple pass.

State::

  static struct pointer_set_t *possibly_undefined_names = 0;
    /* created near start of execute_late_warn_uninitialized;
       freed near end of execute_late_warn_uninitialized
       populated by find_uninit_use
       used by ssa_undefined_value_p, which is public and used
       by 4 other source files
       */

Plan: given the non-trivial interactions with other files, put
this in the context.


`tree-stdarg.c`: pass_stdarg
^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single-instanced gimple pass.

Appears to have already collected all its state within
`struct stdarg_info`, which is passed around by ptr.


`tree-switch-conversion.c`: pass_convert_switch
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single-instanced gimple pass.

State inside lshift_cheap_p::

  static bool init[2] = {false, false};
  static bool cheap[2] = {true, true};

This is a cache.

Used by `expand_switch_using_bit_tests_p` which in turn is used by
`process_switch`, used by the execute hook.

Other than that appears to have no state.

TODO


`tree-tailcall.c`: pass_tail_recursion and pass_tail_calls
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Two instances of pass_tail_recursion, later a single instance of
pass_tail_calls (both gimple).

State within tree-tailcall.c::

  static tree m_acc, a_acc;

Initialized thusly in `tree_optimize_tail_calls_1 (bool opt_tailcalls)`,
though not quite at top::

  a_acc = m_acc = NULL_TREE;

which is called by the execute hook of both passes (the former with
`false`, the latter with `true`).

`tree-vect-generic.c`: pass_lower_vector, pass_lower_vector_ssa
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
"Lower vector operations to scalar operations."

Single instance of pass_lower_vector, two instances of pass_lower_vector_ssa;
both are gimple passes.  The two instances of pass_lower_vector_ssa occur
before the instance of pass_lower_vector.

State within the file::

  static GTY(()) tree vector_inner_type;
  static GTY(()) tree vector_last_type;
  static GTY(()) int vector_last_nunits;

These appear to only be used inside::

  static tree
  build_word_mode_vector_type (int nunits)

and appear to be memoization and consolidation, so that there's a single
vector type per input `nunits` (param) and `word_mode` (global).

Plan:

  * since it needs to be shared by two different passes but they're both
    in the same source file, create a `class lower_vector_state` with
    MAYBE_STATIC so it's all just a singleton in the static build.

  * Have the first instance of `pass_lower_vector_ssa` own one of these,
    the second instance reference it, and the instance of
    `pass_lower_vector` reference it also.  To do the latter, the
    factory function for `pass_lower_vector` will need to access the first
    instance of `pass_lower_vector_ssa` via the `pass_manager`.

  * Turn functions into MAYBE_STATIC methods of the state class as
    necessary

  * Add gty methods to the state class to mark the GTY(()) data, and arrange
    for the passes to call them.

`tree-vectorizer.c`: pass_slp_vectorize, pass_ipa_increase_alignment
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Both have single instances. pass_slp_vectorize is a gimple pass,
pass_ipa_increase_alignment is a simple_ipa pass.

File has global state::

  LOC vect_location;
    /* used throughout the tree-vect-*.c code, though mostly in calls
       to dump_printf_loc () */

  vec<vec_void_p> stmt_vec_info_vec;
    /* used in inline functions defined in tree-vectorizer.h */

TODO

`tree-vrp.c`: pass_vrp
^^^^^^^^^^^^^^^^^^^^^^
Two instances of pass_vrp, a gimple pass.

File state::

  /* Near top of file: */
  static sbitmap *live;
    /* allocated and deleted by find_assert_locations */

  static bitmap need_assert_for;
    /* allocated and deleted by insert_range_assertions */

  static assert_locus_t *asserts_for;
    /* allocated and deleted by insert_range_assertions */

  static unsigned num_vr_values;
    /* set in vrp_initialize */

  static value_range_t **vr_value;
    /* allocated in vrp_initialize, freed in vrp_finalize */

  static bool values_propagated;
    /* set to false in vrp_initialize; set to true in vrp_finalize;
       used in get_value_range */

  static int *vr_phi_edge_counts;
    /* allocated in vrp_initialize, freed in vrp_finalize */

  static vec<edge> to_remove_edges;
    /* created and released within the execute hook */

  static vec<switch_update> to_update_switch_stmts;
    /* created and released within the execute hook */

  /* Near end of file: */
  static vec<tree> equiv_stack;
    /* created by identify_jump_threads (called by vrp_finalize)
       released by finalize_jump_threads (called halfway down execute hook)
       */

Based on the above, it appears that no tree-vrp.c state is shared between
repeated invocations of the pass: the state could be moved to the stack.

Plan:

  * introduce state class `vrp_state` (in an anonymous namespace) and put
    it on the stack of execute_vrp; mark everything as MAYBE_STATIC so that
    it's empty in a non-shared-lib build (and everything is effectively a
    global again).

  * possibly make the `find_assert_locations` data be locals there

  * possibly make the `insert_range_assertions` data be locals there

`tsan.c`: pass_tsan, pass_tsan_O0
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Two instances of pass_tsan, one of pass_tsan_O0.  Both are gimple passes.

The only state in this file appears to be this::

  static struct tsan_map_atomic
  {
    enum built_in_function fcode, tsan_fcode;
    enum tsan_atomic_action action;
    enum tree_code code;
  } tsan_atomic_table[] =

which looks like it could be made const.

Other that that, there appears to be no state within the file itself, AIUI,
there's an underlying thread sanitizer library, which may have state.

Done: made tsan_atomic_table const (patch posted as
http://gcc.gnu.org/ml/gcc-patches/2013-06/msg00204.html )


`var-tracking.c`: pass_variable_tracking
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Single instance of pass_variable_tracking (rtl)

Lots of file state.  `vt_finalize` appears to clean much of it up.

`web.c`: pass_web
^^^^^^^^^^^^^^^^^
Single instance of pass_web

Appears to have no file state.
