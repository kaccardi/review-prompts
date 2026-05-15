# tip-tree Changelog Subsystem Details

This guide loads alongside `x86-tip.md` and applies to commit messages on
any patch routed through the tip tree (x86, sched, locking, irq, timers,
perf, RCU, EFI, RAS).

A changelog that fails these rules is the most common reason a tip-tree
patch series cycles for an extra revision. Reviewers — especially Dave
Hansen — respond to changelog problems before reviewing code, so the patch
is held until the message is fixed even when the diff itself would be
acceptable. Think of the changelog as the single most-read part of the
patch; the kernel git history outlives the patch posting forever.

## Voice and structure

Changelogs use **imperative voice** throughout. Write what the patch makes
the kernel do, in present tense, as a command:

```
WRONG:   This patch adds a helper for foo.
WRONG:   We added a helper for foo.
WRONG:   I'm adding a helper for foo.
WRONG:   Adding a helper for foo.
CORRECT: Add a helper for foo.
```

The handbook calls out that abstract/imperative wording is "more precise
and tend to be less confusing" than novelistic prose. The maintainers
enforce this hard.

Structure the body in three paragraphs, in this order:

1. **Context** — what the kernel currently does, what hardware/spec
   constraints apply
2. **Problem** — what is wrong, missing, broken, or about to become a
   problem
3. **Solution** — what this patch changes to address the problem

Don't lump everything into a single paragraph. Don't put the solution
first.

For race conditions, memory ordering issues, or deadlock scenarios, depict
the parallelism with a CPU0/CPU1-style table:

```
CPU0                       CPU1
free_irq(X)                interrupt X
                           spin_lock(desc->lock)
                           wake_irq_thread()
spin_lock(desc->lock)
remove_action()
shutdown_irq()
release_resources()        thread_handler()
spin_unlock(desc->lock)      access released resources.
                             ^^^^^^^^^^^^^^^^^^^^^^^^^
synchronize_irq()
```

This visual form is much easier to follow than prose.

### SR-DH-CHG-1: "This patch ..." is forbidden

Imperative voice means no "This patch ...", no "We ...", no "I ...", no
"Let's ...". Sentence-initial "Adds", "Adding", "Added" are also wrong.
Dave Hansen will return patches over this single issue, sometimes with
just the response "No 'this patch', please. Imperative voice, please."

**Detection signal**: changelog body contains any of:
- `^This patch ` at the start of a sentence
- `^(We|I|Our|Let's) ` at the start of a sentence
- `^(Adds|Adding|Added|Fixes|Fixing|Fixed) ` at the start of a sentence
  (sentence-leading present-participle/past-tense forms — note that
  `Fixes:` as a *trailer* is correct)
- "the entity can also be used for other purposes. Let's rename it" — drop
  the "Let's"

**REPORT as subjective regressions**: any changelog using "this patch" or
sentence-leading non-imperative forms.

## Function references

When a function name appears in the changelog body or subject, write it
with parentheses: `function_name()`. Bare identifiers can be ambiguous —
`reservation_count` could be a variable; `reservation_count()` is
unambiguously a function.

This applies even when the surrounding sentence makes the meaning
"obvious". The maintainers want a uniform rule.

## Spell-check

Spelling errors in changelogs are not minor. They are visible in the git
history forever, they suggest the author did not re-read the message, and
when they recur across versions of the same series they become a process
problem. Dave Hansen escalates spelling complaints by name when they
recur:

> Could you please break out the spell checker before posting v2?
> ... Vishal! I'm noticing spelling issues right up front and center
> here, just like in v1. Any chance you could actually put some spell
> checking in place before v3?

Run a spell checker on every changelog before posting. Common offenders
that flag immediately: `inatomic`, `preceeding`, `idefinite`,
`seomthing`, `infromation`.

### SR-DH-CHG-2: Spelling errors in the changelog

**Detection signal**: standard dictionary misspellings in the changelog
body. Subject lines should also pass.

**REPORT as subjective regressions**: any spelling errors found in the
changelog.

## Explain *why*, not *what* the diff already shows

The diff shows what changed. The changelog must explain why the kernel
needs this change. Reviewers reach for "Why?" the moment a non-obvious
decision is not motivated in the body.

Specifically, for tip-tree patches:

- The first paragraph must answer: what kernel-side problem does this
  solve? Not: what does the new code do?
