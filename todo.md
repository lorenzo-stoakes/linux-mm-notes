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
