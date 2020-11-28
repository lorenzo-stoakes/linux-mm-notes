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
mem_init()
memblock_free_all()
preallocate_vmalloc_pages()
```

* What is the meaning of e.g. ZONE_DMA in a second node, if the physical memory
  there starts at a later PA? Or does the socket in a separate node perceive the
  PAs as starting from 0 again (surely not due to page tables etc.)?

* What happens if the system has e.g. 2 GiB of RAM - is `ZONE_DMA32`/`ZONE_DMA`
  populated but not `ZONE_NORMAL`? Which code is responsible for assigning pages
  to zones + nodes?
