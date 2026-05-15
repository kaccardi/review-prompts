# TDX / coco Review Subsystem Details

This guide loads alongside `x86-tip.md`, `tip-changelog.md`, and
`tip-code-style.md` for any patch touching TDX host code, TDX guest code,
SEAMCALL/SEAMLDR, or the broader confidential-computing infrastructure
(`coco/`, `arch/x86/virt/vmx/tdx/`, `arch/x86/coco/`).

TDX patches sit at a difficult review intersection: they involve a
proprietary firmware ABI (the TDX module), an active spec, and unusual
kernel-host trust boundaries. The patterns below capture recurring
review feedback specific to TDX — most of which Dave Hansen has raised
multiple times across different threads.

## "TDX is not special" is the meta-rule

The most strongly held position in TDX review feedback is that TDX-only
checks at generic kernel boundaries need a kernel-wide rationale, not a
TDX-side rationale. If KVM-TDX wants to handle hardware-poisoned pages
differently from any other page, the question is not "what does TDX
need" but "what should the kernel do for any code path that hits this
case".

> Why is this TDX code so special that PageHWPoison() needs to be
> checked. ... this would be the *ONLY* arch/x86 site. Why?
> How is this any different from any other kind of hardware poison? Why
> should this specific kind of freeing ... operation be different from
> any other kind of free?

### SR-DH-TDX-5: TDX-only checks at generic boundaries need a kernel-wide rationale

**Detection signal**: TDX-specific code adds a check or special-case at
a generic kernel boundary (page free, MCE handling, kexec, page-table
walk) that doesn't have equivalents in non-TDX paths handling the same
logical operation.

Look for:
- `is_tdx_*()` or `boot_cpu_has(X86_FEATURE_TDX_*)` checks at non-TDX
  call sites
- `PageHWPoison()` checks added only on the TDX path
- Special-case handling of TDX-private memory at sites that don't
  handle SEV/SME-encrypted memory analogously

**REPORT as subjective regressions**: TDX-specific code at generic
kernel boundaries without an explanation of why the same treatment isn't
warranted for other confidential-computing or non-confidential paths.

## SEAMCALL error handling

SEAMCALL (and TDCALL) return TDX module status codes in `u64`. The kernel
side has to convert these to `-EINVAL` / `-EBUSY` / `-EIO` / etc. for
its own callers. The error-mapping logic must live in one place — not at
every wrapper, and certainly not at the leaf call sites.

```c
// WRONG: every wrapper open-codes status mapping
int tdh_mem_page_aug(...)
{
        u64 ret = __tdcall(TDH_MEM_PAGE_AUG, ...);

        if (TDCALL_RETURN_CODE(ret) == TDX_OPERAND_BUSY)
                return -EBUSY;
        if (TDCALL_RETURN_CODE(ret) == TDX_PAGE_ALREADY_ACCEPTED)
                return -EEXIST;
        if (ret)
                return -EIO;
        return 0;
}

// CORRECT: one shared helper does the mapping
int tdh_mem_page_aug(...)
{
        u64 ret = __tdcall(TDH_MEM_PAGE_AUG, ...);

        return tdx_err_to_errno(ret);
}
```

> This a pretty ugly switch statement. ;) ... it would be _nice_ if
> these could eventually look like: err = __tdcall(...); return
> tdx_err_to_errno(err);
> I'd kinda prefer that we just shove the TDX error codes as far up
> into the helpers as possible rather than making them easier to deal
> with in random code.

### SR-DH-TDX-1: SEAMCALL/TDCALL status mapping must be centralized

**Detection signal**:
- New SEAMCALL or TDCALL wrapper that contains an `if/else` chain or
  `switch` on `TDCALL_RETURN_CODE(ret) == TDX_*` constants
- Callers of TDX wrappers that receive raw `u64` status codes and decode
  them themselves
- Multiple wrappers in the same file each open-coding the same mapping

**REPORT as subjective regressions**: open-coded TDX status-to-errno
conversions duplicated across wrappers.

## KeyID and physical address must stay separate

TDX pages carry a KeyID in their physical address: the high bits encode
the KeyID, the low bits the actual physical page. The temptation is to
combine `paddr | (keyid << x86_phys_bits)` into a single `u64` and pass
it around. Don't.

