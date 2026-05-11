# Networking Drivers: Stats, and Synchronization

## Ethtool Driver Statistics vs Standard Stats

Adding statistics to `ethtool -S` that duplicate counters for which a
standard kernel uAPI already exists creates confusion, leads to huge
ethtool -S lists, and adds maintenance burden. Reviewers routinely
reject such patches.

- **Stats that have a standard uAPI must not be duplicated in `ethtool -S`.**
  The `ethtool -S` interface (`get_ethtool_stats()` / `get_sset_count()` /
  `get_strings()`) is for driver-private statistics only — counters that are
  specific to the hardware or driver and have no standard representation.
- Standard uAPIs exist for common SW-maintained and standards-defined HW
  counters. Categories with standard interfaces include:
  - Network device stats (`struct rtnl_link_stats64` via `ip -s link show`)
  - Per-queue statistics (via netlink)
  - Page pool statistics (via netlink, accessible through `ynl` tooling)
  - Ethtool statistics (for which there is a dedicated callback in
    `struct ethtool_ops`)
  - Other counters exposed through standardized netlink attributes
- A stat does not need to be currently reported by the driver to count as
  a duplicate — if a standard uAPI exists for that category of counter,
  the driver must use the standard interface, not `ethtool -S`.
- When a driver wants to expose a statistic that fits an existing standard
  category, it should implement the appropriate standard interface (e.g.,
  `ndo_get_stats64`) rather than adding a private ethtool string.
- `Documentation/networking/statistics.rst` documents the statistics
  hierarchy and which interfaces to use.

**REPORT as bugs**: Driver patches that add **new** counters to `ethtool -S`
for values that have a standard uAPI — whether or not the driver currently
reports them through that standard interface. Pre-existing `ethtool -S` stats
that predate the standard uAPI are not bugs in new patches (migrating them is
a separate cleanup).

## Ad-hoc Synchronization with Flags and Atomics

Driver code that uses atomic variables, bit flags, or boolean fields as
substitutes for real locks or RCU almost always contains races. These
homebrew schemes provide no actual synchronization guarantees and are
invisible to lockdep, so the bugs they introduce go undetected by
standard kernel debugging tools.

Common broken patterns:

- **Atomic/flag as gate guard**: reading an atomic or flag to decide whether
  to proceed, then operating on shared data without holding a lock. The
  flag's value can change immediately after the read, so the "protection"
  is illusory.
  ```c
  // WRONG: intr_sem can change right after the read
  if (atomic_read(&priv->intr_sem) != 0)
      return;
  // ... operates on shared state with no actual lock held
  ```

- **Bit flags as reader/writer protocol**: using `set_bit()` /
  `test_bit()` / `clear_bit()` to coordinate access between readers and
  a teardown path. Multiple concurrent readers can enter, one clears the
  bit while another is still mid-operation, and the teardown path frees
  memory that the remaining reader is still accessing.
  ```c
  // WRONG: concurrent readers race on the bit
  set_bit(STATE_READ_STATS, &priv->state);
  if (!test_bit(STATE_OPEN, &priv->state)) {
      clear_bit(STATE_READ_STATS, &priv->state);
      return;
  }
  // ... reads from shared data that close path may free
  clear_bit(STATE_READ_STATS, &priv->state);
  ```

- **Retry/poll loops on flags**: spinning on a flag waiting for another
  context to clear it, reimplementing a spinlock without the fairness,
  deadlock detection, or memory ordering guarantees.

- **Trylock loops to avoid deadlock**: using `mutex_trylock()` or
  `spin_trylock()` in a loop or repeated invocation to avoid a lock
  ordering issue is a sign that the locking design is wrong. Trylock is
  only acceptable in narrow cases — for example, a work item that calls
  `mutex_trylock()` and on failure reschedules itself (via
  `schedule_work()` / `schedule_delayed_work()`) so the work runs again
  later. Open-coded retry loops around trylock, or trylock with fallback
  to "skip the work entirely", are almost always bugs.

The correct alternatives depend on the access pattern:

- Reader-heavy paths (e.g., `ndo_get_stats64`): use RCU
- Mutual exclusion with sleep: use a `mutex`
- Mutual exclusion in atomic context: use a `spinlock_t`
- Preventing concurrent execution of a timer or work: use
  `del_timer_sync()` / `cancel_work_sync()`

**REPORT as bugs**: any pattern where a flag, atomic variable, or bit
operation appears to guard a section of code rather than express state —
i.e., where the flag is set on entry and cleared on exit of a code region
to prevent concurrent access, instead of using a proper lock or RCU.

## Quick Checks

- **Ethtool -S stat duplication**: check whether any new `ethtool -S` counters cover values for which a standard uAPI exists (rtnl_link_stats64, page pool stats, per-queue stats via netlink), regardless of whether the driver currently uses that standard interface
- **Flags used as locks**: flag/atomic/bit set-on-entry clear-on-exit patterns that guard code sections are ad-hoc locks; use real locks or RCU instead
