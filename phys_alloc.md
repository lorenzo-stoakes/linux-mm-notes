# Physical page allocation

## Nodes and zones

In a [NUMA][numa] (Non-Uniform Memory Access) system memory (and cores) are
divided into __nodes__ of locality. Memory accesses across nodes may be
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

## Zone layout

In order to preserve memory with specific properties such as this the physical
memory is sub-divided into __zones__ - typically on an x86-64 system this will
be `ZONE_DMA`, `ZONE_DMA32` and `ZONE_NORMAL`, with the 'normal' zone
representing all physical memory that sits outside of the DMA/DMA32 zones.

There are additionally 2 'special' zones:

* `ZONE_MOVABLE` - optionally available if the [kernelcore][kernelcore] and/or
  [movablecore][movablecore] command line parameters are specified. The memory
  in this zone is explicitly movable.
* `ZONE_DEVICE` - contains memory identified as device-specific. This memory is
  never free or available for any other use and must be explicitly set up by the
  driver. See [ZONE_DEVICE documentation][zone_device-doc] for more details.

Each node is subdivided into these zones, however lower memory zones will only
be populated for node 0 as physical memory is addressed across all nodes.


```
 ^ node 0                               0 -> |---------------| ^
 |                                           |    ZONE_DMA   | | 16 MiB
 |                                 16 MiB -> |---------------| x
 |                                           |   ZONE_DMA32  | | 4 GiB
 | ^ node 1+                        4 GiB -> |---------------| x
 | |                                         |  ZONE_NORMAL  | | total RAM - 4G16M - movable
 | |                  total RAM - movable -> |---------------| x
 | |                                         |  ZONE_MOVABLE | | movable
 | |                            total RAM -> |---------------| x
 | |                                         |  ZONE_DEVICE  | | device RAM
 v v               total RAM + device RAM -> |---------------| v
```

If there is insufficient memory to fit in the range of a memory zone then that
zone remains unpopulated, e.g. if a system has less than 4 GiB of RAM available
then `ZONE_NORMAL` is empty.

## Node data structure

Each node is described by a [pg_data_t][pg_data_t] structure (I have pared
it down for clarity, see the linked source for the full declaration):

```c
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

```c
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

* __Minimum__ - If free pages are less than or equal to the minimum watermark +
  lowmem reserve (see below) then allocation in this zone is not permitted and
  direct memory reclaim can occur to attempt to free sufficient pages to reach
  the watermark again.

* __Low__ - When free pages reach this level then the `kswapd` process wakes up and
  performs indirect reclaim of pages.

* __High__ - If free pages are equal to or greater than this value `kswapd` can
  sleep for this node/zone.

The watermarks are updated via [setup_per_zone_wmarks()][setup_per_zone_wmarks]
and ultimately [__setup_per_zone_wmarks()][__setup_per_zone_wmarks]. The minimum
watermark is initially set by
[init_per_zone_wmark_min()][init_per_zone_wmark_min] which sets it to roughly
`4 * sqrt(mem_kbytes)` KiB and is controllable via the `vm.min_free_kbytes`
tunable which, when changed, invokes `setup_per_zone_wmarks()` again.

The minimum watermark for each zone is set equal to `vm.min_free_kbytes` (in
pages) multiplied by the fraction of managed pages within in each zone.

The low and high watermarks are determined by
[vm.watermark_scale_factor][watermark_scale_factor] expressed in fractions of
10,000, e.g. the default 10 = 10/10,000 = 0.1%.

The low watermark is equal to the minimum + scale factor * zone managed pages,
and the high watermark is equal to the minimum + 2 * scale factor * zone managed
pages.

## Zone page stats

Each zone has 'spanned', 'present', and 'managed' page counts (viewable from
`/proc/zoneinfo`):

* __Spanned__ - The number of pages contained within the physical address range
  of the zone.

* __Present__ - The number of physical pages that actually exist in
  the range.