- Every non-trivial design decision (extra wakeup, retained field,
  unusual init order, special case) needs an explicit motivation.
- "We need this for our use case" is not a rationale. Name the workload,
  the spec, or the measured behavior.

### SR-DH-CHG-3: Changelog reads like an ABI translation

TDX, SEV, CET, and similar features have heavy spec terminology
(`TDH_MEM_PAGE_AUG`, `TDG_VP_ENTER`, `XFEATURE_CET_S`, MSR names, hex
metadata field IDs). Changelogs that read like a translation of the spec
fail the "explain why" test even if they are technically descriptive.

The kernel rationale comes first; spec names appear (if at all) in service
of the rationale, not as the rationale.

> I don't need to read the ABI names in the changelog. I *REALLY* don't
> need to read the TDX documentation names for them. ... If *ANYTHING*
> these names should be trivially mappable to the patch that sits below
> this changelog.

**Detection signal**: changelog body contains TDX/SEV/CET module function
names (`TDH_*`, `TDG_*`, `SEV_*`, `XFEATURE_*` constants), spec section
numbers, or hex field IDs as load-bearing prose. The kernel-side problem
statement is missing or relegated to one sentence at the end.

**REPORT as subjective regressions**: changelogs that lead with spec
terminology and do not state a kernel-side problem in the first paragraph.

### SR-DH-CHG-4: Long changelog, no rationale