The KeyID and the physical address are conceptually different things:
the kernel's data structures should reflect that. MCE records, page
descriptors, and interfaces should carry KeyID and paddr as separate
fields. Mask off the KeyID at the *earliest* point in the call chain.

> Uhhh, just store the KeyID separately. Have mce->addr and mce->keyid.
> Problem solved.
> if we take it back out, I'd expect it fixes more things than it
> breaks.

### SR-DH-TDX-2: Don't pass combined keyid|paddr through kernel code paths

**Detection signal**:
- Code that ORs a KeyID/HKID into a paddr without later masking
- ABI structures (e.g. MCE records) carrying the combined value
- Function signatures that take a single `u64` argument named in a way
  that suggests it carries both KeyID and paddr (`tdx_paddr`,
  `keyed_addr`)

**REPORT as subjective regressions**: combined keyid|paddr values
flowing through kernel data structures or call signatures.

### SR-DH-TDX-3: Conversion idioms (paddr ↔ keyed-paddr) belong in one helper

If the same `paddr |= keyid << boot_cpu_data.x86_phys_bits` idiom
appears more than once, it needs a helper.

> I've seen this idiom enough times. You need a helper.

**Detection signal**: 2+ instances of `... << boot_cpu_data.x86_phys_bits`
or other paddr/keyid assembly across files in the same series.

**REPORT as subjective regressions**: KeyID-to-paddr assembly idioms
duplicated across files.

## Erratum and quirk handling

TDX has hardware errata (partial-write MCE, MOVDIR64B memory ordering,
etc.) that require workarounds. The workaround code must be:

1. Quarantined behind a single function whose name encodes the erratum
   or quirk
2. Not sprinkled across the tree at every call site that might be
   affected

A function that runs only on CPUs with `X86_BUG_TDX_PW_MCE` and is
called `reset_tdx_pages()` looks like a normal reset operation in the
caller — that's a defect, even if the body is correct.

### SR-DH-TDX-4: Erratum workarounds need the bug name in the function name

> It is *NOT* clear that: tdx_clear_page() and reset_tdx_pages() are
> doing the exact same thing. ... If the only possible use for
> reset_tdx_pages() is for the erratum, then it needs to be in the
> function name.

Suggested style: `tdx_quirk_reset_paddr()`, `tdx_pw_mce_clear_page()`,
`tdx_movdir64b_memcpy()`.

**Detection signal**:
- New function whose body is gated only on `boot_cpu_has_bug(X86_BUG_TDX_*)`,
  `X86_QUIRK_*`, or similar
- New function whose only call sites are workaround paths
- Function name doesn't include `quirk`, the bug name, or another
  erratum-encoding token
- Two functions with similar bodies, one a workaround for the other

**REPORT as subjective regressions**: erratum-only helpers without the
erratum encoded in the name; sibling workaround helpers without sibling
naming (cross-reference: `SR-DH-API-3`).

## TDX cover letters

