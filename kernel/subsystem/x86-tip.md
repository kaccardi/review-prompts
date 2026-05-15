# x86 / tip-tree Subsystem Details

This guide loads for any patch touching `arch/x86/`. It captures the tip-tree
process expectations from `Documentation/process/maintainer-tip.rst` and the
review preferences of the x86 maintainers, with particular emphasis on
patterns Dave Hansen consistently enforces.

Patches that violate these rules cycle through review for weeks. The patch is
not technically broken — but reviewers spend their time correcting form
instead of evaluating substance, and the series stalls.

## When this guide applies

This is the entry point for tip-tree reviews. When the patch touches
`arch/x86/`, also load:

- `tip-changelog.md` — changelog and commit-message rules
- `tip-code-style.md` — coding-style rules beyond CodingStyle that the tip
  maintainers enforce
- `tdx-review.md` — additional rules for TDX/coco patches (load when the
  patch touches TDX, SEAMCALL, SEAMLDR, or `coco/`)
- `tip-citations.md` — lore message-id citations for any SR-DH-* finding

## Subject prefix

The tip tree uses `subsys/component:` prefixes such as `x86/apic:`,
`x86/mm/fault:`, `sched/fair:`, `genirq/core:`. Filenames or paths are not
prefixes. Run `git log <path>` to see what prefix the file's prior commits
used and follow that convention.

The condensed description after the colon starts with an uppercase letter
and is in imperative tone (`Add`, `Fix`, `Remove` — not `Adds`, `Adding`,
`Added`, `This patch ...`).

When a function name appears in the subject, use `function_name()` form
(with parentheses). `Make foo() static` is unambiguous; `Make foo static`
could mean a variable.

### SR-DH-SUBJ-1: Subject must convey the behavior change, not just the new symbol

Subjects of the form `Add tdx_<foo>()` or `Introduce <FOO>` whose entire
description is the new function/macro name communicate nothing about the
patch's effect. Reviewers cannot tell whether the patch is plumbing,
behavior, or refactoring without opening it.

A subject like `Implement $FOO by making miscellaneous changes` is a tell
that the patch needs to be split — see `SR-DH-DEC-4` in this guide.

**Detection signal**: subject after the colon is mostly an identifier
(`tdx_get_pamt_refcount`, `__tdh_mem_page_aug`) with no verb describing
the behavior. Plain English alternatives are usually possible.

Down-weight confidence when:
- The identifier is a recognized hardware/firmware operation name
  (e.g., `Add SEAMCALL wrapper for TDH.SYS.RD` — `TDH.SYS.RD` is the
  spec-defined operation, naming it conveys real meaning)
- The patch is a thin wrapper for a single named operation, where
  "Introduce a wrapper for X" is the most informative description
- The subject already starts with a verb describing the kernel-side
  effect ("Cache X", "Allocate Y") even when the object is a symbol

**REPORT as subjective regressions**: subjects whose description is just
the symbol being introduced AND the symbol is a kernel-internal helper
(not a hardware/firmware operation name).

### SR-DH-SUBJ-2: No marketing terms in subjects

Subjects relying on Intel/vendor product names (`TDX Connect`, `Trust
Domain`, `Total Memory Encryption`) get rewritten. The kernel is not a
marketing surface. Use the kernel feature name (e.g. `PCI/TSM:` for the
underlying mechanism) or describe the kernel-side effect.

**REPORT as subjective regressions**: subjects whose only meaning comes
from a vendor brand or marketing term.

## Cc and routing rules

`x86@kernel.org` is a mail alias, not a list. A patch sent only to
`x86@kernel.org` reaches private inboxes and never appears on the archive.
Patches touching `arch/x86/` must Cc both `x86@kernel.org` and
`linux-kernel@vger.kernel.org`. The MAINTAINERS file does not always call
out the x86 maintainers — Cc them on any `arch/x86/` patch even when
get_maintainer.pl does not.

Bug fixes targeting mainline must apply against mainline, not against the
tip tree. Maintainers handle conflicts when the fix lands in tip.

Pull requests are accepted only from sub-maintainers feeding trees into
tip. New patch submissions must go through the mailing list — pull
requests for new patches are rejected.

## Branch selection

Develop against the head of the relevant tip branch:

- `x86/*` branches for x86 development (note: x86 KVM and x86 Xen go via
  their own subsystems and only Cc x86 maintainers)