* __Managed__ - The number of present pages that are currently under the
  control of the physical memory allocator (i.e. not otherwise reserved).

## Zone assignment

When performing allocations we attempt to use the 'highest' zone specified by
the Get Free Pages (GFP) flags (which specify how an allocation should be
performed, usually all zones are game).

By 'highest' this is in reference to the value of the zone enums, which range in
ascending order from the smaller lower memory regions (e.g. `ZONE_DMA`,
`ZONE_DMA32`) to ones that span the rest of the physical memory.

However if the allocation fails at the higher zone (e.g. `ZONE_NORMAL`) then the
algorithm will try to allocate from the lower zones (e.g. `ZONE_DMA32`,
`ZONE_DMA`) utilising the low memory reserve mechanism to maintain a minimum
number of free pages in each zone for zone-specific allocations.

## Low memory reserve

In order to avoid exhausting lower memory the
[vm.lowmem_reserve_ratio][lowmem_reserve_ratio] tunable specifies per-zone
ratios of memory to reserve in each zone.

The tunable is quite confusing - it is specified as a series of integers which
specify the ratio of managed pages above the zone which are to be kept in
reserve when an allocation is performed which uses a lower zone.

For example:

```
[~]$ cat /proc/sys/vm/lowmem_reserve_ratio
256    128    32    0
```

This indicates that we should reserve pages at the following ratios of the sum
of all pages managed by zones higher than the zone we're reserving pages for:

| Alloc at / Could alloc at | ZONE_DMA | ZONE_DMA32 | ZONE_NORMAL | ZONE_MOVABLE |
|---------------------------|----------|------------|-------------|--------------|
| ZONE_DMA                  | .        | 1/256      | 1/256       | 1/256        |
| ZONE_DMA32                | .        | .          | 1/128       | 1/128        |
| ZONE_NORMAL               | .        | .          | .           | 1/32         |
| ZONE_MOVABLE              | .        | .          | .           | .            |

The zones listed on the left are those that the allocation is occurring in, the
zones listed at the top are the maximum zone that an allocation is permitted to
be made in, e.g. if no GFP flags constrain which zone an allocation can be
sourced from then `ZONE_MOVABLE` is the maximum zone we can allocate from and so
we reference the last column. If the GFP flags indicated that the allocation
must be in `ZONE_DMA32`or lower, then we reference the 2nd column, etc.

No reservation need be made for allocations that are strictly only permitted in
the zone being allocated in (e.g. `ZONE_DMA32` in `ZONE_DMA32`), so the reserve
is not applied there. Also of course you cannot make an allocation that
specifies a lower zone in a higher one (e.g. `ZONE_DMA` allocation in
`ZONE_DMA32`) so in both cases these are marked with '.'s.

The ratio is applied to the _sum_ of all _managed_ pages in zones _above_ the
one being allocated in. This is because the ratio is designed to apply an
increasing penalty for allocations which could have been allocated in a higher
zone.

Looking at a portion of the output of `/proc/zoneinfo` from my test system:

```
Node 0, zone      DMA
        managed  3977
        protection: (0, 2991, 10163, 30084)
Node 0, zone    DMA32
        managed  765917
        protection: (0, 0, 14344, 54185)
Node 0, zone   Normal
        managed  1836032
        protection: (0, 0, 0, 159364)
Node 0, zone  Movable
        managed  5099663
        protection: (0, 0, 0, 0)
```

These 'protection' values indicate the actual number of pages reserved for
allocations that could have been allocated at higher zones, e.g. `ZONE_DMA`
reserves 2,991 pages for `ZONE_DMA32` allocations, 10,163 for `ZONE_NORMAL`
allocations and 30,084 for `ZONE_MOVABLE` allocations (the latter values exceed
managed pages in `ZONE_DMA` so in effect prevent allocations from these zones
occurring in `ZONE_DMA`).

'Reserving' pages is implemented by checking the sum of the minimum watermark
and the relevant `lowmem_reserve` value.