A 3-paragraph changelog where every paragraph describes mechanics ("the
code does X then Y then Z") but no paragraph describes a user-visible
problem, performance/security/correctness rationale, or hardware/spec ask.

> I feel like these changelogs are long but say very little.
> This changelog is pretty rough. It's got a lot of words but not much
> substance.

**Detection signal**: changelog body is 8+ lines AND every paragraph
describes diff mechanics AND no sentence matches one of: "This is needed
because ...", "Without this, ...", "Currently, ... which is broken
because ...", "<workload> requires ...", "This eliminates
<measured-cost>", "<feature> introduces <capability> for <user>". A
changelog stating a concrete capability (e.g. "expose X to userspace via
sysfs so foo can do Y") satisfies the rationale requirement and should
NOT fire this rule.

A heavily-negative diffstat (more removals than additions) also weakens
the signal — pure-cleanup changelogs often don't need extensive
rationale.

**REPORT as subjective regressions**: changelogs that paraphrase the diff
without articulating a kernel-side problem, capability, or measurable
impact.

### SR-DH-CHG-5: Resource costs must be in the changelog itself

When a patch consumes memory, code size, or runtime overhead, the
quantification must be in the changelog body — not just in the cover
letter, not just in a follow-up email, not silently. Dave Hansen has
asked for this repeatedly:

> Please mention the 0.4%=>0.0004% overhead here in addition to the
> cover letter. It's important.
> This patch *WASTES* resources. Granted, it's only for a single patch,
> but it's totally not obvious.

**Detection signal**: patch adds `vmalloc()`, `alloc_pages()`,
`kmem_cache_*`, large statics, large per-CPU/per-VM/per-task fields, or
extends an XSAVE / FPU / SEAMCALL buffer, but the changelog has no number
quantifying the cost.

**REPORT as subjective regressions**: changelogs for resource-consuming
patches that do not state the cost (in bytes, percent, or pages).

## Don't let drive-by changes go unmentioned

A patch with one stated purpose plus several unrelated drive-by changes
("while at it I also tweaked X") forces reviewers to discover the extra
changes by reading the diff carefully. Either justify each hunk in the
changelog, or split the unmentioned hunks into separate patches.

### SR-DH-CHG-6: Hunk not mentioned in the changelog

> This 'pgd_changed' hunk isn't mentioned in the changelog.
> What's with the #ifdef munging? It's not mentioned in the changelog.

**Detection signal**: hunks in the diff that touch files or functions
outside what the changelog describes. Particularly common: `#ifdef`
restructuring, signature tweaks, whitespace changes, or rename
substitutions added "while at it".

**REPORT as subjective regressions**: any patch where the diff includes
significant hunks not mentioned in the changelog.

## Don't put kerneldoc-style content in the changelog

Bullet lists or tables documenting struct fields belong in the kerneldoc
block above the struct, not in the changelog. The changelog is read once
when reviewing the patch; the kerneldoc is read every time someone reads
the struct.

### SR-DH-CHG-9: No forward-looking statements

Don't reference work that hasn't landed yet. "This will be used by the
upcoming X series" or "in preparation for Y, which will be posted
next" promises something the reader cannot verify and ages badly: if
the future series never lands, the changelog reads as a stub for
something that never happened.

Live in the present. Describe what *this* patch does, in the kernel as
it exists today. If a future user is genuinely necessary to motivate
the change, the future series should be posted first (or at least
included in the cover letter as part of the same submission), not
referenced as a vague intent.

This applies to code comments as well as changelogs. Comments saying
"will be used by future foo()" rot the same way changelog references
do.

**Detection signal**: changelog body or code comments contain phrases
like:
- `in preparation for ...` (referencing un-posted work)
- `will be used by ... [some future patch]`
- `to support an upcoming ...`
- `will need ... when we add ...`
- `lays the groundwork for ...`

A reference to a series **also being posted** in the same submission
(cover-letter / sibling patches in the same series) does not fire this
rule — those land or fail together, so the reference does not rot.

**REPORT as subjective regressions**: changelogs or comments
referencing work that is not part of the current submission.

### SR-DH-CHG-10: Avoid phrasings that force the reader's brain to leap

Several specific phrasings recur in Dave's review feedback as causes
of cognitive friction. The reader's job is to understand the change;
the writer's job is to remove obstacles.

Avoid:

- **`etc.`** — implies "more that I couldn't be bothered to write
  down". Either enumerate the items that matter or generalize: write
  "MSRs in the IA32_X2APIC_* range" rather than "MSRs like
  IA32_X2APIC_APICID, etc.".
- **`e.g.`** — same problem in a different costume. State the example
  inline: "the TDX module" rather than "a kernel-loaded firmware,
  e.g., the TDX module".
- **`former` / `latter`** — forces the reader to scan back to figure
  out which thing is which. Just name the thing again.
- **`unlike X, ...`** at the start of a sentence — assumes the reader
  has X already loaded. State the property being introduced; mention
  the contrast only if the contrast itself is the point.
- **Forward-reference prose** like `not suitable for X (see below for
  reasons)` — the reader's brain leaps forward and gets thrown when
  the "below" turns out to be three paragraphs away. State the reason
  inline.
- **Vague assertions** without a backing reason — `this is not
  suitable`, `this is the wrong place`, `this needs adjustment` with
  no concrete fault behind them. The fault must accompany the
  assertion.
- **Sentence-leading vague quantifier `Some`** when the patch could
  enumerate. "Some platforms" / "Some users" should become a list
  with the actual platforms or use cases, not a hand wave.
- **The `X is:` colon-then-bullet construct** — `Foo is:` followed
  by a bulleted list reads awkwardly. Either inline the items
  ("Foo is bar and baz") or restructure as `Foo:` heading the list
  without the verb.

> not suitable [...] makes your brain leap around. ... we are saying
> it's not suitable without a real reason.
> I hate 'e.g.'.
> Not 'Some'.
> remove from your changelogs: this 'is: ' construct. It's horribly
> awkward.

**Detection signal**: changelog body, cover letter, or non-trivial
code comments contain any of:
- `\betc\.?$` (mid-sentence or trailing `etc.`)
- `\be\.g\.\b`
- `\b(former|latter)\b`
- `^Unlike\b` at the start of a sentence
- `see below` / `as discussed below` / `for reasons described below`
- `not suitable` / `is wrong` / `inappropriate` without an immediately
  following concrete reason
- `^Some\s+\w+` at the start of a sentence (vague quantifier where
  enumeration would be possible)
- `\b\w+\s+is:\s*$` (a sentence ending with `is:` that introduces a
  bullet list)

This rule applies equally to code comments — the same phrasings cause
the same friction in source.

**REPORT as subjective regressions**: any of the listed phrasings,
unless the surrounding text already provides the missing specificity
inline.

### SR-DH-CHG-12: Use concrete examples, not abstract placeholders

When a changelog illustrates a policy or behavior with an example, use
concrete real-world values: actual product names, actual version
numbers, named workloads. Symbolic placeholders (`1.5.x`, `1.5.y+1`,
`vendor X`, `module Z`) make the reader hunt for an anchor and convey
nothing inferable from the abstract form.

> Could you try a concrete example, please?
> Again, I'd just give an actual example.

```
WRONG:   When module 1.5.x is upgraded to 1.5.y+1, the platform
         transitions from state A to state B.
CORRECT: When the TDX module on Sapphire Rapids is upgraded from
         1.5.6 to 1.5.7, the platform transitions from state A to
         state B.
```

**Detection signal**: changelog body contains a worked example using
placeholders like `<vendor>`, `<product>`, `1.5.x`, `1.5.y+1`,
`module Z`, `vendor X`, `\bX\b` / `\bY\b` as standalone tokens
substituting for product names. Real version numbers, real product
names, and real module names should appear instead.

**REPORT as subjective regressions**: changelogs that use abstract
placeholders where concrete example values are available.

### SR-DH-CHG-13: Don't bury the main solution sentence in mid-paragraph

The single sentence that names the patch's primary action ("Add a
wrapper for X", "Parse Y and populate Z") must end its paragraph or
sit alone. When the headline is followed by a secondary nit, or
trails a context clause, the reader's eye stops on the wrong thing
and the change's purpose gets buried.

> what is in a more prominent place in the changelog?
> It's taking the key imperative and burying it.

```
WRONG:
P-SEAMLDR consumes the seamldr_params data structure to install a
TDX module. Parse the struct tdx_image into a seamldr_params layout
and pass it to seamldr_install_module(). Use seamcall_prerr() rather
than seamcall_ret() because we want to log the error code.

CORRECT:
P-SEAMLDR consumes the seamldr_params data structure to install a
TDX module.

Parse the struct tdx_image into a seamldr_params layout and pass it
to seamldr_install_module().

Log error codes via seamcall_prerr() rather than seamcall_ret().
```

**Detection signal**: a paragraph whose imperative-mood "headline"
sentence (Add/Parse/Configure/Allocate/Install) is followed by an
additional sentence in the same paragraph that addresses a secondary
mechanism, nit, or detail. The headline should end the paragraph or
sit alone.

**REPORT as subjective regressions**: changelog paragraphs where the
headline imperative is buried mid-paragraph.

### SR-DH-CHG-11: Kconfig changelogs explain when to disable

When a patch introduces a `Kconfig` option, the commit log should
make clear why someone would turn it off. If the only motivation in
the log is "add a config to enable X", the reader has no signal for
when disabling makes sense. Either the help text or the changelog (or
both) needs to cover the disable case.

> if it's not obvious why someone would want to turn it off (and it
> is actually needed), then, improve commit log.

**Detection signal**: patch adds a new entry in a `Kconfig` file and
the changelog plus the Kconfig `help` block together do not contain a
sentence describing when disabling is appropriate (`disable when ...`,
`leave off if ...`, `requires ...`, `costs ...`).

If the disable case is not just unmentioned but genuinely unrealistic
given the option's `depends on` chain (i.e., disabling the parent
option already disables this one, and nobody would build the parent
without the child), the user-visible *prompt* itself may be
gratuitous. Consider whether the option should be a non-prompted
`def_tristate` / `def_bool` instead of a `tristate "..."` /
`bool "..."` prompt.

