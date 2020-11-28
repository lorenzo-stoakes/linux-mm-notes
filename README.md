# Linux MM Notes

## Contents

* [Virtual memory layout][virt_layout] - x86-64 page table structure, virtual
  memory address space and conversion between physical and virtual memory.

* [Physical page allocation][phys_alloc] - NUMA nodes, zones, and the physical
  page (buddy) allocator.

### WIP

* [OOM Killer][oom] - Description of the OOM killer.
* [Raw][raw] - Rough unpolished notes.

## Descriptions

This is a set of notes on the linux memory management subsystem.

They assume an x86-64 architecture (with 4 levels of pagetables) and all
references to kernel code will reference x64-64 specific data structures and
code.

Links to actual code will be taken from the current tip (permalinked via github)
but obviously anything might change at any time.

These notes, unlike my [originals][0] will make little to no effort to explain
underlying core concepts, rather they will assume that the reader understands
the basic principles.

[0]:https://github.com/lorenzo-stoakes/linux-vm-notes

[virt_layout]:virt_layout.md
[phys_alloc]:phys_alloc.md

[oom]:oom.md
[raw]:raw.md