In order to see how these protection values have derived from the ratios
specified in the tunable, let's work out how to sum the managed pages _above_
each zone:

| Alloc at / Could alloc at | ZONE_DMA | ZONE_DMA32 | ZONE_NORMAL    | ZONE_MOVABLE             |
|---------------------------|----------|------------|----------------|--------------------------|
| ZONE_DMA                  | .        | DMA32      | DMA32 + NORMAL | DMA32 + NORMAL + MOVABLE |
| ZONE_DMA32                | .        | .          | NORMAL         | NORMAL + MOVABLE         |
| ZONE_NORMAL               | .        | .          | .              | MOVABLE                  |
| ZONE_MOVABLE              | .        | .          | .              | .                        |

Taking the actual page counts:

| Alloc at / Could alloc at | ZONE_DMA | ZONE_DMA32 | ZONE_NORMAL | ZONE_MOVABLE |
|---------------------------|----------|------------|-------------|--------------|
| ZONE_DMA                  | .        | 765,917    | 2,601,949   | 7,701,612    |
| ZONE_DMA32                | .        | .          | 1,836,032   | 6,935,695    |
| ZONE_NORMAL               | .        | .          | .           | 5,099,663    |
| ZONE_MOVABLE              | .        | .          | .           | .            |

And finally multiplying by the ratios from the first table:

| Alloc at / Could alloc at | ZONE_DMA | ZONE_DMA32 | ZONE_NORMAL | ZONE_MOVABLE |
|---------------------------|----------|------------|-------------|--------------|
| ZONE_DMA                  | .        | 2,991      | 10,163      | 30,084       |
| ZONE_DMA32                | .        | .          | 14,344      | 54,185       |
| ZONE_NORMAL               | .        | .          | .           | 159,364      |
| ZONE_MOVABLE              | .        | .          | .           | .            |

Which matches the values reported by `/proc/zoneinfo`.

These are used in the code directly from the `lowmem_reserve` field in [struct
zone][zone] and recalculated if the `vm.lowmem_reserve_ratio` tunable is altered
(and on startup) via
[setup_per_zone_lowmem_reserve()][setup_per_zone_lowmem_reserve].

## struct page

Physical pages are described by 64-byte [struct page][page] objects each of
which represent a 4 KiB page (larger page sizes are represented by a compound
set of page elements).

The struct itself is consists of a set of flags and a series of
context-dependent unions:

```c
struct page {
    unsigned long flags;        /* Atomic flags, some possibly
                                 * updated asynchronously */
    /*
     * Five words (20/40 bytes) are available in this union.
     * WARNING: bit 0 of the first word is used for PageTail(). That
     * means the other users of this union MUST NOT use the bit to
     * avoid collision and false-positive PageTail().
     */
    union {
        struct {    /* Page cache and anonymous pages */
            /**
             * @lru: Pageout list, eg. active_list protected by
             * lruvec->lru_lock.  Sometimes used as a generic list
             * by the page owner.
             */
            struct list_head lru;
            /* See page-flags.h for PAGE_MAPPING_FLAGS */
            struct address_space *mapping;
            pgoff_t index;      /* Our offset within mapping. */
            /**
             * @private: Mapping-private opaque data.
             * Usually used for buffer_heads if PagePrivate.
             * Used for swp_entry_t if PageSwapCache.
             * Indicates order in the buddy system if PageBuddy.
             */
            unsigned long private;
        };
        struct {    /* page_pool used by netstack */
            /**
             * @dma_addr: might require a 64-bit value even on
             * 32-bit architectures.
             */
            dma_addr_t dma_addr;
        };
        struct {    /* slab, slob and slub */
            union {
                struct list_head slab_list;
                struct {    /* Partial pages */
                    struct page *next;
                    int pages;  /* Nr of pages left */
                    int pobjects;   /* Approximate count */
                };
            };
            struct kmem_cache *slab_cache; /* not slob */
            /* Double-word boundary */
            void *freelist;     /* first free object */
            union {
                void *s_mem;    /* slab: first object */
                unsigned long counters;     /* SLUB */
                struct {            /* SLUB */
                    unsigned inuse:16;
                    unsigned objects:15;
                    unsigned frozen:1;
                };
            };
        };
        struct {    /* Tail pages of compound page */
            unsigned long compound_head;    /* Bit zero is set */

            /* First tail page only */
            unsigned char compound_dtor;
            unsigned char compound_order;
            atomic_t compound_mapcount;
            unsigned int compound_nr; /* 1 << compound_order */
        };
        struct {    /* Second tail page of compound page */
            unsigned long _compound_pad_1;  /* compound_head */
            atomic_t hpage_pinned_refcount;
            /* For both global and memcg */
            struct list_head deferred_list;
        };
        struct {    /* Page table pages */
            unsigned long _pt_pad_1;    /* compound_head */
            pgtable_t pmd_huge_pte; /* protected by page->ptl */
            unsigned long _pt_pad_2;    /* mapping */
            union {
                struct mm_struct *pt_mm; /* x86 pgds only */
                atomic_t pt_frag_refcount; /* powerpc */
            };
            spinlock_t ptl;
        };
        struct {    /* ZONE_DEVICE pages */
            /** @pgmap: Points to the hosting device page map. */
            struct dev_pagemap *pgmap;
            void *zone_device_data;
            /*
             * ZONE_DEVICE private pages are counted as being
             * mapped so the next 3 words hold the mapping, index,
             * and private fields from the source anonymous or
             * page cache page while the page is migrated to device
             * private memory.
             * ZONE_DEVICE MEMORY_DEVICE_FS_DAX pages also
             * use the mapping, index, and private fields when
             * pmem backed DAX files are mapped.
             */
        };

        /** @rcu_head: You can use this to free a page by RCU. */
        struct rcu_head rcu_head;
    };

    union {     /* This union is 4 bytes in size. */
        /*
         * If the page can be mapped to userspace, encodes the number
         * of times this page is referenced by a page table.
         */
        atomic_t _mapcount;

        /*
         * If the page is neither PageSlab nor mappable to userspace,
         * the value stored here may help determine what this page
         * is used for.  See page-flags.h for a list of page types
         * which are currently stored here.
         */
        unsigned int page_type;

        unsigned int active;        /* SLAB */
        int units;          /* SLOB */
    };

    /* Usage count. *DO NOT USE DIRECTLY*. See page_ref.h */
    atomic_t _refcount;

#ifdef CONFIG_MEMCG
    unsigned long memcg_data;
#endif
} _struct_page_alignment;
```

### Page flags

Pages are flagged by both the `flags` and `page_type` fields, neither of which
should be accessed directly but rather via `PageXXX()` macros. All of the flags
and macro mechanisms are defined in [include/linux/page-flags.h][page-flags].

The available macros are `PageXXX()` which tests if the flag is set,
`SetPageXXX()` which sets the flag, `ClearPageXXX()` which clears it, and for
some flags `TestSetPageXXX()` which sets the flag returning its original value,
`TestClearPageXXX()` which clears the flag returning its original value.

Page type flags can only be used if the page is neither `PageSlab` nor mappable
to userspace.

Each page flag that references the `flags` field must apply a policy. As described
in `page-flags.h`:

```c
/*
 * Page flags policies wrt compound pages
 *
 * PF_POISONED_CHECK
 *     check if this struct page poisoned/uninitialized
 *
 * PF_ANY:
 *     the page flag is relevant for small, head and tail pages.
 *
 * PF_HEAD:
 *     for compound page all operations related to the page flag applied to
 *     head page.
 *
 * PF_ONLY_HEAD:
 *     for compound page, callers only ever operate on the head page.
 *
 * PF_NO_TAIL:
 *     modifications of the page flag must be done on small or head pages,
 *     checks can be done on tail pages too.
 *
 * PF_NO_COMPOUND:
 *     the page flag is not relevant for compound pages.
 *
 * PF_SECOND:
 *     the page flag is stored in the first tail page.
 */
```

