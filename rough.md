## Rough phys_alloc

`init/main.c:start_kernel()`
`init/main.c:mm_init()`
`arch/x86/mm/init_64.c:mem_init()`

```
pg_data_t
```

VMEMMAP_START -> struct page array

```
dma       15992kB -> 16 MiB
dma32    994792kB -> 1 GiB
normal 66027520kB -> 62.9 GiB

e):0kB mapped:599520kB dirty:4316kB writeback:0kB shmem:375424kB shmem_thp: 0kB shmem_pmdmapped: 0kB anon_thp: 0kB writeback_tmp:0kB kernel_stack:11648kB all_unreclaimable? no
[  +0.000001] Node 0 DMA free:11804kB min:16kB low:28kB high:40kB reserved_highatomic:0KB active_anon:0kB inactive_anon:0kB active_file:0kB inactive_file:0kB unevictable:0kB writepending:0kB present:15992kB managed:15900kB mlocked:0kB pagetables:0kB bounce:0kB free_pcp:0kB local_pcp:0kB free_cma:0kB
[  +0.000002] lowmem_reserve[]: 0 894 64216 64216 64216
[  +0.000003] Node 0 DMA32 free:921852kB min:940kB low:1852kB high:2764kB reserved_highatomic:0KB active_anon:0kB inactive_anon:0kB active_file:0kB inactive_file:0kB unevictable:0kB writepending:0kB present:994792kB managed:929048kB mlocked:0kB pagetables:0kB bounce:0kB free_pcp:3096kB local_pcp:0kB free_cma:0kB
[  +0.000002] lowmem_reserve[]: 0 0 63322 63322 63322
[  +0.000002] Node 0 Normal free:53072736kB min:66624kB low:131464kB high:196304kB reserved_highatomic:0KB active_anon:10552kB inactive_anon:3201960kB active_file:3276192kB inactive_file:4324376kB unevictable:32kB writepending:4316kB present:66027520kB managed:64847720kB mlocked:32kB pagetables:31504kB bounce:0kB free_pcp:17648kB local_pcp:488kB free_cma:0kB
[  +0.000003] lowmem_reserve[]: 0 0 0 0 0
[  +0.000002] Node 0 DMA: 1*4kB (U) 1*8kB (U) 1*16kB (U) 2*32kB (U) 3*64kB (U) 2*128kB (U) 0*256kB 0*512kB 1*1024kB (U) 1*2048kB (M) 2*4096kB (M) = 11804kB
[  +0.000006] Node 0 DMA32: 9*4kB (UM) 9*8kB (M) 9*16kB (M) 10*32kB (M) 9*64kB (M) 7*128kB (M) 7*256kB (UM) 7*512kB (UM) 7*1024kB (M) 7*2048kB (M) 218*4096kB (M) = 921852kB
[  +0.000007] Node 0 Normal: 2346*4kB (UME) 33804*8kB (UME) 20404*16kB (UME) 10252*32kB (UME) 4325*64kB (UME) 1741*128kB (UME) 533*256kB (ME) 104*512kB (UME) 13*1024kB (UM) 5*2048kB (M) 12555*4096kB (M) = 53072520kB


DMA      15908 kB /   15992 kB 16 MiB
DMA32  1279984 kB / 2080624 kB 1.98 GiB
Normal   41572 kB / 2097152 kB 2 GiB




[9826845.902096] Node 0 active_anon:256028kB inactive_anon:33000kB active_file:1396520kB inactive_file:740308kB unevictable:0kB isolated(anon):0kB isolated(file):0kB mapped:159896kB dirty:672kB writeback:0kB shmem:41060kB shmem_thp: 0kB shmem_pmdmapped: 0kB anon_thp: 0kB writeback_tmp:0kB unstable:0kB all_unreclaimable? no
[9826845.902096] Node 0 DMA free:15908kB min:268kB low:332kB high:396kB reserved_highatomic:0KB active_anon:0kB inactive_anon:0kB active_file:0kB inactive_file:0kB unevictable:0kB writepending:0kB present:15992kB managed:15908kB mlocked:0kB kernel_stack:0kB pagetables:0kB bounce:0kB free_pcp:0kB local_pcp:0kB free_cma:0kB
[9826845.902137] lowmem_reserve[]: 0 1967 3896 3896 3896
[9826845.902140] Node 0 DMA32 free:1279984kB min:33992kB low:42488kB high:50984kB reserved_highatomic:0KB active_anon:8372kB inactive_anon:12kB active_file:420448kB inactive_file:273024kB unevictable:0kB writepending:544kB present:2080624kB managed:2015088kB mlocked:0kB kernel_stack:240kB pagetables:1416kB bounce:0kB free_pcp:1668kB local_pcp:452kB free_cma:0kB
[9826845.902145] lowmem_reserve[]: 0 0 1929 1929 1929
[9826845.902147] Node 0 Normal free:41572kB min:33320kB low:41648kB high:49976kB reserved_highatomic:0KB active_anon:247656kB inactive_anon:32988kB active_file:976072kB inactive_file:467284kB unevictable:0kB writepending:128kB present:2097152kB managed:1982172kB mlocked:0kB kernel_stack:3216kB pagetables:10808kB bounce:0kB free_pcp:2700kB local_pcp:1424kB free_cma:0kB
[9826845.902151] lowmem_reserve[]: 0 0 0 0 0
[9826845.902154] Node 0 DMA: 1*4kB (U) 0*8kB 0*16kB 1*32kB (U) 2*64kB (U) 1*128kB (U) 1*256kB (U) 0*512kB 1*1024kB (U) 1*2048kB (M) 3*4096kB (M) = 15908kB
[9826845.902162] Node 0 DMA32: 502*4kB (UME) 341*8kB (UME) 178*16kB (UME) 153*32kB (UME) 113*64kB (UME) 90*128kB (UME) 60*256kB (UM) 15*512kB (UME) 19*1024kB (UM) 31*2048kB (M) 279*4096kB (UM) = 1280000kB
[9826845.902171] Node 0 Normal: 141*4kB (UME) 148*8kB (UE) 91*16kB (UE) 47*32kB (UME) 24*64kB (UME) 4*128kB (U) 4*256kB (UME) 2*512kB (UE) 2*1024kB (UE) 3*2048kB (M) 6*4096kB (UM) = 41572kB
```

## Handy

```
/proc/zoneinfo # Per-zone memory info -> spanned, present, managed
/proc/meminfo
/proc/vmstat
```

Tunables... https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/sysctl/vm.rst

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
