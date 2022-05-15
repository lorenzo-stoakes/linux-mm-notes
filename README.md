# Linux Memory Management

**IMPORTANT**: This repository is deprecated in favour of a new project to write
a book about the mm subsystem - I will be developing it (over a long period!)
see my homepage [ljs.io](https://ljs.io) for updates. This will supercede this
project and all future efforts to finally write something comprehensive will be
made over there.

## Contents

* [Virtual memory layout][virt_layout] - x86-64 page table structure, virtual
  memory address space and conversion between physical and virtual memory.

* [Physical page allocation][phys_alloc] - NUMA nodes, zones, and the physical
  page (buddy) allocator.

* [Memory reclaim][reclaim] - Memory reclaim, compaction and OOM - i.e. how the
  kernel finds more memory when zones drop below low and minimum thresholds.

## Descriptions

This is a set of notes on the linux memory management subsystem.

They assume an x86-64 architecture and all architecture-specific references to
kernel code will reference x64-64 specific data structures and code.

Links to actual code will be taken from the current tip (permalinked via github)
at circa-5.10. I may try to stabilise this at a specific release at some point
but can't guarantee that at the moment.

These notes, unlike my [originals][0], will make little to no effort to explain
absolutely basic concepts, but rather assume that the reader understands or can
research these kinds of things.

[0]:https://github.com/lorenzo-stoakes/linux-vm-notes

[virt_layout]:virt_layout.md
[phys_alloc]:phys_alloc.md
[reclaim]:reclaim.md