- `sched/core` for scheduler
- `locking/core` for locking and atomics
- `irq/core` for generic IRQ
- `timers/core` for time, timers, NOHZ
- `perf/core` for performance counter core
- `efi/core` for EFI core
- `core/rcu` for RCU
- `ras/core` for RAS

For subsystems aggregated into tip from a separate maintainer tree (perf
tools, irqchip drivers, clocksource drivers), develop against that
maintainer's tree, not against tip itself.

## Merge window

The tip trees close to all but urgent fixes during the merge window. Don't
expect review or merging during this period. Large series should be
submitted in mergeable state at least a week before the merge window opens.

## Testing

Anything beyond minor changes must be built, booted, and tested with the
heavyweight kernel debugging options enabled:

```
make x86_debug.config
```

These options live in `kernel/configs/x86_debug.config`. Patches that turn
out to fail with debug options enabled trigger immediate bounce.

## Tag ordering

The tip maintainers use a strict tag order. From the top of the trailer
block to the bottom:

1. `Fixes:` — 12+ char SHA, subject in parens
2. `Reported-by:`
3. `Closes:` — bug report URL or message-ID
4. `Originally-by:`
5. `Suggested-by:`
6. `Co-developed-by:` / `Signed-off-by:` pairs
7. `Signed-off-by:` (author)
8. `Signed-off-by:` (handler, if different)
9. `Tested-by:`
10. `Reviewed-by:`
11. `Acked-by:`
12. `Cc:`
13. `Link:`

Combined tags (`Reported-and-tested-by`) break automated extraction and are
forbidden. `Co-developed-by` and `Signed-off-by` for the co-author must
appear as a pair.

If a handler modified the patch, the modification notice goes between the
changelog body and the tags, separated by two blank lines on each side:

```
... changelog text ends.

[ handler: Replaced foo by bar and updated changelog ]

Fixes: ...
Signed-off-by: ...
```

If a handler is sending on behalf of the author, the first line of the
changelog is `From: Author <author@mail>` followed by an empty line.

### SR-DH-AUTH-1: Long author chains warrant a contributor note

When a patch carries a long author chain — `Co-developed-by:` tags or
unpaired `Signed-off-by:` tags from non-handlers — reviewers cannot
tell who did what. The silent assumption that "everyone contributed
equally" usually isn't accurate. Add a brief `[note: ...]` block
above the tag block (in the same style as the handler-modification
notice) summarizing who contributed what.

> OK, yes, that's a long chain of developers, a note probably is
> warranted at this point.
> This SoB chain at _least_ needs a note. It looks quite bizarre.

**Detection signal**: any of the following without a preceding
`[note: ...]` block clarifying the contributor split:
- 4+ `Co-developed-by:` trailers
- 3+ `Signed-off-by:` trailers from non-handlers (i.e., not counting
  the trailing handler SoB) with no `Co-developed-by:` pairing
- 4+ author-side trailers in any combination
  (`Co-developed-by` + non-handler `Signed-off-by`)

**REPORT as subjective regressions**: long author chains without a
contributor-note block.

## Fixes: tags are mandatory even for non-stable fixes

The tip maintainers want `Fixes:` tags on any change addressing a previously
introduced issue, including issues that only affect tip or recent mainline
and do not need stable backporting. The tag exists for automated extraction,
not just stable selection. The format is:

```
Fixes: abcdef012345678 ("x86/xxx: Replace foo with bar")
```

Don't burn the lead paragraph announcing the broken commit. Describe the
problem in prose; put the broken-commit reference in a `Fixes:` trailer.

## Quick Checks

- **Cc check**: any patch touching `arch/x86/` must Cc both `x86@kernel.org`
  and `linux-kernel@vger.kernel.org`. Sending only to `x86@kernel.org`
  reaches private inboxes.
- **Subject case**: description after the colon starts with an uppercase
  letter and uses imperative tone.
- **Function references**: in subject and changelog body, function names
  carry parentheses: `function_name()`.
- **Marketing terms**: Intel/vendor product names in subjects get rewritten.
- **Branch routing**: bug fixes apply against mainline, not tip. Sub-maintainer
  pull requests aggregate into tip; new patches go via mailing list.
- **Tag order**: `Fixes` first; `Cc` and `Link` last; no combined tags.
