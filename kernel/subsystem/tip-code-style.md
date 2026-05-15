# tip-tree Code Style Subsystem Details

This guide loads alongside `x86-tip.md` and applies to code in any
tip-tree-routed patch. It captures coding-style preferences enforced by
the tip maintainers (especially Dave Hansen) that go beyond
`Documentation/process/coding-style.rst` and beyond what
`Documentation/process/maintainer-tip.rst` codifies.

Code that is technically correct but violates these rules cycles through
review for additional revisions. The patterns below are derived from
recurring corrections in lore.

## Comments: explain *why*, not *what*

The kernel coding style says comments should explain non-obvious
operations. The tip maintainers go further: comments that paraphrase the
following line of code are noise, and uncommented non-obvious code is a
defect. Both are correctable, both come up repeatedly in review.

```c
// WRONG: comment restates the next line
/* Decrement refcount and check for zero */
if (refcount_dec_and_test(&p->refcnt)) {
        // ...
}

// CORRECT: comment explains why this matters
if (refcount_dec_and_test(&p->refcnt)) {
        /*
         * Last reference dropped here and the memcg charge must be
         * uncharged before the slab is freed, otherwise the per-cgroup
         * counters drift permanently.
         */
        // ...
}
```

The high-value content of a comment is: ordering constraints, locking
constraints, hardware/spec-mandated behavior, "why this must be last",
why a less-obvious approach was preferred over a more-obvious one.

### SR-DH-COM-1: Comments restating code; uncommented voodoo

Two failure modes, same root cause:

1. The comment tells the reader what the next line does. The next line
   already says that.
2. The code does non-obvious bit math, pointer aliasing, ABI-shaped
   magic, or unusual ordering, and there is no comment at all.

> This is sheer voodoo. Voodoo on its own is OK. But uncommented voodoo
> is not.
> This comment is giving the 'what' but is weak on the 'why'.

**Detection signal**:
- A comment whose text could be machine-generated from the next line of
  code (`/* Allocate page */` over `alloc_page()`, `/* Loop over CPUs */`
  over `for_each_cpu()`).
- Bit shifts, masks, type punning, casts, or pointer arithmetic without
  a comment explaining the encoding or invariant.
- ABI-shaped values (TDX module status codes, SEAMCALL packed args, MSR
  bit layouts) constructed inline without a reference to the spec or the
  invariant being maintained.

**REPORT as subjective regressions**: code that paraphrases itself in
comments, or non-obvious code with no comment.

### SR-DH-COM-2: New code is severely under-commented

A series that introduces hundreds of lines of new code with only a
handful of comments earns a series-wide rejection. Dave Hansen will
literally count comment lines:

> There are 4 lines of comments in the 216 lines of new code.
> this series is gloriously unencumbered by comments
> This is getting to be severely under-commented.

The threshold is judgment-based, not a strict ratio, but as a calibration:
a non-trivial subsystem patch averaging fewer than one comment per ~30
lines of new logic is a flag.

**Detection signal**: count comment lines (excluding licensing/SPDX
headers and comments that just paraphrase code per `SR-DH-COM-1`)
against new logic lines. Low ratio in non-trivial subsystem code is a
red flag.

**REPORT as subjective regressions**: significant new code with
near-zero meaningful comments.

### SR-DH-COM-4: Spec-comment preambles add words without information

