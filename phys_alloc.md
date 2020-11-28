# Physical page allocation

## Nodes and zones

In a [NUMA][numa] (Non-Uniform Memory Access) system memory (and cores) are
divided into __node__s of locality. Memory accesses across nodes may be
significantly slower than accesses within them so it is vastly preferable to
allocate memory within the local node.

In practice this tends to be a product of multi-socket systems with each socket
having its own DIMM slots and a (slow) cross-socket bus inter-node communication.

Different areas of physical memory have different characteristics - for example
many older devices were only capable of accessing 24 bits of physical address
space (i.e. 16MiB), so any allocations for such devices must be in this range.
A more modern 32-bit device might only be capable of performing DMA within the
lower 4GiB of physical memory so any allocations for these devices must be in
this range.

In order to preserve memory with specific properties such as this the physical
memory is sub-divided into __zone__s - typically on an x86-64 system this will
be `ZONE_DMA`, `ZONE_DMA32` and `ZONE_NORMAL`, with the 'normal' zone
representing all physical memory that sits outside of the DMA/DMA32 zones.

Each node is subdivided into these zones.

## Node data structure

Each node is described by a [pg_data_t][pg_data_t] structure (I have pared
it down for clarity, see the linked source for the full declaration):

```
typedef struct pglist_data {
    /*
     * node_zones contains just the zones for THIS node. Not all of the
     * zones may be populated, but it is the full list. It is referenced by
     * this node's node_zonelists as well as other node's node_zonelists.
     */
    struct zone node_zones[MAX_NR_ZONES];

    /*
     * node_zonelists contains references to all zones in all nodes.
     * Generally the first zones will be references to this node's
     * node_zones.
     */
    struct zonelist node_zonelists[MAX_ZONELISTS];

    int nr_zones; /* number of populated zones in this node */

    unsigned long node_start_pfn;
    unsigned long node_present_pages; /* total number of physical pages */
    unsigned long node_spanned_pages; /* total size of physical page
                         range, including holes */
    int node_id;

    int kswapd_order;
    enum zone_type kswapd_highest_zoneidx;

    int kswapd_failures;        /* Number of 'reclaimed == 0' runs */

    int kcompactd_max_order;
    enum zone_type kcompactd_highest_zoneidx;
    wait_queue_head_t kcompactd_wait;
    struct task_struct *kcompactd;

    /*
     * This is a per-node reserve of pages that are not available
     * to userspace allocations.
     */
    unsigned long       totalreserve_pages;

    /*
     * node reclaim becomes active if more unmapped pages exist.
     */
    unsigned long       min_unmapped_pages;
    unsigned long       min_slab_pages;
} pg_data_t;
```

Essentially this contains zone data structures and statistics.

## Zone data structure

The [zone][zone] data structure describes memory within a zone (again stripped
down for clarity, see the link for the full declaration):

