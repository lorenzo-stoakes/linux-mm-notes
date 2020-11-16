# Raw notes

Ad-hoc raw notes.

## Notes from Chris Down presentation

https://media.ccc.de/v/arch-conf-online-2020-6390-linux-memory-management-at-scale
https://chrisdown.name/2018/01/02/in-defence-of-swap.html


* Different types of memory - anonymous (unbacked), cache/buffers (part of
  unified page cache), etc.

* 'Reclaimability' is a factor of use.

* RSS - skews towards anonymous, mapped file. A program might rely on file
  buffers/caches to work performantly.

* Swap - increases reclaim equality and reliability of forward progress of the
  system, NOT emergency memory. Can reduce page cache availability.

* Swap - when not present causes a fall off the cliff very quickly when memory
  pressure is high.

* Swap - People worry that it can delay OOM. But it is a blunt tool that occurs
  when reclaim fails.

* `pte_young()` gives an indication of page usage. We don't know we're OOM ahead
  of time.

* oom killer will kill the highest RSS, not always the right choice.

* Reclaim is done by `kswapd` in the background when resident pages goes above a
  threshold.

* Direct reclaim is when an application has no memory available and is blocked
  until it is found.

* Tries to reclaim the coldest pages first, but some pages might not be
  reclaimable - e.g. very hot cache pages, dirty pages, pages otherwise
  unavailable.

* Hard to determine memory pressure even with scan rates - might be very
  efficient.

* Under cgroupv2 can do:

```
cat /sys/fs/cgroup/system.slice/memory.pressure
```

* Kernel measurement 'psi' - measures e.g. undesirable memory operations.

* `oomd` - `bit.ly/fboomd` - Early-warning highly configurable OOM detector.

* Limits don't compose very well - easy to get wrong. Preferable to use a
  'protection' threshold, e.g. `memory.{low,min}` - won't do reclaim below min,
  and low only if system-wide memory contention.

* Writeback threshold.

* `bit.ly/fbtax2` - FB tool for managing resources.

* `vm.overcommit_memory` - Operates at VM level so not so useful.

* Can get PSI from `/proc/pressure/memory`.

* `CONFIG_PSI` - Get pressure measures.