| Flag             | field                        | Policy      | Notes                                               |
|------------------|------------------------------|-------------|-----------------------------------------------------|
| Head             | flags                        | any         |                                                     |
| Tail             | compound_head                | -           | Own function                                        |
| Compound         | flags, compound_head         | -           | Own function                                        |
| Poisoned         | flags                        | -           | Own function                                        |
| Locked           | flags                        | No tail     |                                                     |
| Waiters          | flags                        | Only head   |                                                     |
| Error            | flags                        | No tail     |                                                     |
| Referenced       | flags                        | head        |                                                     |
| Dirty            | flags                        | head        |                                                     |
| LRU              | flags                        | head        |                                                     |
| Active           | flags                        | head        |                                                     |
| Workingset       | flags                        | head        |                                                     |
| Slab             | flags                        | no tail     |                                                     |
| SlobFree         | flags                        | no tail     |                                                     |
| Checked          | flags                        | no compound |                                                     |
| Pinned           | flags                        | no compound |                                                     |
| SavePinned       | flags                        | no compound |                                                     |
| Foreign          | flags                        | no compound |                                                     |
| XenRemapped      | flags                        | no compound |                                                     |
| Reserved         | flags                        | no compound |                                                     |
| SwapBacked       | flags                        | no tail     |                                                     |
| Private          | flags                        | any         |                                                     |
| Private2         | flags                        | any         |                                                     |
| OwnerPriv1       | flags                        | any         |                                                     |
| Writeback        | flags                        | no tail     |                                                     |
| MappedToDisk     | flags                        | no tail     |                                                     |
| Reclaim          | flags                        | no tail     |                                                     |
| Readahead        | flags                        | no compound |                                                     |
| SwapCache        | flags                        | no tail     | Own function                                        |
| Unevictable      | flags                        | head        |                                                     |
| Mlocked          | flags                        | no tail     |                                                     |
| Uncached         | flags                        | no compound |                                                     |
| HWPoison         | flags                        | any         |                                                     |
| Reported         | flags                        | no compound |                                                     |
| MappingFlags     | mapping                      | -           | Own function                                        |
| Anon             | mapping                      | -           | Own function                                        |
| __Movable        | mapping                      | -           | Own function, __Page prefix                         |
| Uptodate         | page                         | -           | Own function, memory barriers required, FS-specific |
| Huge             | compound_head, compound_dtor | -           | Own function, only checks for hugetlbfs             |
| HeadHuge         | flags, compound_dtor         | any         | Own function, only checks for hugetlbfs             |
| TransHuge        | flags                        | any         | Own function, must know not hugetlbfs               |
| TransCompound    | flags                        | any         | Own function, must know not hugetlbfs               |
| TransCompoundMap | flag, _mapcount              | any         | Own function, must know not hugetlbfs               |
| TransTail        | compound_head                | -           | Own function, must know not hugetlbfs               |
| DoubleMap        | flags                        | second      |                                                     |
| Buddy            | page_type                    | -           | Page is free and in the buddy system                |
| Offline          | page_type                    | -           | Page offline though section online                  |
| Table            | page_type                    | -           | Indicates used for a page table                     |
| Guard            | page_type                    | -           | Indicates used with debug_pagealloc                 |
| Isolated         | flags                        | any         |                                                     |
| SlabPfmemalloc   | flags                        | head        | Own function, checks PageSlab()                     |

### Initial struct page allocation

x86-64 uses the [sparse memory model][sparsemem] which divides contiguous sets
of `struct page`s into sections and, as x86-64 specifies
`CONFIG_SPARSEMEM_VMEMMAP`, provides a virtual mapping so `struct page`s can be
accessed as a simple offset.