```
struct zone {
    /* zone watermarks, access with *_wmark_pages(zone) macros */
    unsigned long _watermark[NR_WMARK];
    unsigned long watermark_boost;

    unsigned long nr_reserved_highatomic;

    /*
     * We don't know if the memory that we're going to allocate will be
     * freeable or/and it will be released eventually, so to avoid totally
     * wasting several GB of ram we must reserve some of the lower zone
     * memory (otherwise we risk to run OOM on the lower zones despite
     * there being tons of freeable ram on the higher zones).  This array is
     * recalculated at runtime if the sysctl_lowmem_reserve_ratio sysctl
     * changes.
     */
    long lowmem_reserve[MAX_NR_ZONES];

    int node;

    struct pglist_data  *zone_pgdat;
    struct per_cpu_pageset __percpu *pageset;

    /*
     * Flags for a pageblock_nr_pages block. See pageblock-flags.h.
     * In SPARSEMEM, this map is stored in struct mem_section
     */
    unsigned long       *pageblock_flags;

    /* zone_start_pfn == zone_start_paddr >> PAGE_SHIFT */
    unsigned long       zone_start_pfn;

    /*
     * spanned_pages is the total pages spanned by the zone, including
     * holes, which is calculated as:
     *  spanned_pages = zone_end_pfn - zone_start_pfn;
     *
     * present_pages is physical pages existing within the zone, which
     * is calculated as:
     *  present_pages = spanned_pages - absent_pages(pages in holes);
     *
     * managed_pages is present pages managed by the buddy system, which
     * is calculated as (reserved_pages includes pages allocated by the
     * bootmem allocator):
     *  managed_pages = present_pages - reserved_pages;
     *
     * So present_pages may be used by memory hotplug or memory power
     * management logic to figure out unmanaged pages by checking
     * (present_pages - managed_pages). And managed_pages should be used
     * by page allocator and vm scanner to calculate all kinds of watermarks
     * and thresholds.
     *
     * Locking rules:
     *
     * zone_start_pfn and spanned_pages are protected by span_seqlock.
     * It is a seqlock because it has to be read outside of zone->lock,
     * and it is done in the main allocator path.  But, it is written
     * quite infrequently.
     *
     * The span_seq lock is declared along with zone->lock because it is
     * frequently read in proximity to zone->lock.  It's good to
     * give them a chance of being in the same cacheline.
     *
     * Write access to present_pages at runtime should be protected by
     * mem_hotplug_begin/end(). Any reader who can't tolerant drift of
     * present_pages should get_online_mems() to get a stable value.
     */
    atomic_long_t       managed_pages;
    unsigned long       spanned_pages;
    unsigned long       present_pages;

    const char      *name;

    /*
     * Number of isolated pageblock. It is used to solve incorrect
     * freepage counting problem due to racy retrieving migratetype
     * of pageblock. Protected by zone->lock.
     */
    unsigned long       nr_isolate_pageblock;

    int initialized;

    /* free areas of different sizes */
    struct free_area    free_area[MAX_ORDER];

    /* zone flags, see below */
    unsigned long       flags;

    /*
     * When free pages are below this point, additional steps are taken
     * when reading the number of free pages to avoid per-cpu counter
     * drift allowing watermarks to be breached
     */
    unsigned long percpu_drift_mark;

    /* pfn where compaction free scanner should start */
    unsigned long       compact_cached_free_pfn;
    /* pfn where compaction migration scanner should start */
    unsigned long       compact_cached_migrate_pfn[ASYNC_AND_SYNC];
    unsigned long       compact_init_migrate_pfn;
    unsigned long       compact_init_free_pfn;

    /*
     * On compaction failure, 1<<compact_defer_shift compactions
     * are skipped before trying again. The number attempted since
     * last failure is tracked with compact_considered.
     * compact_order_failed is the minimum compaction failed order.
     */
    unsigned int        compact_considered;
    unsigned int        compact_defer_shift;
    int         compact_order_failed;

    /* Set to true when the PG_migrate_skip bits should be cleared */
    bool            compact_blockskip_flush;

    bool            contiguous;
};
```

## Zone watermarks

Each zone has 'watermarks' which determine reclaim behaviour (the kernel
mechanisms for freeing up physical memory), all expressed in pages:

* Minimum - If free pages is less than or equal to the minimum watermark +
  lowmem reserve (see below) then allocation in this zone is not permitted (and
  reclaim might be started).

* Low - When free pages reach this level then the `kswapd` process wakes up and
  performs indirect reclaim of pages.

* High - At free pages above this point `kswapd` can sleep for this node/zone.

## Zone page stats

Each zone has 'spanned', 'present', and 'managed' page counts (viewable from
`/proc/zoneinfo` which also has watermark and much other information):

* __spanned__ is the number of pages contained within the physical address range
  of the zone.

* __present__ is the number of physical pages that actually exist in
  the range.

* __managed__ is the number of present pages that are currently under the
  control of the physical memory allocator (i.e. not otherwise reserved).

## Zone assignment

The [__alloc_pages_nodemask()][__alloc_pages_nodemask] allocation algorithm
(discussed in more detail below) which performs all physical memory allocations
attempts to use the 'highest' zone specified by the Get Free Pages (GFP)
flags. A standard allocation will set all zones as acceptable.

By 'highest' this is in reference to the value of the zone enums, which range in
ascending order from the smaller lower memory regions (e.g. `ZONE_DMA`,
`ZONE_DMA32`) to ones that span the rest of the physical memory.

