# tip-tree Review Citations

This file is a reference index for the SR-DH-* rules in `x86-tip.md`,
`tip-changelog.md`, `tip-code-style.md`, and `tdx-review.md`. Each rule
links to lore message-ids of Dave Hansen review messages where the rule
was raised.

These are evidence, not definitions. The rule text lives in the prompt
files; the message-ids here let reviewers confirm the rule reflects
real, repeated feedback.

To view a citation: `https://lore.kernel.org/all/<message-id>/` — append
the message-id (without angle brackets) to the URL.

Citations are listed in chronological order within each rule where
multiple are present. Date range covered: 2025-01 through 2026-05.

## Tag block (x86-tip.md)

### SR-DH-AUTH-1: Long author chains warrant a contributor note
- v8 TDX module update series, Patch 4 ("OK, yes, that's a long
  chain of developers, a note probably is warranted at this point")
- v9 TDX module update series, Patch 4 (3-name SoB chain — "This SoB
  chain at _least_ needs a note. It looks quite bizarre.")

## Subject lines (x86-tip.md)

### SR-DH-SUBJ-1: Subject must convey the behavior change
- `cfcfb160-fcd2-4a75-9639-5f7f0894d14b@intel.com` (TDX Module Extensions)
- `21824e7a-092e-...` (x86/fpu xfeature dynamic)
- `3c069d9f-501e-...` (TDX Connect)
- `8012d986-0d1f-...` (INVLPGB)

### SR-DH-SUBJ-2: No marketing terms in subjects
- `3c069d9f-501e-...` (TDX Connect)
- `97eff128-52e5-...` (TDX Connect rename)

## Changelogs (tip-changelog.md)

### SR-DH-CHG-1: "This patch ..." is forbidden
- `4e07bbe6-9f74-...` (tdx_safe_halt)
- `e029f69b-6d99-...` (ptdesc)
- `17e58542-1221-...` (mm-local)
- `21824e7a-092e-...` (xfeature dynamic)

### SR-DH-CHG-2: Spell-check before posting
- `4e07bbe6-9f74-...` (tdx_safe_halt)
- `054b7cab-4507-...` (tdx_safe_halt v2)

### SR-DH-CHG-3: Explain why, not the ABI/spec names
- `cfcfb160-fcd2-4a75-9639-5f7f0894d14b@intel.com` (Module Extensions)
- `9053a4ef-2de6-...` (tdx_safe_halt v5)
- `62bec236-4716-4326-8342-1863ad8a3f24@intel.com` (tdx_enable_ext)
- `fb5addcb-1cfc-...` (PAMT refcount)
- `4e07bbe6-9f74-...` (tdx_safe_halt)
- `054b7cab-4507-...` (tdx_safe_halt v2)

### SR-DH-CHG-4: Long changelog without rationale
- `74e49413-dd1a-...` (CET supervisor)
- `ea73a36a-8ac5-...` (fpu_guest_cfg)
- `8012d986-0d1f-...` (INVLPGB)
- `0b568da8-d9c3-...` (INVLPGB followup)
- `8c6c5d0a-c5e1-...` (32-bit memory-failure)

### SR-DH-CHG-5: Resource costs must be in the changelog
- `fb5addcb-1cfc-...` (PAMT refcount)
- `74e49413-dd1a-...` (CET supervisor)
- `62bec236-4716-4326-8342-1863ad8a3f24@intel.com` (Module Extensions, 12800 pages)

### SR-DH-CHG-6: Hunk not mentioned in the changelog
- `b0fb40f6-8baa-...` (ASI pgd)
- `b6fdba02-cc3e-...` (PMD ptdesc)

### SR-DH-CHG-7: Field listings belong in kerneldoc
- `ea73a36a-8ac5-...` (fpu_guest_cfg)

### SR-DH-CHG-8: Neutral tone in changelogs
- `a14531ab-f069-41f9-8c5c-9fe6f28a9454@intel.com` (PFN-direct mapping —
  "tone down the editorializing", "misguided struct page assumptions",
  "no business placing requirements")

### SR-DH-CHG-9: No forward-looking statements
- v8/v9 TDX module update series (Patch 2 in v8 review notes — "don't
  put stuff in for future series that may or may not happen")
- Subject thread: `[PATCH v8 ...] x86/virt/seamldr: ...`
  (`20260427152854.101171-1-chao.gao@intel.com` cover-letter thread)

### SR-DH-CHG-10: Avoid phrasings that force the reader's brain to leap
- v8/v9 TDX module update series, Patches 5–10 (etc., former/latter,
  "unlike", "see below", "not suitable" without backing reason)

### SR-DH-CHG-11: Kconfig changelogs explain when to disable
- v8/v9 TDX module update series, Patch 4 (Kconfig motivation gap;
  also unwarranted-prompt extension — promptless `def_tristate` would
  fit better given the `depends on` chain)

### SR-DH-CHG-12: Use concrete examples, not abstract placeholders
- `5e097cb0-362a-4dee-af68-9ce583312c97@intel.com` (v9 patch 05 —
  `1.5.x`/`1.5.y+1` rewritten with Sapphire Rapids / Granite Rapids /
  1.5.6 → 1.5.7)
- `5639b59e-8167-4e27-b7de-9e4be7f299f3@intel.com` (v9 patch 10 —
  vendor-x style placeholders)
- `9fbe4b85-a42d-42c3-8f8f-8e008b049b34@intel.com` (v9 patch 11
  followup — "Could you try a concrete example, please?")

### SR-DH-CHG-13: Don't bury the main solution sentence in mid-paragraph
- `6f2ea784-3fb2-4e91-ab8c-b8f26d591f38@intel.com` (v9 patch 06 —
  solution sentence followed by secondary `seamcall_prerr()` nit)
- `7d7fff5a-53a5-439e-9ff8-dcbd97f473cd@intel.com` (v9 patch 11 —
  "It's taking the key imperative and burying it.")

## Comments (tip-code-style.md)

### SR-DH-COM-1: Comments restating code; uncommented voodoo
- `355ad607-52ed-...` (PAMT helpers)
- `fd9ebb1c-8a5a-...` (alloc_pamt)
- `fb5addcb-1cfc-...` (PAMT refcount)
- `9053a4ef-2de6-...` (STI-shadow)
- `4e07bbe6-9f74-...` (idle naming)
- `b0fb40f6-8baa-...` (curpage=0)
- `554c886f-6678-...` (preallocate_sub_pgd)
- `a578e3b5-9fd3-4f69-943f-9415f4047e19@intel.com` (cache_state_incoherent)

### SR-DH-COM-2: New code is severely under-commented
- `fd9ebb1c-8a5a-...` (alloc_pamt)
- `fb5addcb-1cfc-...` (PAMT refcount)
- `a49c523c-0c9e-...` (general)
- `355ad607-52ed-...` (PAMT helpers)
- `112cc926-30aa-...` (mempool suggestion)

### SR-DH-COM-3: Don't delete prior comments without preserving rationale
- `68938275-3f6a-46fc-9b38-2c916fdec3d6@intel.com` (poison comment)
- `554c886f-6678-...` (sub_pgd comment)

### SR-DH-COM-4: Spec-comment preambles add words without information
- `e54e9647-18ec-4691-a979-c67798a7d85f@intel.com` (v9 patch 07 —
  "This is called the 'SEAMLDR_INFO' data structure ..." preamble
  rewritten without preamble; "The SEAMLDR.INFO documentation
  requires this to be aligned to a 256-byte boundary." rewritten as
  "Must be aligned to a 256-byte boundary.")
- `7d7fff5a-53a5-439e-9ff8-dcbd97f473cd@intel.com` (v9 patch 11 —
  SEAMLDR_PARAMS comment redirected from preamble to mechanism)

## Naming (tip-code-style.md)

### SR-DH-NAM-1: Function names must match semantics
- `fb5addcb-1cfc-...` (tdx_get_pamt_refcount)
- `bb0077bc-9dd2-...` (rename suggestion)
- `fd9ebb1c-8a5a-...` (tdx_nr_pamt_pages)
- `4e07bbe6-9f74-...` (tdx_idle naming)

### SR-DH-NAM-2: Erratum-only helpers must name the erratum
- `4bce6d90-c0d4-...` (reset_tdx_pages)
- `ce8923c7-a3df-...` (clear/reset_tdx_pages)

### SR-DH-NAM-3: Don't bury renames in unrelated patches
- `e029f69b-6d99-...` (ptdesc rename)
- `29599d4e-c2c7-...` (pmd_page → pmd)
- `9d088525-bbea-...` (table rename pushback)

### SR-DH-NAM-4: Match established kernel terminology
- `635e5c2d-9b4d-4c2b-8e7d-b9b6b3ec538f@intel.com` (tdx_host device —
  "'TDX module' was the first and it's 20x more common in the history")

### SR-DH-NAM-5: Variable name drift
- v9 TDX module update series, Patch 11 (variable repurposed without
  rename)

### SR-DH-NAM-6: Function and parameter names must be specific
- `7d7fff5a-53a5-439e-9ff8-dcbd97f473cd@intel.com` (v9 patch 11 —
  `populate_pa_list(... start, nr_pages)` rewritten as
  `populate_pa_list(... vmalloc_addr, vmalloc_len_pages)`)
- `488eeba6-5a08-4772-881f-7cf863f4a3f2@intel.com` (v9 patch 11 —
  `seamldr_install_module(const u8 *data, u32 size)` rewritten as
  `(... const u8 *tdx_image, u32 tdx_image_len)`; broadcast as
  series-wide ask)

## Code structure (tip-code-style.md)

### SR-DH-STR-1: Don't bifurcate code paths just to vary a parameter
- `cfcfb160-fcd2-4a75-9639-5f7f0894d14b@intel.com` (TDH_SYS_CONFIG bifurcation)

### SR-DH-STR-2: One-use error gotos are fine — defend them
- `a49c523c-0c9e-...` (Chao's removal suggestion → Dave pushed back)
- `70056924-1702-...` (rework with goto)
- `e3830b06-dac8-...` (followup)

### SR-DH-STR-3: `__free()` cleanup macros are not always an improvement
- `62586df1-fa54-...` (scoped_cond_guard rejection)
- `96d66314-a0f8-...` (__free_page)
- `70056924-1702-...` (full rework back to goto)

### SR-DH-STR-4: Don't increase indentation across a whole function
- `62586df1-fa54-...` (scoped_cond_guard)
- `17e58542-1221-...` (mm-local double-indent)

### SR-DH-STR-5: Hide complications inside helpers
- `c3f77974-f1d6-...` (PAMT bitmap)
- `112cc926-30aa-...` (mempool_t suggestion)

### SR-DH-STR-6: Use existing kernel infrastructure
- `112cc926-30aa-...` (mempool_t)
- `fb5addcb-1cfc-...` (kpte_to_vaddr suggestion)
- `e4c0c578-50e9-...` (pmd_ptdesc)

### SR-DH-STR-7: Don't copy-paste; refactor
- `c9f72a69-e220-...` (tdx_page_array_create_iommu_mt)
- `543bcbd5-c217-...` (tdx_clear_page consolidation)
- `ce8923c7-a3df-...` (clear/reset_tdx_pages)

### SR-DH-STR-8: Vertically align related lines
- `fb5addcb-1cfc-...` (start/end alignment)
- `1e142e20-e3b8-...` (CPU-ID table alignment)

### SR-DH-STR-9: Prefer the obvious form over the clever one
- v9 TDX module update series, Patch 1 (5-read return-value flow —
  "If you have to read through code 5 times to understand the flow,
  you are doing something wrong. This is how we miss race conditions.")
- v9 TDX module update series, Patch 8 (ternary form rejected)
- v9 TDX module update series, Patch 10 (overly clever switch when an
  if would suffice)

## Patch decomposition (tip-code-style.md)

### SR-DH-DEC-1: Don't combine code moves with new functionality
- `5cfb2e09-7ecb-...` (Consolidate TDX error handling)

### SR-DH-DEC-2: Refactor → infrastructure → feature ordering
- `a287cfc1-da35-...` (CET supervisor reorder)
- `97eff128-52e5-...` (TDX Connect series structure)

### SR-DH-DEC-3: Big series must be broken into chapters
- `3c069d9f-501e-...` (TDX Connect)
- `97eff128-52e5-...` (TDX Connect followup)
- `cfcfb160-fcd2-4a75-9639-5f7f0894d14b@intel.com` ("needs to be 4 or 5 patches")

### SR-DH-DEC-4: Single patch doing >3 logical things needs splitting
- `cfcfb160-fcd2-4a75-9639-5f7f0894d14b@intel.com`
- `17e58542-1221-...` (mm-local)

## API design (tip-code-style.md)

### SR-DH-API-1: APIs should not require const-cast traps
- `e029f69b-6d99-...` (pagetable_free)

### SR-DH-API-2: Common defaults belong inside the helper
- `e029f69b-6d99-...` (pagetable_alloc / __GFP_ZERO)
- `1a1b0f7c-aa3f-...` (X86_FEATURE_HYPERVISOR encoding)

### SR-DH-API-3: Asymmetric pairs must share naming
- `ce8923c7-a3df-...` (tdx_clear_page vs reset_tdx_pages)
- `543bcbd5-c217-...`

### SR-DH-API-4: Use lockdep_assert_*; don't write "caller must hold X"
- `a578e3b5-9fd3-4f69-943f-9415f4047e19@intel.com` (preemption assert)

## Type discipline (tip-code-style.md)

### SR-DH-TYP-1: Use `struct page *` for kernel-side physical memory
- `872c17f3-9ded-...`
- `033f56f9-fb66-...`
- `ad9b3b96-324a-...`
- `4f8c63f8-6b5f-...` (tdx_paddr_t / struct-page discussion)

### SR-DH-TYP-2: Don't conflate hardware width with kernel type
- `3a32ce4a-b108-...`
- `9a4752a4-e783-4f03-babf-23c31cee4ff9@intel.com` (KeyID separation)
- `b4f08d43-c421-4e8d-9bbb-c954c4472f8a@intel.com` (char→u8)

### SR-DH-TYP-3: Bitfields are dangerous when the value gets passed around
- `753cd9f1-5eb7-...`
- `5907bad4-5b92-...`

## TDX / coco specific (tdx-review.md)

### SR-DH-TDX-1: SEAMCALL/TDCALL status mapping must be centralized
- `14b13fe4-7a0d-...` (extend_rtmr)
- `5cfb2e09-7ecb-...` (consolidation)
- `3e55fd58-1d1c-...`
- `1fbdfffa-ac43-...`

### SR-DH-TDX-2: Don't pass combined keyid|paddr through kernel paths
- `487c5e63-07d3-41ad-bfc0-bda14b3c435e@intel.com`
- `9a4752a4-e783-4f03-babf-23c31cee4ff9@intel.com`
- `79eca29a-8ba4-4ad9-b2e0-54d8e668f731@intel.com` (MCE recovery for TDX/SEAM)

### SR-DH-TDX-3: paddr ↔ keyed-paddr conversion belongs in one helper
- `9654f59b-9b8b-...` (proposed mk_keyed_paddr)
- `7df33918-4780-...` (landed)

### SR-DH-TDX-4: Erratum workarounds need the bug name in the function name
- `4bce6d90-c0d4-...`
- `ce8923c7-a3df-...`
- `543bcbd5-c217-...`
- `a0d5b60d-ea40-4f99-aed7-003102517248@intel.com` (tdx_clear_page consolidation)

### SR-DH-TDX-5: Don't apply TDX-specific policy at generic kernel boundaries
- `91df7051-2405-...`
- `b439abd6-9fd9-...`
- `ca275d32-c9fd-...`
- `68938275-3f6a-46fc-9b38-2c916fdec3d6@intel.com` (PageHWPoison check)

### SR-DH-TDX-6: TDX cover letters must motivate the kernel-side problem
- `3c069d9f-501e-...`
- `fb5addcb-1cfc-...`
- `2537ad07-6e49-...`

### SR-DH-TDX-7: Don't depend on TDX-module behavior; user-visible UX costs need extraordinary justification
- `0b3e3123-b15d-...` (Face 1: locking-scheme dependency)
- `ca688bca-df3f-...` (Face 1: 5-year stability)
- `7b3babcd-0b09-4d36-b713-3e55ded1696d@intel.com` (Face 2: forced CPUs
  online — "There needs to be a *LONG* justification why there is no
  other choice here. There are very good reasons to leave CPUs offline
  forever.")

## CPU feature detection (tip-code-style.md)

### SR-DH-CPU-1: Use x86_match_cpu() and x86_cpu_id[]
- `4da8a846-d4c5-...`
- `1e142e20-e3b8-...`

### SR-DH-CPU-2: Microcode-version lists are despised — use synthetic flags
- `4da8a846-d4c5-...`
- `1e142e20-e3b8-...`

### SR-DH-CPU-3: Reference X86_FEATURE_* constants in CPUID predicates
- `1a1b0f7c-aa3f-...`
- `cc091a6e-74a1-...`

## Init ordering (tip-code-style.md)

### SR-DH-INIT-1: Separate "populate the data" from "enable enforcement"
- `6e768f25-3a1c-...` (setup_cr_pinning)
- `f9e4ef3f-2844-...` (LASS deferral)

## Process (tip-code-style.md)

### SR-DH-PROC-1: Don't post `vN.1` patches; resend full vN+1 instead
- `a1c95555-aa68-...`

### SR-DH-PROC-2: Don't churn code just to use new infrastructure
- `62586df1-fa54-...` (scoped_cond_guard)
- `e029f69b-6d99-...` (ptdesc)
- `29599d4e-c2c7-...` (pmd rename)

## Socratic style (tip-code-style.md)

### SR-DH-SOC-1: Single-line "Why?" responses signal missing rationale
- `a0d5b60d-ea40-4f99-aed7-003102517248@intel.com` ("Why keep the old prototype?")
- `9d86698d-525e-...`
- `5a25c22b-1ed8-...`
- `b0fb40f6-8baa-...`
- `a578e3b5-9fd3-4f69-943f-9415f4047e19@intel.com`
- `4be5db34-aadb-49e3-9a94-49d39c8bd31d@intel.com`
- `7927271c-61e6-...`
- `8c6c5d0a-c5e1-...`

### SR-DH-SOC-2: "What does this comment mean?" signals vague terminology
- `cfcfb160-fcd2-4a75-9639-5f7f0894d14b@intel.com`
- `62bec236-4716-4326-8342-1863ad8a3f24@intel.com`
- `b0fb40f6-8baa-...`
- `4e07bbe6-9f74-...`
- `7df33918-4780-...`

### SR-DH-SOC-3: "Which one is right?" signals internal inconsistency
- `355ad607-52ed-...` (sizeof variants)
- `08338619-6aa1-...` (unrestricted/nonsensitive)
- `c9f72a69-e220-...` (copy-paste asymmetry)

## Other (tip-code-style.md)

### SR-DH-OTH-1: Defensive code for theoretical bugs is rejected
- `a8d517f5-80fc-...`
- `0d8be9b7-1d81-...`

### SR-DH-OTH-2: "We need this" without specifics gets bounced
- `a2042a7b-2e12-...` (kdump)
- `8012d986-0d1f-...` (INVLPGB)
- `0b568da8-d9c3-...` (INVLPGB followup)

### SR-DH-OTH-3: Acked-by does not mean "all comments addressed"
- `ce5e2f44-144b-...`
- `04c4544b-6acc-...`
- `b4f08d43-c421-4e8d-9bbb-c954c4472f8a@intel.com`

## Note on truncated message-ids

Many message-ids above are shown in shortened form (e.g.
`4e07bbe6-9f74-...`) because the full IDs were captured during corpus
mining without consistent full-form preservation. The first few
characters are unique within the corpus, and the full IDs can be
recovered by searching lore for the corresponding subject from the
patterns file at `.work/dave-patterns.md`. For systematic
re-verification, the corpus files at `.work/dave-corpus/` carry the
full IDs.