**Detection signal — unwarranted prompt**: a new `tristate "..."` or
`bool "..."` prompt whose `depends on` chain implies disabling the
parent disables this one, AND the help text or changelog does not
describe a realistic case where someone would want this on while
having the parent on. In that case, suggest `def_tristate m if
INTEL_TDX_HOST` (or equivalent) rather than a prompt.

**REPORT as subjective regressions**:
- New Kconfig options whose changelog and help text together do not
  motivate the disable case.
- New Kconfig prompts where the disable case is not merely unmentioned
  but genuinely unrealistic — these should be promptless
  `def_tristate` / `def_bool` instead.

### SR-DH-CHG-8: Neutral tone in changelogs

Changelogs describe technical changes; they are not the place to
editorialize about prior author decisions. Adjectives like *misguided*,
*broken*, *no business*, *completely unnecessary*, *useless*, *wrong*
applied to existing kernel code (without a specific technical fault
backing the word) read as judgment rather than analysis.

Folks can have honest disagreements without one side being "misguided."
Either back the strong word with a concrete technical defect, or use
neutral language: "this assumption does not hold for case X" rather
than "this assumption is broken".

> I think this goes a bit too far. ... It's quite another to say what
> other bits of the codebase have 'business' doing.
> Could we maybe tone down the editorializing a bit, please? Folks can
> have honest disagreements about this stuff while not being
> 'misguided'.

