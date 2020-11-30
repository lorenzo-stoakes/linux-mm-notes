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

init/main.c

mem_init()
memblock_free_all()
preallocate_vmalloc_pages()
```

* What is the meaning of e.g. ZONE_DMA in a second node, if the physical memory
  there starts at a later PA? Or does the socket in a separate node perceive the
  PAs as starting from 0 again (surely not due to page tables etc.)?

* Which code is responsible for assigning pages to zones + nodes?

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