TDX series cover letters that recap the TDX module spec ("This series
implements TDH_SYS_RD and adds support for TDH_VP_FLUSH ...") without
explaining what kernel users gain bounce immediately. The cover letter
must lead with a kernel-side problem statement.

### SR-DH-TDX-6: TDX cover letters must motivate the kernel-side problem

> Plain, acronym-free language when possible would be much appreciated
> there and across this series. For instance, there's not even a
> problem statement to be seen.

**Detection signal**:
- Cover letter body consists of ABI quotes / module feature names
- No plain-English problem framing in the first paragraph
- TDX module spec terminology (`TDH_*`, `TDG_*`, register names) used as
  load-bearing prose

**REPORT as subjective regressions**: TDX cover letters lacking a
plain-English problem statement.

This is a stronger form of `SR-DH-CHG-3` (changelogs explain the
kernel-side problem) applied to the cover-letter level.

## Don't lock kernel correctness to TDX module behavior

Designs that bind kernel correctness to current TDX module
locking/behavior are fragile: the TDX module is firmware, it can be
updated, and a kernel that depends on its current internals breaks when
the module changes.

> Yeah, but what if the TDX module changes its locking scheme in 5
> years or 10? What happens to 6.17.9999-stable?

### SR-DH-TDX-7: Don't depend on TDX-module behavior, and don't impose user-visible costs because of it

This rule has two faces.

**Face 1 — kernel correctness binding to TDX module internals**:
locking, ordering, or correctness documentation that references
"matching the TDX module's locks", or that justifies skipping
kernel-side checks because "the TDX module guarantees X".

**Face 2 — user-visible UX costs imposed because the TDX module
requires them**: forcing all CPUs online forever, forcing memory to
stay contiguous, forcing kdump/kexec disabled, forcing a single
in-flight operation. These constraints are felt by users, not just by
TDX paths, so they need a *long* justification — there are very good
reasons to leave CPUs offline forever, to kdump, to use memory the way
the rest of the kernel uses it.

> Yeah, but what if the TDX module changes its locking scheme in 5
> years or 10? What happens to 6.17.9999-stable?
> There needs to be a *LONG* justification why there is no other choice
> here. There are very good reasons to leave CPUs offline forever.

**Detection signal — Face 1**: kernel locking, ordering, or
correctness documentation referencing TDX module internals.

**Detection signal — Face 2**: patch introduces a user-visible
constraint (refuses operation when CPUs are offline, blocks kexec/
kdump, requires contiguous memory, serializes a previously concurrent
operation) where the rationale is "the TDX module / firmware needs
it". Hallmark phrases in changelog: "P-SEAMLDR requires", "TDX module
requires", "must hold all CPUs online", "block kexec because".

**REPORT as subjective regressions**:
- Kernel correctness arguments that rely on TDX module internal
  behavior remaining stable (Face 1).
- User-visible kernel constraints justified only by "the TDX module
  needs it", without an extended discussion of why no kernel-side
  workaround was possible (Face 2).

## TDX module internal terminology vs kernel-facing names

The TDX module ABI uses names like `TDH.MEM.PAGE.AUG`, `TDH.SYS.RD`, and
metadata field IDs like `0x90000010`. These belong inside the SEAMCALL
wrapper or in a single comment near the call site. They do not belong
in:

- Function names exposed to the kernel side (use kernel-flavored names:
  `tdx_aug_page()`, not `tdh_mem_page_aug()` for callers)
- Changelog or cover-letter bodies (cross-reference: `SR-DH-CHG-3`,
  `SR-DH-TDX-6`)
- Code comments that callers will read (the wrapper hides the ABI; its
  callers should not need to know the ABI name)

Subject lines should describe the kernel behavior, not the SEAMCALL
being added (cross-reference: `SR-DH-SUBJ-1`, `SR-DH-SUBJ-2`).

## Don't sprinkle TDX-specific knowledge across the tree

Knowledge of TDX-specific quirks (MOVDIR64B memory ordering, KeyID
encoding, `X86_BUG_TDX_*` workarounds) must live in one place. When the
same TDX-specific check appears at multiple sites, the abstraction is
broken — consolidate.

> Could we consolidate them, please? There's no reason to sprinkle
> knowledge of movdir64b's memory ordering rules all across the tree.

This is the TDX-specific application of `SR-DH-STR-7` (don't copy-paste,
refactor) and `SR-DH-STR-5` (hide complications inside helpers).

**Detection signal**: TDX-specific behaviors (MOVDIR64B, KeyID encoding,
SEAMCALL retry logic, partial-write MCE) referenced at >1 site outside a
single dedicated helper.

**REPORT as subjective regressions**: TDX-specific knowledge appearing
at multiple sites that should share a helper.

## Quick Checks

- **TDX is not special**: any TDX-only check at a generic kernel
  boundary needs a kernel-wide rationale, not a TDX rationale.
- **Centralize SEAMCALL error mapping**: `tdx_err_to_errno()` exists (or
  should); call sites take `int`, not raw `u64` status codes.
- **Separate KeyID from paddr**: ABI structures carry both as separate
  fields; conversion happens at one place via a helper.
- **Erratum names in function names**: `tdx_quirk_*`, `tdx_pw_mce_*` —
  not generic verbs.
- **Cover letters**: kernel-side problem statement first, ABI names only
  in service of the problem statement.
- **Don't depend on TDX module internals**: kernel correctness must hold
  even if the module's internal locking changes.
- **User-visible UX costs need extraordinary justification**: forcing
  CPUs online, blocking kexec, requiring contiguous memory because
  "TDX module needs it" requires a *long* changelog rationale.
- **Hide TDX module ABI names**: kernel-facing function names use
  kernel-flavored names; ABI names live inside wrappers.
- **Consolidate TDX-specific knowledge**: MOVDIR64B, KeyID, retry logic,
  errata — one helper each, not sprinkled across files.