**Detection signal**: changelog body contains judgment adjectives
applied to existing code, decisions, or assumptions:
- `misguided`, `broken`, `wrong`, `useless`, `unnecessary`, `pointless`,
  `nonsense`, `silly`
- `no business`, `has no place`, `should never`
- Phrases of the form "X is wrong" / "X is broken" referring to
  existing kernel code without an immediately following concrete
  technical reason

A neutral phrasing of the same critique ("X assumes a property that
doesn't always hold; this patch removes that assumption") does not fire.

**REPORT as subjective regressions**: changelogs containing judgment
adjectives applied to existing code without a backing technical
specific.

### SR-DH-CHG-7: Field listings belong in kerneldoc

> This looks like kerneldoc, not changelog. Are you sure you want it
> _here_?

**Detection signal**: changelog body contains a bullet list or table of
struct fields, function arguments, or enum values with descriptions.

**REPORT as subjective regressions**: changelogs containing reference
material that should be in code comments.

## Subject + changelog as a unit

When the subject reduces to "Implement $FOO by making miscellaneous
changes", that signals the patch needs to be split (see `SR-DH-DEC-4` in
`tip-code-style.md`). The reviewer should not have to ask "what does this
patch actually do?" after reading subject + first paragraph.

## Cover-letter discipline

For multi-patch series, the cover letter (`PATCH 0/N`) must include:

1. A plain-English problem statement in the first paragraph. Acronyms only
   if previously expanded or universally understood (`KVM`, `TDX` is
   fine; `TDX Connect` is not).
2. What the series does, at a level a maintainer outside the immediate
   subsystem can follow.
3. Patch chapter structure if the series is large enough to need
   chapters (see `SR-DH-DEC-3` in `tip-code-style.md`).

### SR-DH-TDX-6: TDX cover letter must motivate the kernel-side problem

(Cross-referenced from `tdx-review.md`.) TDX series cover letters that
recap TDX module spec terminology without explaining what kernel users
gain bounce immediately.

**Detection signal**: cover letter body consists of ABI quotes / module
feature names with no plain-English problem framing in the first
paragraph.

**REPORT as subjective regressions**: TDX cover letters lacking a plain
problem statement.

## Quick Checks

- **Imperative voice**: no "this patch", "we", "I", "Let's", "Adds",
  "Adding". Sentences are commands.
- **Three-paragraph structure**: context → problem → solution, in that
  order.
- **Function refs use parens**: `foo_bar()` not `foo_bar`.
- **Why, not what**: the first paragraph answers "what kernel-side problem
  does this solve?".
- **Spelling**: run a spell checker; recurring typos are escalated by name.
- **Costs in body**: memory, percent overhead, performance numbers go in
  the changelog, not just the cover letter.
- **Drive-by hunks**: every significant hunk has a corresponding sentence
  in the changelog, or it gets split into its own patch.
- **No kerneldoc in changelogs**: field listings belong above the struct,
  not in the commit message.
- **Marketing terms**: subject and changelog use kernel feature names, not
  vendor product names.
- **Neutral tone**: don't editorialize about existing code with words
  like "misguided", "broken", "no business" unless a concrete technical
  defect immediately follows.
- **Live in the present**: don't reference future series, "upcoming"
  work, or "in preparation for" goals that aren't part of this
  submission.
- **No phrasing thrash**: avoid `etc.`, `e.g.`, `former`/`latter`,
  sentence-initial `Unlike`, sentence-leading `Some`, the `X is:`
  colon-then-bullet construct, forward-references like `see below`,
  and vague assertions (`not suitable`, `is wrong`) without backing
  specifics.
- **Kconfig disable case**: new Kconfig options must motivate when to
  turn them off — and if the disable case is unrealistic, the prompt
  itself is gratuitous; use `def_tristate` / `def_bool` instead.
- **Concrete examples**: use actual product names and version numbers
  (`Sapphire Rapids`, `1.5.7`), not placeholders (`vendor X`, `1.5.x`).
- **Don't bury the headline**: the imperative-mood "what this patch
  does" sentence ends its paragraph or stands alone — not buried
  before a secondary nit.