The process starts at [sparse_init()][sparse_init] (called via `setup_arch() ->
x86_init.paging.pagetable_init() -> paging_init()`) which in turn invokes
[sparse_init_nid()][sparse_init_nid] for each node. This allocates early boot
[memblock][memblock] memory via [sparse_buffer_init()][sparse_buffer_init] and
populates each section via
[__populate_section_memmap()][__populate_section_memmap] and
[vmemmap_populate()][vmemmap_populate] which does the heavy lifting via the
following call stack:

```
setup_arch()
x86_init.paging.pagetable_init()
paging_init()
sparse_init()
sparse_init_nid()
  sparse_buffer_init()
  __populate_section_memmap()
    vmemmap_populate()
```

Note that the [struct page][page]s allocated here are uninitialised. The
initialisation occurs elsewhere, also called from [paging_init()][paging_init]
and eventually invoking [__init_single_page()][__init_single_page] for each
individual `struct page` via the following call stack:

```
setup_arch()
x86_init.paging.pagetable_init()
paging_init()
zone_sizes_init()
free_area_init()
free_area_init_node()
free_area_init_core()
memmap_init()
memmap_init_zone()
__meminit()
__init_single_page()
```

### Accessing struct pages

Once the pages have been setup they can be accessed via
[pfn_to_page()][pfn_to_page] (and PFNs for a page obtained from
[page_to_pfn()][page_to_pfn]) which both invoke the memory model-specific
[__pfn_to_page()][__pfn_to_page] and [__page_to_pfn()][__page_to_pfn]:

```c
/* memmap is virtually contiguous.  */
#define __pfn_to_page(pfn)  (vmemmap + (pfn))
#define __page_to_pfn(page) (unsigned long)((page) - vmemmap)
```

Since x86-64 implements `CONFIG_SPARSEMEM_VMEMMAP` and thus provides virtually
contiguous `struct page`s this is a simple offset.

## Buddy allocator

Linux uses a [buddy allocator][buddy] to allocate physical memory. This is a
simple yet effective algorithm which allocates memory in `2^order` contiguous
page blocks.

Physical contiguity is important as device drivers often require a contiguous
block of physical memory and so the kernel needs to have the ability to
efficiently mete these out.

Each allocation aside from the top one ([MAX_ORDER][MAX_ORDER], typically 11 =
2,048 pages or 8 MiB) is paired with a 'buddy' and when it is freed, the two are
coalesced and freed at the next highest order, cascading as far as it can go.

The means for determining the buddy of a particular allocation is, by design,
O(1) - this is a fundamental part of what makes a buddy allocator fast. Linux's
is [__find_buddy_pfn()][__find_buddy_pfn]:

```c
/*
 * Locate the struct page for both the matching buddy in our
 * pair (buddy1) and the combined O(n+1) page they form (page).
 *
 * 1) Any buddy B1 will have an order O twin B2 which satisfies
 * the following equation:
 *     B2 = B1 ^ (1 << O)
 * For example, if the starting buddy (buddy2) is #8 its order
 * 1 buddy is #10:
 *     B2 = 8 ^ (1 << 1) = 8 ^ 2 = 10
 *
 * 2) Any buddy B will have an order O+1 parent P which
 * satisfies the following equation:
 *     P = B & ~(1 << O)
 *
 * Assumption: *_mem_map is contiguous at least up to MAX_ORDER
 */
static inline unsigned long
__find_buddy_pfn(unsigned long page_pfn, unsigned int order)
{
    return page_pfn ^ (1 << order);
}
```

The PFN of a higher order allocation is equal to the lowest page in the block,
so we can determine buddies by simply flipping the `order`th bit of the PFN. The
PFN of a page of a certain order will be aligned to `2^order` as each higher
order level comprises all the pages below it.

Visually:

```
...............................................................| order 6
...............................|...............................| order 5
...............|...............|...............|...............| order 4
.......|.......|.......|.......|.......|.......|.......|.......| order 3
...|...|...|...|...|...|...|...|...|...|...|...|...|...|...|...| order 2
.|.|.|.|.|.|.|.|.|.|.|.|.|.|.|.|.|.|.|.|.|.|.|.|.|.|.|.|.|.|.|.| order 1
|||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||| order 0
3210987654321098765432109876543210987654321098765432109876543210 PFN
   6         5         4         3         2         1
```

Here each `|` represents the PFN of the first page of each orders' allocation
(this is how allocation blocks are referenced).

And highlighting buddies:

```
[..............................................................] order 6
{..............................}[..............................] order 5
{..............}[..............]{..............}[..............] order 4
{......}[......]{......}[......]{......}[......]{......}[......] order 3
{..}[..]{..}[..]{..}[..]{..}[..]{..}[..]{..}[..]{..}[..]{..}[..] order 2
{}[]{}[]{}[]{}[]{}[]{}[]{}[]{}[]{}[]{}[]{}[]{}[]{}[]{}[]{}[]{}[] order 1
}]}]}]}]}]}]}]}]}]}]}]}]}]}]}]}]}]}]}]}]}]}]}]}]}]}]}]}]}]}]}]}] order 0
---|---------|---------|---------|---------|---------|----------
3210987654321098765432109876543210987654321098765432109876543210 PFN
   6         5         4         3         2         1
```

Here each adjacent `]` and `}` represent the start PFN of buddies. For example,
order 3 buddies exist at (0, 8), (16, 24), etc.

When pages are freed in the buddy allocator the buddy is checked and, if free,
the two are coalesced into a block at the order level above. The process is
repeated until a buddy is found not to be free. This way memory of each order is
efficiently refilled as blocks are freed.

When pages are allocated the free list at the requested order level is checked -
if a free block can be found then that is returned otherwise the order levels
above are checked until an available block is located. This block is then split
until a block of the order required is obtained and this is returned.

As a consequence of this at least 1 additional block is left at each level above
the allocation meaning that future allocations can be performed more
efficiently. It also ensures that block sizes are divided according to need (and
coalesced again as soon as blocks at any given order are no longer required).

### Allocator initialisation

Once the kernel has set up the [direct physical mapping][direct-phys-mapping],
the [struct page][page]s and the vmemmap to the struct pages, all of the
[memblock][memblock] allocated memory is put into the buddy allocator by each
individual page being freed via the following call stack:

```
start_kernel()
mm_init()
mem_init()
memblock_free_all()
free_low_memory_core_early()
__free_memory_core()
__free_pages_memory()
memblock_free_pages()
__free_pages_core()
__free_pages_ok()
free_one_page()
__free_one_page()
```

### Page allocation

The [__alloc_pages_nodemask()][__alloc_pages_nodemask] function is the core
function performing page allocation.

