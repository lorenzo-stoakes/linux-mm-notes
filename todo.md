## TODO

* Page table locking.

* `mm_struct`, VMAs.

* Slab allocator, `kmalloc()`, `kfree()`, `kmem_cache_alloc()`, `kmem_cache_free()`.

* GFP flags.

* `vmalloc()`.

* Page faulting, allocation of userland memory -> `alloc_pages_vma()`.

* Unified page cache.

* Dirty pages and write-back.

* CMA?

* CoW

* file-backed memory.

* `ioremap()` :)

* hugetlbfs vs. transparent huge pages.

* `struct page -> _mapcount`?

* `fpi_flags`, gfp flags.

* `head_compound_pincount()` etc.

* `shrink_lruvec()` and swapping. https://linux-mm.org/Swapout

* per-cpu pages - `free_the_page()` etc.

* `compaction_capture()`.

* isolation migratetype, esp. as referenced in `__free_one_page()`.

* ref counts on pages.

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
