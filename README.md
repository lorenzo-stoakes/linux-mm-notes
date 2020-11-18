# Linux MM Notes

## Contents

* [Fundamentals][p1] - Fundamental principles of linux-mm.
* [OOM Killer][p2] - Description of the OOM killer.
* [Raw][p3] - Rough unpolished notes.

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
[1]:https://ljs.io/patches.html

[p1]:fundamentals.md
[p2]:oom.md
[p3]:raw.md