However if the allocation fails at the higher zone (e.g. `ZONE_NORMAL`) then the
algorithm will try to allocate from the lower zones (E.g. `ZONE_DMA32`).

In order to avoid exhausting lower memory, the
[lowmem_reserve_ratio][lowmem_reserve_ratio] mechanism is implemented to specify
the proportion of pages that are reserved per-memory zone as specified via the
`vm.lowmem_reserve_ratio` tunable.

To do this, a number of pages is added to the minimum pages watermark for each
zone determined by the original maximum zone that the allocation could have
used.

For example a tunable of `[256, 256, 32]` for `ZONE_DMA`, `ZONE_DMA32`,
`ZONE_NORMAL` respectively implies that the minimum watermark must be expanded
by 1/256, 1/256, and 1/32 of all managed pages in zones above the one being
checked for allocations that have a maximum allowed zone of `ZONE_DMA`,
`ZONE_DMA32`, and `ZONE_NORMAL` respectively (this is somewhat confusing!)

You can observe the actual page counts listed under `protection`
in `/proc/zoneinfo`, e.g. on my system the DMA32 zone indicates:

```
Node 0, zone      DMA
        ...
        managed  3975
        protection: (0, 907, 64235)
...
Node 0, zone    DMA32
        ...
        managed  232262
        protection: (0, 0, 63327)
...
Node 0, zone   Normal
        managed  16211930
        protection: (0, 0, 0)
```

These are taken directly from the `lowmem_reserve` field in [struct zone][zone]
and recalculated if the `vm.lowmem_reserve_ratio` tunable is altered (and on
startup) via [setup_per_zone_lowmem_reserve()][setup_per_zone_lowmem_reserve].

The algorithm is designed such that all zones at or below the one being
allocated from incur no reserved pages. For those greater, we sum all managed
pages for the highest zone we could use and divide by the ratio of the zone we
are intended to allocate from.

So for the DMA zone, if we could have allocated from `ZONE_NORMAL` there are
232,262 + 16,211,930 = 16,444,192 managed pages we could have allocated from at
higher zones. Dividing by the 256 ratio gives us 64,235 pages which is greater
than the sum of all managed pages thus such allocations will always fail.

For DMA allocating from DMA32 we are left with 232,262 / 256 = 907 pages.

This way we are able to use lower zones if needed but reserving pages at each
proportional to the number of pages managed by zones above.

## Buddy allocator

Linux uses the [buddy allocator][buddy] algorithm invented in 1963 in order to
allocate memory pages (at the smallest available page size), e.g. on x86-64
4KiB.

The buddy allocator provides a number of pages equal to the 'order' of the
allocation which is simply a power of 2. E.g. an order 3 allocation will provide
2^3 = 8 pages, an order 0 will provide 2^0 = 1 page, etc.

The algorithm keeps a track of pages at each supported order level, starting
with the largest supported order ([MAX_ORDER][MAX_ORDER], typically 11 = 2,048
pages or 8 MiB) to the lowest.

TBD

[numa]:https://en.wikipedia.org/wiki/Non-uniform_memory_access
[buddy]:https://en.wikipedia.org/wiki/Buddy_memory_allocation
[pg_data_t]:https://github.com/torvalds/linux/blob/45e885c439e825c19f3a51e46ef8210984bc0a9c/include/linux/mmzone.h#L726
[zone]:https://github.com/torvalds/linux/blob/45e885c439e825c19f3a51e46ef8210984bc0a9c/include/linux/mmzone.h#L448
[MAX_ORDER]:https://github.com/torvalds/linux/blob/45e885c439e825c19f3a51e46ef8210984bc0a9c/include/linux/mmzone.h#L27
[__alloc_pages_nodemask]:https://github.com/torvalds/linux/blob/45e885c439e825c19f3a51e46ef8210984bc0a9c/mm/page_alloc.c#L4917
[lowmem_reserve_ratio]:https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/sysctl/vm.rst#lowmem_reserve_ratio
[setup_per_zone_lowmem_reserve]:https://github.com/torvalds/linux/blob/45e885c439e825c19f3a51e46ef8210984bc0a9c/mm/page_alloc.c#L7791