[numa]:https://en.wikipedia.org/wiki/Non-uniform_memory_access
[buddy]:https://en.wikipedia.org/wiki/Buddy_memory_allocation
[pg_data_t]:https://github.com/torvalds/linux/blob/45e885c439e825c19f3a51e46ef8210984bc0a9c/include/linux/mmzone.h#L726
[zone]:https://github.com/torvalds/linux/blob/45e885c439e825c19f3a51e46ef8210984bc0a9c/include/linux/mmzone.h#L448
[MAX_ORDER]:https://github.com/torvalds/linux/blob/45e885c439e825c19f3a51e46ef8210984bc0a9c/include/linux/mmzone.h#L27
[lowmem_reserve_ratio]:https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/sysctl/vm.rst#lowmem_reserve_ratio
[setup_per_zone_lowmem_reserve]:https://github.com/torvalds/linux/blob/5c8fe583cce542aa0b84adc939ce85293de36e5e/mm/page_alloc.c#L7871
[kernelcore]:https://github.com/torvalds/linux/blob/5c8fe583cce542aa0b84adc939ce85293de36e5e/Documentation/admin-guide/kernel-parameters.txt#L2128
[movablecore]:https://github.com/torvalds/linux/blob/5c8fe583cce542aa0b84adc939ce85293de36e5e/Documentation/admin-guide/kernel-parameters.txt#L2918
[zone_device-doc]:https://github.com/torvalds/linux/blob/master/Documentation/vm/memory-model.rst#zone_device
[__alloc_pages_nodemask]:https://github.com/torvalds/linux/blob/45e885c439e825c19f3a51e46ef8210984bc0a9c/mm/page_alloc.c#L4917
[__find_buddy_pfn]:https://github.com/torvalds/linux/blob/5c8fe583cce542aa0b84adc939ce85293de36e5e/mm/internal.h#L174
[setup_per_zone_wmarks]:https://github.com/torvalds/linux/blob/c76e02c59e13ae6c22cc091786d16c01bee23a14/mm/page_alloc.c#L7969
[__setup_per_zone_wmarks]:https://github.com/torvalds/linux/blob/c76e02c59e13ae6c22cc091786d16c01bee23a14/mm/page_alloc.c#L7900
[init_per_zone_wmark_min]:https://github.com/torvalds/linux/blob/c76e02c59e13ae6c22cc091786d16c01bee23a14/mm/page_alloc.c#L8002
[watermark_scale_factor]:https://sysctl-explorer.net/vm/watermark_scale_factor
[page]:https://github.com/torvalds/linux/blob/c76e02c59e13ae6c22cc091786d16c01bee23a14/include/linux/mm_types.h#L69
[sparsemem]:https://github.com/torvalds/linux/blob/master/Documentation/vm/memory-model.rst#sparsemem
[sparse_init]:https://github.com/torvalds/linux/blob/c76e02c59e13ae6c22cc091786d16c01bee23a14/mm/sparse.c#L575
[sparse_init_nid]:https://github.com/torvalds/linux/blob/c76e02c59e13ae6c22cc091786d16c01bee23a14/mm/sparse.c#L523
[sparse_buffer_init]:https://github.com/torvalds/linux/blob/c76e02c59e13ae6c22cc091786d16c01bee23a14/mm/sparse.c#L474
[memblock]:https://0xax.gitbooks.io/linux-insides/content/MM/linux-mm-1.html
[__populate_section_memmap]:https://github.com/torvalds/linux/blob/dea8dcf2a9fa8cc540136a6cd885c3beece16ec3/mm/sparse-vmemmap.c#L251
[vmemmap_populate]:https://github.com/torvalds/linux/blob/dea8dcf2a9fa8cc540136a6cd885c3beece16ec3/arch/x86/mm/init_64.c#L1554
[paging_init]:https://github.com/torvalds/linux/blob/dea8dcf2a9fa8cc540136a6cd885c3beece16ec3/arch/x86/mm/init_64.c#L813
[__init_single_page]:https://github.com/torvalds/linux/blob/dea8dcf2a9fa8cc540136a6cd885c3beece16ec3/mm/page_alloc.c#L1451
[pfn_to_page]:https://github.com/torvalds/linux/blob/dea8dcf2a9fa8cc540136a6cd885c3beece16ec3/include/asm-generic/memory_model.h#L82
[page_to_pfn]:https://github.com/torvalds/linux/blob/dea8dcf2a9fa8cc540136a6cd885c3beece16ec3/include/asm-generic/memory_model.h#L81
[__pfn_to_page]:https://github.com/torvalds/linux/blob/dea8dcf2a9fa8cc540136a6cd885c3beece16ec3/include/asm-generic/memory_model.h#L54
[__page_to_pfn]:https://github.com/torvalds/linux/blob/dea8dcf2a9fa8cc540136a6cd885c3beece16ec3/include/asm-generic/memory_model.h#L55
[direct-phys-mapping]:/virt_layout.md#direct-physical-memory-mapping
