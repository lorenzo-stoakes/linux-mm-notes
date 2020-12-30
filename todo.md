## TODO

* Page table locking.

* `mm_struct`, VMAs.

* `struct page`s etc.

* Buddy allocator, order, etc. - `__alloc_pages_nodemask()` in
  `page_alloc.c`. `alloc_page() -> struct page*`

* Slab allocator, `kmalloc()`, `kfree()`, `kmem_cache_alloc()`, `kmem_cache_free()`.

* GFP flags.

* `vmalloc()`.

* Page faulting, allocation of userland memory -> `alloc_pages_vma()`.

* Nodes, zones, etc. i.e. NUMA.

* Unified page cache.

* Dirty pages and write-back.

* CMA?

* CoW

* sparse mem sections.

* file-backed memory.

### Questions

* Initial mapping of physical mapping, vmem, etc.?

```
arch/x86/mm/init.c:

init_mem_mapping() ->
  memory_map_top_down(), memory_map_bottom_up() -> init_range_memory_mapping()
  init_memory_mapping()
  phys_p**_init()

arch/x86/mm/init_64.c:
  kernel_physical_mapping_init()
  __kernel_physical_mapping_init()

arch/x86/mm/numa.c:alloc_node_data()

preallocate_vmalloc_pages()
```

* Which code is responsible for assigning pages to zones + nodes?

`numa_init()`?

* Is there a way to determine how many movable pages there are specifically
  per-zone/order/etc.? It seems the `show_free_areas()` output lists _all_
  migrate types available at each order level.

* `ioremap()` :)

* compound pages:

```c
/*
 * Higher-order pages are called "compound pages".  They are structured thusly:
 *
 * The first PAGE_SIZE page is called the "head page" and have PG_head set.
 *
 * The remaining PAGE_SIZE pages are called "tail pages". PageTail() is encoded
 * in bit 0 of page->compound_head. The rest of bits is pointer to head page.
 *
 * The first tail page's ->compound_dtor holds the offset in array of compound
 * page destructors. See compound_page_dtors.
 *
 * The first tail page's ->compound_order holds the order of allocation.
 * This usage means that zero-order pages may not be compound.
 */

void free_compound_page(struct page *page)
{
    mem_cgroup_uncharge(page);
    __free_pages_ok(page, compound_order(page), FPI_NONE);
}
```

* hugetlbfs vs. transparent huge pages.

* `struct page -> _mapcount`?

* `fpi_flags`, gfp flags.

* free lists

* per-cpu pages

## virt_layout

```
__pa_symbol()
```

### phys_alloc

https://github.com/torvalds/linux/blob/master/Documentation/vm/memory-model.rst

```
CONFIG_SPARSEMEM
vmemmap_populate
pfn_to_page(), page_to_pfn()
```

Initial phys mem -> buddy etc.

```
physical init:

init/main.c

mem_init()
memblock_free_all()
```

e820!

## Handy

```
/proc/zoneinfo # Per-zone memory info -> spanned, present, managed
/proc/meminfo
/proc/vmstat
/proc/vmallocinfo
/proc/pressure/memory
/proc/slabinfo
/proc/iomem

/proc/buddyinfo
/proc/pagetypeinfo

/proc/*/maps
/proc/*/smaps
/proc/*/status
/proc/*/numa_maps

From /proc/meminfo:
DirectMap4k:       32628 kB
DirectMap2M:     4161536 kB
DirectMap1G:    29360128 kB
```

Tunables... https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/sysctl/vm.rst

## articles

[lwn-zone-device]:https://lwn.net/Articles/717555/