Comments that introduce a spec-defined data structure or constraint
must lead with the substantive fact. The "this is called X and is
defined in Y" preamble adds words without information — the reader
spends a line on the meta-fact ("this thing exists in a spec") before
reaching the load-bearing fact ("this is the breakdown into
page-sized pieces" / "alignment is N").

```
WRONG:
/*
 * This is called the "SEAMLDR_INFO" data structure and is defined
 * in the Intel SEAMLDR Interface Specification.
 */

CORRECT:
/* The "SEAMLDR_INFO" structure defined in the SEAMLDR ABI spec. */

WRONG:
/*
 * The SEAMLDR.INFO documentation requires this to be aligned to a
 * 256-byte boundary.
 */

CORRECT:
/* Must be aligned to a 256-byte boundary. */

WRONG:
/*
 * This is called the "SEAMLDR_PARAMS" data structure and is defined
 * in the SEAMLDR spec. It describes the TDX module that will be
 * installed.
 */

CORRECT:
/*
 * SEAMLDR_PARAMS: breaks the TDX module image into page-sized
 * pieces for the P-SEAMLDR ABI.
 */
```

The third example also shows the related failure: when the comment
follows the preamble with a re-statement of what the struct's name
already conveys ("describes the TDX module that will be installed"
above a struct named `SEAMLDR_PARAMS` for installing modules), the
substantive content — the *mechanism* — never appears.

> Yeah, but that's not super useful.

**Detection signal**: comment block introducing a spec-derived ABI
struct or constraint contains:
- `This is (called|known as|named) "?<NAME>"?` as the leading
  sentence
- `<spec-name> (documentation )?(requires|states|specifies)` as the
  leading clause when the constraint itself could be stated directly
- The first sentence after the preamble re-states what the struct's
  identifier already conveys (a `_PARAMS` struct's first comment
  sentence saying "parameters for X")

**REPORT as subjective regressions**: spec-derived comments whose
opening sentence is meta-narration rather than the substantive fact.

### SR-DH-COM-3: Don't delete a comment without preserving its rationale

When refactoring removes a multi-line comment, the rationale of the
original must survive somewhere — either in the new comment, in the
function name, or in adjacent kerneldoc. Silent deletion of explanatory
comments is a regression.

> I think the old comment needs to stay in some form.
> This comment is pretty important, IMNHO, and you zapped it.

**Detection signal**: hunk deletes a multi-line `/* ... */` block and the
surrounding new code does not pick up an equivalent rationale.

**REPORT as subjective regressions**: refactoring patches that remove
load-bearing comments.

## Locking documentation uses lockdep, not prose

Comments saying "Caller must hold foo->lock" don't enforce anything.
`lockdep_assert_held(&foo->lock)` does — and lockdep will warn at
runtime if a caller forgets.

```c
// WRONG: comment is not enforced
/* Caller must hold foo->lock */
void func(struct foo *foo)
{
        // ...
}

// CORRECT: lockdep enforces; comment is unnecessary
void func(struct foo *foo)
{
        lockdep_assert_held(&foo->lock);
        // ...
}
```

Same applies to preemption (`lockdep_assert_preemption_disabled()`),
RCU (`rcu_read_lock_held()` / `RCU_LOCKDEP_WARN()`), and IRQ context.

**Detection signal**: function-level comments matching
`/\*.* (must|should) (hold|be called with|be in) .*(lock|preempt|irq|rcu)/`
without an accompanying `lockdep_assert_*` call.

**REPORT as subjective regressions**: locking requirements documented in
prose without lockdep enforcement.

## Function names

Function names must accurately describe what the function returns or
does. The kernel coding style covers this in spirit; the tip maintainers
enforce it stringently.

### SR-DH-NAM-1: Names must match semantics

`get_*` and `*_get` conventionally mean "acquire a reference". A function
named `tdx_get_pamt_refcount()` that returns a pointer to a refcount
location, but does not increment it, is misnamed.

`*_pages` conventionally means "the number of pages". A function named
`tdx_nr_pamt_pages()` that returns "the number of pages needed for a
single dynamic PAMT granule" is misnamed.

> 'get refcount' usually means 'get a reference'. This is looking up the
> location of the refcount.
> Despite the naming this function does not return the number of TDX PAMT
> pages. It returns the number of pages needed for each *dynamic* PAMT
> granule.

**Detection signal**:
- `get_*` / `*_get` that returns a pointer rather than a refcount-acquired
  object
- `*_pages` / `*_size` / `*_count` whose return value does not exactly
  match the suggested noun
- Names that suggest one operation and perform a different one

**REPORT as subjective regressions**: misleading function names where the
verb or noun does not match the function body.

### SR-DH-NAM-2: Erratum-only helpers must name the erratum

A function whose only reason for existing is to work around a specific
erratum needs the erratum/quirk in its name. A generic name like
`reset_tdx_pages()` for a function that only runs on CPUs with
`X86_BUG_TDX_PW_MCE` makes the call sites look like normal reset
operations rather than workarounds.

> If the only possible use for reset_tdx_pages() is for the erratum,
> then it needs to be in the function name. tdx_quirk_reset_paddr().

**Detection signal**: new function whose body is gated on a `BUG`/`QUIRK`
flag (e.g. `boot_cpu_has_bug(X86_BUG_*)`, `cpu_feature_enabled(X86_QUIRK_*)`),
or whose only call sites are on workaround paths, but whose name does not
include `quirk`, the bug name, or another erratum-encoding token.

**REPORT as subjective regressions**: erratum-only helpers with generic
names.

### SR-DH-NAM-6: Function and parameter names must be specific to what they hold

Generic parameter names like `data`, `size`, `start`, `len`,
`max_entries`, `buf` are only acceptable for genuinely generic
helpers (memcpy-style primitives). When the function operates on a
specific kind of object — a `struct tdx_image`, a `vmalloc()`-backed
payload, a P-SEAMLDR PA list — the parameter names must encode that
specificity. Generic names hide the type, the lifetime, and the
provenance of the data, which are exactly the things a reader needs
to reason about boundary conditions.

> Can we please do better than 'data' and 'size'? ... This goes for
> the *entire* series. Why not call it 'tdx_image'?

```
WRONG:
int seamldr_install_module(const u8 *data, u32 size);

void populate_pa_list(u64 *pa_list, u32 max_entries,
                      const u8 *start, u32 nr_pages);

CORRECT:
int seamldr_install_module(const u8 *tdx_image, u32 tdx_image_len);

void populate_pa_list(u64 *pa_list, u32 pa_list_len,
                      const u8 *vmalloc_addr, u32 vmalloc_len_pages);
```

This is the parameter-level analogue of `SR-DH-NAM-1`
(function-name-matches-semantics).

**Detection signal**: a new function takes parameters with generic
names (`data`, `size`, `len`, `start`, `buf`, `max_entries`, `nr`,
`count`) when:
- The function name implies a specific object type (`tdx_*`,
  `seamldr_*`, `pamt_*`, etc.)
- The function body uses the parameter as a specific kind of object
  (deserializes it as a struct, treats it as a virtual address from
  a specific allocator, indexes it with a specific stride)

**REPORT as subjective regressions**: function or parameter names
that use generic placeholders when the function operates on a
specific object type.

### SR-DH-NAM-5: Variable name must keep matching what the variable holds

Renaming a variable when its meaning changes is cheap; failing to
rename when the meaning has shifted is a defect. A variable named
`pamt_array` that started life holding an array but has been
repurposed to hold a single granule descriptor is misleading at every
later read site.

This is the variable-level analogue of `SR-DH-NAM-1` (function names
matching semantics). It comes up specifically when patches modify
existing functions: the body changes, the name stays.

**Detection signal**: a hunk modifies the assignments to or uses of an
existing variable such that the variable now holds a meaningfully
different kind of value (a count instead of a pointer, a single item
instead of an array, an offset instead of an absolute address) but the
variable's name is unchanged.

**REPORT as subjective regressions**: variables whose new behavior in
the patch contradicts their existing name.

### SR-DH-NAM-4: Match established kernel terminology

When the kernel git log has established a canonical capitalization,
hyphenation, or word-form for a name (e.g., `TDX module` — lowercase
`m`, no hyphen), patches that introduce a new variant (`TDX-Module`,
`TDX Module`, `tdx-module` in user-visible places) get bounced. The
established form has authority because it appears in commit messages,
function names, struct names, and Documentation/ across the existing
tree.

> How about doing what the Linux kernel does -- and has been doing --
> instead of trying to pick a new policy a few years into the kernel
> dealing with TDX? 'TDX module' was the first and it's 20x more common
> in the history.
> If you don't have a strong preference, why are you arguing for
> change now?

**Detection signal**: a term in the changelog, comments, sysfs path,
or user-visible string uses a capitalization or hyphenation form that
does not match the dominant form in `git log --grep` history. Quick
check: `git log --oneline --grep='<term>' | grep -ic '<variant>'` vs
`<dominant>`. A 5%-or-less minority share of the dominant form is the
threshold for a flag.

This rule is most actionable for terms a series introduces or modifies
broadly (e.g., a sysfs ABI label, a series-wide naming convention).

**REPORT as subjective regressions**: introduction of a term variant
that conflicts with the dominant historical form for the same concept.

### SR-DH-NAM-3: Don't bury renames in unrelated patches

When a patch's primary purpose is type conversion, bug fix, or feature
addition, drop-in renames mixed into the same patch make the diff harder
to review and the git history harder to bisect. Renames belong in
dedicated rename patches, justified on their own.

> Why change the name? Nobody cares what this is doing internally.
> Could we please leave out the unnecessary churn from the renames?
> Let's just talk about renaming later, please.

**Detection signal**: a single patch contains both:
- A non-rename change (type conversion, refactor, feature)
- Function/variable/struct renames not mentioned in the subject or
  changelog

**REPORT as subjective regressions**: renames buried in patches whose
primary purpose is something else.

## Code structure

### SR-DH-STR-1: Don't bifurcate a code path just to vary one parameter

When the same call must run with two argument variants, set up the
variables above and call once below. Don't write `if (cond) call(A); else
call(B);` when the only difference is one argument.

```c
// WRONG: two near-identical call sites
if (extended)
        ret = seamcall(TDH_SYS_CONFIG_EXT, &args, NULL);
else
        ret = seamcall(TDH_SYS_CONFIG, &args, NULL);

// CORRECT: parameterize, call once
u64 op = extended ? TDH_SYS_CONFIG_EXT : TDH_SYS_CONFIG;
ret = seamcall(op, &args, NULL);
```

> bifurcating code paths is discouraged. It's much better to not copy
> and paste the code and instead name your variables and change *them*
> in a single path.

**Detection signal**: `if/else` where both branches call the same
function with different argument vectors.

**REPORT as subjective regressions**: bifurcated call sites that could be
parameterized.

### SR-DH-STR-2: One-use error gotos are fine — don't refactor them away

Single-use `goto` for error cleanup is **idiomatic kernel C** and Dave
Hansen actively defends it. Reviewers who suggest "just inline the
cleanup since it's only used once" get explicit pushback. The error goto
serves as an error landing site that lets the reader scan the
non-error-case flow without being interrupted by cleanup code.

> There's no rule saying that gotos need to be used more than once. It's
> idiomatic kernel C to use a goto as an error landing site. ... I
> *prefer* this because it lets me read the main, non-error-case flow
> through the function.
> I just reworked this to use normal goto's. It looks a billion times
> better.

**Detection signal**: review feedback or refactoring suggestions
proposing the inlining of single-use error gotos. This is a signal to
*push back*, not to approve the change.

**REPORT as subjective regressions**: patches whose stated purpose is
removing single-use error gotos in favor of inlined cleanup or `__free()`
attributes.

### SR-DH-STR-3: `__free()` cleanup macros are not always an improvement

The cleanup attributes (`__free()`, `guard()`, `scoped_guard()`) are
available, but the tip maintainers — Dave Hansen in particular — push
back on patches that introduce them just because they are new. When line
length is already long, when the allocator types vary across paths, or
when the readability gain is unclear, plain goto is preferred.

> Please remove all these __free()'s unless you have specific evidence
> that they make the code better.
> I suspect the use of __free() here is actually hurting readability.
> We're not going to introduce bugs ... just to move to the newest shiny
> thing.

This is intentionally in tension with `subsystem/cleanup.md`. That guide
covers correctness rules for `__free()` *when it is used*. This rule
covers whether to introduce `__free()` *at all* in tip-tree code. When
both apply, the tip-tree preference takes precedence for new code in
`arch/x86/`.

**Detection signal**:
- A patch's primary purpose is "convert X to use `__free()`"
- New `__free()` annotations are introduced as part of conversion churn,
  not as part of net-new code
- `__free()` is added to a function that previously had readable goto-based
  error handling
- Lines containing `__free(...)` exceed 80 columns and the overflow is
  caused by the cleanup annotation

**REPORT as subjective regressions**: `__free()` introduced into code
that already had working goto-based cleanup, especially when it forces
line-length problems or replaces idiomatic error landing sites.

### SR-DH-STR-4: Don't increase indentation across a whole function

Wrapping a function body in a `scoped_guard()` block, a `scoped_cond_guard()`,
or a nested `if (X) { ... }` that re-indents everything makes the function
less readable. Prefer the early-return form: `if (!X) return; ...`.

```c
// WRONG: whole function indented one extra level
void func(...)
{
        scoped_guard(mutex)(&lock) {
                /* every line here is one indent deeper */
                ...
                return;
        }
}

// CORRECT: function-scoped guard, no extra indent
void func(...)
{
        guard(mutex)(&lock);
        /* function body at original indent */
        ...
}
```

> Indentation matters. Increasing the indenting on the whole function
> makes it less readable. Don't do it like that ^.
> What you have here double-indents the whole function.

**Detection signal**: a hunk takes an existing function body and inserts
a wrapper block (scoped_*, nested if) that bumps indentation by one level
across the entire function.

**REPORT as subjective regressions**: refactors that increase whole-function
indentation without removing comparable indentation elsewhere.

### SR-DH-STR-5: Hide complications inside helpers; keep the main flow clean

When a feature adds a new "case" (dynamic-PAMT path, virtualization
extension), fold the case into existing helpers rather than crufting it
into the main loop with conditionals.

> This is the wrong place to do this. Hide it in tdmr_get_pamt_sz().
> Don't inject it in the main code flow here and complicate the for
> loop.

**Detection signal**: new conditional branches inserted directly into
top-level loops or state machines for cross-cutting features. Often a
smell that the helper API needs widening, not the caller.

**REPORT as subjective regressions**: cross-cutting feature checks added
to top-level control flow rather than encapsulated in helpers.

### SR-DH-STR-6: Use existing kernel infrastructure, don't open-code

The kernel has primitives for many of the things people open-code:
`mempool_t`, `apply_to_page_range()`, `kpte_to_vaddr()`, `cmpxchg()`-based
counters, `xa_*`. Reach for the existing primitive before writing a new
one.

> This is compact and all. But it's really just an open-coded, simplified
> version of what mempool_t plus mempool_init_page_pool() would do. Could
> you take a look at that and double check that it's not a good fit
> here, please?
> we really need a kpte_to_vaddr() helper here. This is really ugly.

**Detection signal**:
- A new small data structure that re-implements list-of-pages with a
  counter (likely `mempool_t`)
- Ad-hoc page-table walks (`__va(PFN_PHYS(pte_pfn(ptep_get(...))))`
  sequences)
- Hand-rolled per-CPU/per-arch primitives that already exist
- Custom flag-bit allocators where `IDR` or `xarray` would do

**REPORT as subjective regressions**: open-coded versions of existing
kernel primitives.

### SR-DH-STR-7: Don't copy-paste; refactor

Two new helpers with mostly-identical bodies, or a new helper that is
mostly identical to an existing one, gets bounced. The maintainers want
one canonical implementation per logical operation.

> Please endeavor to find another way to do this. This is virtually a
> copy-and-paste of the earlier code.
> Could we consolidate them, please? There's no reason to sprinkle
> knowledge of movdir64b's memory ordering rules all across the tree.

**Detection signal**: two new functions in the same patch (or a new
function alongside an existing one in the same subsystem) with >70%
identical bodies.

**REPORT as subjective regressions**: copy-paste duplication that should
be a shared helper.

### SR-DH-STR-9: Prefer the obvious form over the clever one

Code that requires the reader to think hard about *the form* (rather
than the logic) is a defect. Specific anti-patterns:

- **Ternary expressions where an `if/else` would do**, especially when
  one or both arms are non-trivial. `x ? f(a, b, c) : g(a, b, c)` reads
  worse than four lines of `if`.
- **`switch` statements with two or three cases** that could be a plain
  `if/else if`. The `switch` form looks "clever" but adds visual
  weight without expressive value.
- **Forward-reference prose in code** (`/* explained below */`,
  `/* see comment in foo() */`) where the explanation could go inline.
- **Code paths where the reader has to read through five times to
  understand the flow** (e.g., what fills the return value vs. the
  "done" state in a multi-exit function). If tracing the flow takes
  multiple passes, restructure for a single linear read.

> If you have to read through code 5 times to understand the flow,
> you are doing something wrong. This is how we miss race conditions.

**Detection signal**:
- Ternary expression where either arm is a function call with 3+
  arguments
- `switch` with ≤3 case labels (excluding `default`) where the cases
  do not benefit from `switch`-specific behavior (fallthrough,
  exhaustiveness over an enum)
- `/* see (above|below|comment in) ... */` style references in code
- A function with 3+ exit points where the relationship between the
  exit point and the value of the return variable requires tracing

**REPORT as subjective regressions**: code that uses the clever form
when the obvious form would carry the same meaning.

### SR-DH-STR-8: Vertically align related lines

When several consecutive lines do parallel work — array initializers,
CPU-ID match table entries, sequence of paired assignments — align the
columns so the structural symmetry is visible at a glance.

```c
// WRONG: unaligned
static const struct x86_cpu_id table[] = {
        X86_MATCH_VFM_STEPS(INTEL_ATOM_GRACEMONT, X86_STEP_MIN, 0x4, 0),
        X86_MATCH_VFM_STEPS(INTEL_ATOM_CRESTMONT, X86_STEP_MIN, X86_STEPPING_ANY, 0),
        X86_MATCH_VFM_STEPS(INTEL_ATOM_TREMONT, X86_STEP_MIN, X86_STEPPING_ANY, 0),
        {}
};

// CORRECT: aligned, structure visible
static const struct x86_cpu_id table[] = {
        X86_MATCH_VFM_STEPS(INTEL_ATOM_GRACEMONT, X86_STEP_MIN, 0x4,              0),
        X86_MATCH_VFM_STEPS(INTEL_ATOM_CRESTMONT, X86_STEP_MIN, X86_STEPPING_ANY, 0),
        X86_MATCH_VFM_STEPS(INTEL_ATOM_TREMONT,   X86_STEP_MIN, X86_STEPPING_ANY, 0),
        {}
};
```

> Please try to vertically align these:
> Please make this look like all the other lists and make an attempt to
> vertically align things.

**Detection signal**: 3+ consecutive lines with similar shape but
unaligned columns. Especially: x86 CPU-ID match tables, struct
initializers in static tables, parallel `start = ...; end = ...;` pairs.

**REPORT as subjective regressions**: tabular code with unaligned
columns where alignment is straightforward.

## Patch decomposition

Series structure is review surface. A series that mixes refactoring with
new features in a single patch, or that scrambles the canonical ordering,
is harder to review patch-by-patch and harder to bisect.

### SR-DH-DEC-1: Don't combine code moves with new functionality

A patch that says "consolidate X and add new Y" must be split: a pure
move/rename patch first, the new functionality second. The pure move is
a no-op behaviorally, easy to verify; the new feature is then reviewable
in isolation.

> I really prefer that code moves and introduction of new things be done
> _separately_. It's a lot easier to check for errors in the move when
> it's the only thing going on.

**Detection signal**: a single patch contains both a large rename/move
hunk and net-new behavior. Subject contains both "Consolidate" or "Move"
and "Add" or "Introduce".

**REPORT as subjective regressions**: patches mixing pure code moves
with new behavior.

### SR-DH-DEC-2: Patch ordering: refactor → infrastructure → feature

When a series introduces a new feature that requires both refactoring of
existing code and new infrastructure, the canonical patch ordering is:

1. Refactor or fix existing code to prepare it
2. Add new infrastructure (helpers, types, headers)
3. Add the new feature itself

Series that put the feature patch at position 1-3 while later patches
refactor the code it depends on are out of order.

> 1. Refactor/fix existing code  2. Add new infrastructure  3. Add new
> feature

**Detection signal**: cover-letter patch list contains a feature patch
with later patches doing prerequisite refactoring.

**REPORT as subjective regressions**: series with feature patches
preceding their prerequisite refactor patches.

### SR-DH-DEC-3: Big series must be broken into chapters

A series touching multiple distinct subsystems (mm cleanups + ABI
plumbing + KVM glue) under a single cover letter is harder to review than
the same work split into chapter series — even if there is a functional
dependency between the chapters.

> Let me know if anyone feels differently, but I really think the 'TDX
> Host Extensions' need to be reviewed as a different patch set.
> But I do think there's some value in breaking this series up into
> pieces that are relatively unrelated, even if there is a functional
> dependency between them.

**Detection signal**: cover letter diffstat shows several distinct
subsystems being touched simultaneously, or 5+ patches do logically
separable work.

**REPORT as subjective regressions**: large series that bundle
logically separable chapters.

### SR-DH-DEC-4: A single patch doing >3 logical things needs splitting

When the subject reduces to "Implement $FOO by making miscellaneous
changes", that's diagnostic of a patch that should have been 4 or 5
patches.

> I suspect it's because the patch is doing about 15 discrete things and
> it's impossible to write a subject that's anything other than some
> form of: x86/virt/tdx: Implement $FOO by making miscellaneous changes
> ... So it's a symptom of the real disease.
> This patch is getting waaaaaaaaaaaaaaay too long. I'd say it needs to
> be 4 or 5 patches, just eyeballing it.

**Detection signal**:
- Single patch touches >5 functions or files in unrelated ways
- Subject contains generic verbs ("Implement", "Add support for") with
  no narrow object
- Patch body diff exceeds ~300 lines without being a verbatim move

**REPORT as subjective regressions**: oversized single patches that need
splitting.

## API design

### SR-DH-API-1: Don't take const pointers when callers have writable ones

A new public API that takes `const T*` while every existing call site
has `T*` and would need a cast to call it is broken. Either change the
API to take a writable pointer, or document why the const matters and
ensure callers naturally have const objects.

> If the API takes a const pointer that requires callers to cast it, I
> think the API is broken.
> Most callers are going to have a pointer that they've been modifying.
> They're not going to have a ptdesc handy.

**Detection signal**: new API takes `const T*` while every existing call
site holds a writable `T*`. Or: API takes a descriptor type (`ptdesc`)
when callers naturally have the underlying type (`page`).

**REPORT as subjective regressions**: APIs forcing callers to cast.

### SR-DH-API-2: Common defaults belong inside the helper

If every caller of a new helper passes the same flag, bake the flag into
the helper. Don't make every call site duplicate it.

> What kind of maniac is ever going to allocate page tables without
> __GFP_ZERO? __GFP_ZERO really should be a part of pagetable_alloc(),
> don't you think?

**Detection signal**: every call site of a new helper passes the same
boolean, flag, or constant.

**REPORT as subjective regressions**: helpers with vestigial parameters
that all callers set the same way.

### SR-DH-API-3: Asymmetric pairs of helpers must share naming

Two functions doing the same operation under different names confuse
readers. Either consolidate or make the names obviously sibling-ish.

> It is *NOT* clear that: tdx_clear_page() and reset_tdx_pages() are
> doing the exact same thing. ... Worst case, have two functions that
> have similar names to make it clear that they are doing the same
> thing.

**Detection signal**: two helpers in the same subsystem with unrelated
names but similar bodies.

**REPORT as subjective regressions**: sibling helpers without sibling
names.

## Type discipline

### SR-DH-TYP-1: Use `struct page *` for kernel-side physical memory

Hardware ABIs that traffic in `u64` physical addresses can lead to
"`u64`s with no type safety" all over the kernel side of an interface.
For kernel-side variables that represent allocated physical memory,
prefer `struct page *` — it's unambiguous about which address space and
can't be silently confused with a virtual address, PFN, or KeyID.

This is a preference, not an absolute rule. The TDX module ABI itself
takes `u64`s; convert at the boundary.

> It allows an unambiguous reference to normal, (mostly) allocated
> physical memory. ... This is one place to bring a wee little bit of
> order to the chaos.
> The 'use struct page * instead of u64 for physical addresses' thingy
> is a good pattern, not an absolute rule.

**Detection signal**: a new TDX/SEAMCALL/coco helper *introduces* a
`u64` or `phys_addr_t` argument when the underlying object is a
kernel-allocated page.

Do NOT fire this rule when the patch *removes* a `struct page *`
argument and replaces it with `pfn_t` / `kvm_pfn_t` / `u64` with a
stated rationale (e.g., "the SEAMCALL is a glorified PTE write and
doesn't need the page reference"). The rule is about *introducing* the
weaker type, not *converting away from* the stronger one.

**REPORT as subjective regressions**: kernel-side helper signatures
that newly introduce `u64` for physical memory when `struct page *`
would carry more meaning. Skip when the patch is a deliberate
conversion *away* from `struct page *` with rationale.

### SR-DH-TYP-2: Don't conflate hardware width with kernel type

Just because the hardware ABI says HKID is 16 bits doesn't mean kernel
code should pass it as `u16`. Pick a kernel type that's convenient
(often `int`); convert at the ABI boundary. Casts like `(u16)ret` at
many sites are the symptom.

> It can also make a lot of sense to pass around a 16-bit value in an
> 'int' in some cases. Think about NUMA nodes. ... I'd personally
> probably just keep 'hkid' as an int everywhere until the point where
> it gets shoved into the TDX module ABI.

**Detection signal**: function signatures use `u8`/`u16` for identifiers
passed across the kernel and cast at every call.

**REPORT as subjective regressions**: ABI-width types propagated through
kernel-side code paths.

### SR-DH-TYP-3: Bitfields are dangerous when the value gets passed around

C bitfields used to model a hardware encoding that will be passed around
as a `u64` (and shifted, masked, transmitted) introduce subtle compiler
and ABI quirks. Prefer explicit shift/mask helpers.

> This is functionally OK, but seeing bitfields on a value that's
> probably going to get shifted around makes me nervous because of:
> [link]
> Look at the kernel page table management. Why don't we use bitfields
> for _that_?

**Detection signal**: new C bitfield types meant to model a hardware
encoding that will be passed around as a `u64`.

**REPORT as subjective regressions**: bitfields modeling hardware
encodings that flow through kernel call paths.

## Init ordering

### SR-DH-INIT-1: Separate "populate the data" from "enable enforcement"

For features where some state must be ready early (so APs can use it)
but enforcement can come late, split the setup function into two:

1. Populate the value from the boot CPU so secondaries can use it
2. Enable the static key so enforcement is active

> split up the [setup_cr_pinning()] code into its two logical pieces:
> 1. Populate 'cr4_pinned_bits' from the boot CPU so the secondaries
>    can use it.
> 2. Enable the static key so pinning enforcement is enabled.

**Detection signal**: a `setup_*` init function that both writes a
config value and calls `static_key_enable()` / `static_branch_enable()`.

**REPORT as subjective regressions**: combined populate-and-enforce
init functions where the call ordering forces awkward placement.

## CPU feature detection

### SR-DH-CPU-1: Use `x86_match_cpu()` and `x86_cpu_id[]`, not hand-rolled VFM switches

Hand-rolled `switch (c->x86_vfm) { case INTEL_ATOM_GRACEMONT: ... }` with
microcode-revision checks gets bounced. Use `X86_MATCH_VFM_STEPS()` and
`x86_match_cpu()` against an `x86_cpu_id[]` table instead.

> No, this just isn't how we do these. Please make an x86_cpu_id[] array
> and use x86_match_cpu() on it. You can even match on steppings in
> those.

**Detection signal**: new `switch (c->x86_vfm)` in
`arch/x86/kernel/cpu/intel.c` or similar, when the same matching could be
expressed as an `x86_cpu_id[]` table.

**REPORT as subjective regressions**: hand-rolled VFM switches.

### SR-DH-CPU-2: Microcode-version lists are despised — use synthetic flags

Mitigation gating that hand-codes microcode revision tables gets replaced
with `boot_cpu_has_bug(X86_BUG_OLD_MICROCODE)` plus a synthetic
`X86_FEATURE_*_CLEAR` cap.

> I also despise these microcode version lists. Let's just use:
> boot_cpu_has_bug(X86_BUG_OLD_MICROCODE);

**Detection signal**: numeric `c->microcode >= 0x...` comparisons in
mitigation code; new MCU revision tables embedded in source.

**REPORT as subjective regressions**: hand-coded microcode version
gating in mitigation logic.

### SR-DH-CPU-3: Reference `X86_FEATURE_*` constants in CPUID predicates

A new helper checking `native_cpuid_ecx(1) & BIT(31)` should mention
`X86_FEATURE_HYPERVISOR` somewhere — even as a build-time sanity check.
Hard-coded BIT positions without referencing the named constant lose
their connection to the feature naming infrastructure.

> Could you please put X86_FEATURE_HYPERVISOR in here somewhere? Even if
> it's just (logically): return native_cpuid_ecx(1) &
> (X86_FEATURE_HYPERVISOR & 0x1f);

**Detection signal**: new CPUID-bit predicates that hard-code bit
positions without referencing the existing `X86_FEATURE_*` enum.

**REPORT as subjective regressions**: CPUID feature checks that don't
mention the corresponding named constant.

## Process

### SR-DH-PROC-1: No `vN.1` patches; resend as `vN+1`

Posting `[PATCHv2.1]` two weeks after `v2` breaks `b4` and other tooling
that expects monotonic versioning. Either resend within a couple of
hours of the original post (still effectively the same version) or wait
and resend as the next integer version.

> Please stop doing these vXXX.1 patches. b4 can't deal with them. ...
> It's one thing to post a whole new series 2 hours after the old one.
> It's a very different thing to resend a series in the 2 *weeks* since
> you updated it.

**Detection signal**: subject line contains `[PATCHvN.M]` where M>0.

**REPORT as subjective regressions**: any `vN.M` (M>0) version tag in a
patch subject.

### SR-DH-PROC-2: Don't churn code just to use new infrastructure

Patches whose entire purpose is "convert X to use the new shiny thing"
(scoped guards, `__free()`, `ptdesc`) get pushback unless there's an
independent reason to touch the code. The conversion churn introduces
bugs, makes other people's series harder to merge, and clutters the git
history.

> we're not going to introduce bugs (and this kind of rework *WILL* have
> bugs), make everyone else's code harder to merge, and clutter up the
> history just to move to the newest shiny thing.
> I don't want to grow its use in arch/x86 until it's a wee bit more
> mature.
> Could we please leave out the unnecessary churn from the renames?

**Detection signal**:
- Cover letter / changelog frames the change as "convert X to Y" with no
  other rationale
- Changes have a high churn-to-function ratio
- The justification is "use the new API" or "modernize the code"

**REPORT as subjective regressions**: pure-conversion patches in
arch/x86 without independent justification.

## Socratic style

These patterns are not detection rules; they are diagnostic of *Dave
Hansen's review voice*. Recognizing them in past review threads helps
predict what he'll flag.

### SR-DH-SOC-1: One-line "Why?" responses signal missing rationale

Dave responds to non-obvious decisions with a single-line question:
"Why?", "Why keep the old prototype?", "Why does it need to be done
early?", "How would we determine that?". These are not rhetorical — they
mean the *changelog* failed to motivate the change.

The fix is in the changelog, not in the code: every non-obvious decision
needs an explicit "why" sentence in the body.

**Cross-reference**: `SR-DH-CHG-3`, `SR-DH-CHG-4`.

### SR-DH-SOC-2: "What does this comment mean?" signals vague terminology

When a comment or changelog sentence uses vague or ambiguous nouns
("shared memory", "control", "managed"), Dave will quote it and ask.
Domain-specific terms exist for a reason: prefer "TDX private memory"
over "shared memory" when the latter could mean either shared mappings
or unencrypted memory.

**Detection signal**: vague nouns in comments or changelogs where a
domain-specific term exists.

### SR-DH-SOC-3: "Which one is right?" signals internal inconsistency

When a single patch uses two conventions for the same thing
(`sizeof(*args_array)` in one place, `sizeof(u64)` in another;
`unrestricted` in one place, `nonsensitive` in another), Dave will quote
both and ask for consistency.

**Detection signal**: same logical operation written two different ways
in the same patch.

**REPORT as subjective regressions**: internal inconsistencies within a
single patch.

## Other recurring patterns

### SR-DH-OTH-1: Defensive code for theoretical bugs is rejected

"Add this check in case the TDX module is buggy" is not enough rationale.
Defensive code adds kernel complexity without paying for itself unless
there's a concrete bug being defended against.

> It's quite another thing to add kernel complexity to preemptively
> lessen the chance of a theoretical TDX bug.
> This patch is looking more and more optional. ... I think it should
> be dropped.

**Detection signal**: defensive checks added with rationale "in case the
[external component] does X" but without evidence of X happening.

**REPORT as subjective regressions**: defensive code for theoretical
failures.

### SR-DH-OTH-2: "We need this" without specifics gets bounced

"We need this for production" or "our use case requires this" is met
with "Care to send along a patch?", "What workload?", or "Plain-English
problem statement, please." The rationale must name the workload, the
spec, or the measured behavior.

> Care to send along a patch representing the 'best solution'? That
> should clear things up.
> I'd also just appreciate a blurb on why you care. Why did you bother
> to send this patch?

**Detection signal**: changelog or cover letter cites "we" or a vendor
without specifying use case, workload, or measured behavior.

**REPORT as subjective regressions**: vague-need rationale in changelogs.

### SR-DH-OTH-3: An `Acked-by` from Dave does not mean "all comments addressed"

Dave will give an `Acked-by:` while leaving open nits or questions. A
prior `Acked-by` on a previous version of the series does not mean the
nits from that round have been addressed in the new version.

When reviewing a respin, do not skip-review patches that carry an
`Acked-by:` tag — re-check whether the open nits from the previous round
were addressed.

> Either way: Acked-by: Dave Hansen <dave.hansen@intel.com>
> These are a little ugly, but it's better than defining three more
> helpers that only get used once. Acked-by: ...

**Detection signal**: respin carries `Acked-by` from a prior version,
but the prior thread had open commentary that the new version does not
address.

**REPORT as subjective regressions**: nits left unaddressed across
respin boundaries when the respin carries an `Acked-by`.

## Quick Checks

- **Brain-dead obvious wins**: if a reader has to re-read code to trace
  the flow, restructure. Multiple-read code is how race conditions get
  missed. Prefer plain `if/else` over clever ternaries; prefer
  `if/else if` over a 2- or 3-case `switch`; explain inline rather
  than via "see below".
- **Comments say why**: don't paraphrase the next line; document
  ordering, locking, hardware/spec rationale.
- **No spec-comment preamble**: lead with the substantive fact, not
  "This is called X and is defined in Y".
- **Comment density**: low ratio of explanatory comments in non-trivial
  new code is a flag.
- **Lockdep over comments**: replace "Caller must hold X" with
  `lockdep_assert_held()`.
- **Names match semantics**: `get_*` acquires references; `*_pages`
  returns page counts; erratum helpers carry the bug name.
- **Match established kernel terminology**: when a name is established
  in git history (`TDX module`), don't invent a new variant
  (`TDX-Module`).
- **Variable name drift**: when a patch repurposes an existing
  variable, rename it to match the new meaning.
- **Specific parameter names**: don't use `data`/`size`/`start`/`len`
  for parameters carrying a specific object type — encode the type
  in the name (`tdx_image`, `vmalloc_addr`, `pa_list_len`).
- **No buried renames**: rename patches stand alone.
- **Single-use error gotos are good**: don't refactor them away.
- **`__free()` is not always an improvement**: in arch/x86, default to
  goto unless `__free()` clearly wins.
- **No whole-function indent bumps**: prefer early returns or
  function-scoped guards.
- **Use existing primitives**: `mempool_t`, `apply_to_page_range()`,
  `kpte_to_vaddr()`, `xa_*` before open-coding.
- **No copy-paste**: shared logic gets a shared helper.
- **Vertically align tables**: CPU-ID tables, struct initializers, paired
  assignments.
- **Patch ordering**: refactor → infrastructure → feature.
- **No combined moves + new behavior**: pure moves first, new behavior
  second.
- **Helper defaults**: if every caller passes the same flag, bake it in.
- **`struct page *` over `u64`**: for kernel-side physical memory.
- **`x86_match_cpu()` over VFM switches**: in CPU feature detection.
- **No `vN.1` versioning**: post `vN+1` instead.
- **No new-shiny churn**: don't convert just to use the new API.
