# Virtual memory layout

## Page tables

[Page tables][page-table] are a series of memory pages which sparsely map
virtual memory addresses to physical ones. The x86-64 CPU has a specific
register where the physical address of the top-level page table (PGD) is placed
and the CPU hardware then uses this to decode all memory addresses used.

Each entry in a page table must be aligned to 4 KiB and can only use the number
of available bits allowed for virtual addresses (48 for 4-level x86-64, 57 for
5-level) which leaves a number of bits which can be used as flags to indicate
various properties of those pages.

### Declarations

Each page table entry is defined as an `unsigned long` (declared as
[pxxval_t][0] types) i.e. `uint64_t` values.

Some makeshift type safety is enforced by wrapping each of these types in a
struct via e.g. `typedef struct { pgdval_t pgd; } pgd_t`.

The page table structures vary depending on whether [Intel 5-level paging][ref0]
is enabled.

### Levels

| Level | Type | Count | Shift | Description |
|-------|------|-------|-------|-------------|
| PGD | [pgd_t][1] | [PTRS_PER_PGD][6] (512) | [PGDIR_SHIFT][13] (48 or 39)\* | Page Global Directory |
| P4D | [p4d_t][2] | [PTRS_PER_P4D][7] (512 or 1) | [P4D_SHIFT][14] (39) | Page 4 Directory\* |
| PUD | [pud_t][3] | [PTRS_PER_PUD][10] (512) | [PUD_SHIFT][14] (30) | Page Upper Directory \*\* |
| PMD | [pmd_t][4] | [PTRS_PER_PMD][11] (512) | [PMD_SHIFT][15] (21) | Page Middle Directory\*\* |
| PTE | [pte_t][5] | [PTRS_PER_PTE][12] (512) | [PAGE_SHIFT][16] (12) | Page Table Entry directory |

\* The number of P4D entries depends on whether `CONFIG_X86_5LEVEL` is set and
whether the hardware supports it. `PTRS_PER_P4D` is stored in the global
[ptrs_per_p4d][8] value and determined in [check_la57_support()][9] on boot. It
is set to 512 if enabled, otherwise 1. `PGDIR_SHIFT` is set to 39 (replacing the
P4D offset) if 5-level is not enabled.

\*\* PMD and PTE directory entries are skipped (and the PUD treated as a PTE) if
the PUD is marked huge (1 GiB page size), the PTE directory entry is skipped if
the PMD is marked huge (2 MiB page size).

### Virtual address layout

Note that the bits from the MSB up to AND INCLUDING the first bit of the
addressable range (57 bits for 5-level, 48 bits for 4-level) must all be set the
same otherwise the processor will fault (denoted as 'xxxx...' below).

As a convention the kernel sets x=1 for kernel addresses and x=0 for userland
addresses making it easy to differentiate between the two and placing kernel
mappings in the upper half of the PGD.

The valid address range is determined by [__VIRTUAL_MASK][24] and
[__VIRTUAL_MASK_SHIFT][25] (56 for 5-level, 47 for 4-level, note that number of
bits = shift + 1).

#### 5-level

```
xxxxxxxx                       57 bits
<-----><------------------------------------------------------->
   6         5         4         3         2         1
---|---------|---------|---------|---------|---------|----------
3210987654321098765432109876543210987654321098765432109876543210
       [  PGD  ][  P4D  ][  PUD  ][  PMD  ][  PTE  ][  OFFSET  ]
111111111111111111111111111111111111111111100000000|000000000000  PMD_MASK
111111111111111111111111111111111100000000|00000000|000000000000  PUD_MASK
111111111111111111111111100000000|00000000|00000000|000000000000  P4D_MASK
111111111111111100000000|00000000|00000000|00000000|000000000000  PGDIR_MASK
               |        |        |        |        |
               |        |        |        |        |------------- PAGE_SHIFT  (12)
               |        |        |        |---------------------- PMD_SHIFT   (21)
               |        |        |------------------------------- PUD_SHIFT   (30)
               |        |---------------------------------------- P4D_SHIFT   (39)
               |------------------------------------------------- PGDIR_SHIFT (48)
```

#### 4-level

```
xxxxxxxxxxxxxxxxx                   48 bits
<--------------><---------------------------------------------->
   6         5         4         3         2         1
---|---------|---------|---------|---------|---------|----------
3210987654321098765432109876543210987654321098765432109876543210
                [  PGD  ][  PUD  ][  PMD  ][  PTE  ][  OFFSET  ]
111111111111111111111111111111111111111111100000000|000000000000  PMD_MASK
111111111111111111111111111111111100000000|00000000|000000000000  PUD_MASK
111111111111111111111111100000000|00000000|00000000|000000000000  PGDIR_MASK
                        |        |        |        |
                        |        |        |        |
                        |        |        |        |------------- PAGE_SHIFT  (12)
                        |        |        |---------------------- PMD_SHIFT   (21)
                        |        |------------------------------- PUD_SHIFT   (30)
                        |---------------------------------------- PGDIR_SHIFT (39)
```

### Flags

Declared in [arch/x86/include/asm/pgtable_types.h][17].

```
   6         5         4         3         2         1
---|---------|---------|---------|---------|---------|----------
3210987654321098765432109876543210987654321098765432109876543210
||||||                                              ||||||||||||
||||||                                              ||||||||||||- _PAGE_PRESENT    - Present (i.e. available)
||||||                                              |||||||||||-- _PAGE_RW         - Writable
||||||                                              ||||||||||--- _PAGE_USER       - User accessible
||||||                                              |||||||||---- _PAGE_PWT        - Page write-through
||||||                                              ||||||||----- _PAGE_PCD        - Page cache disabled
||||||                                              |||||||------ _PAGE_ACCESSED   - Was accessed
||||||                                              ||||||------- _PAGE_DIRTY      - Was written to
||||||                                              |||||-------- _PAGE_PSE        - Is huge
||||||                                              ||||--------- _PAGE_GLOBAL     - TLB not cleared on TLB flush
||||||                                              |||---------- _PAGE_SPECIAL    - Page marked 'special' *
||||||                                              ||----------- _PAGE_UFFD_WP    - userfaultfd write-protected *
||||||                                              |------------ _PAGE_SOFT_DIRTY - Software-tracked dirty bit *
||||||----------------------------------------------------------- _PAGE_DEVMAP     - User-definable *
|||||------------------------------------------------------------ _PAGE_PKEY_BIT0  - Intel memory protection key bits
||||------------------------------------------------------------- _PAGE_PKEY_BIT1  - Intel memory protection key bits
|||-------------------------------------------------------------- _PAGE_PKEY_BIT2  - Intel memory protection key bits
||--------------------------------------------------------------- _PAGE_PKEY_BIT3  - Intel memory protection key bits
|---------------------------------------------------------------- _PAGE_NX         - No execute
```

\* Each of these bits are actually software-defined bits as far as the CPU is
concerned, however they are used this way by the kernel. `_PAGE_SPECIAL` also
doubles up as `_PAGE_CPA_TEST`.

The flags for a page table entry are obtained via `pXX_flags()` declared in
[pgtable_types.h][17].

The hardware will set `_PAGE_ACCESSED` once the page is first accessed, and once
set it remains set (sticky bit). It also sets `_PAGE_DIRTY` if data is written
to the page (which we can later clear).

### Aggregated flags

There are some flags that aggregate a number of flags to represent common page
flag configurations located at [arch/x86/include/asm/pgtable_types.h][17]:

| Name | PRESENT | RW | USER | ACCESSED | NX | DIRTY | PSE | GLOBAL | PWT | PCD |
|------|---------|----|------|----------|----|-------|-----|--------|-----|-----|
| PAGE_NONE |  |  |  | x |  |  |  | x |  |  |
| PAGE_SHARED | x | x | x | x | x |  |  |  |  |  |
| PAGE_SHARED_EXEC | x | x | x | x |  |  |  |  |  |  |
| PAGE_COPY_NOEXEC | x |  | x | x | x |  |  |  |  |  |
| PAGE_COPY_EXEC | x |  | x | x |  |  |  |  |  |  |
| PAGE_COPY | x |  | x | x | x |  |  |  |  |  |
| PAGE_READONLY | x |  | x | x | x |  |  |  |  |  |
| PAGE_READONLY_EXEC | x |  | x | x |  |  |  |  |  |  |
| PAGE_KERNEL | x | x |  | x | x | x |  | x |  |  |
| PAGE_KERNEL_EXEC | x | x |  | x |  | x |  | x |  |  |
| _KERN_PG_TABLE | x | x |  | x |  | x |  |  |  |  |
| _PAGE_TABLE | x | x | x | x |  | x |  |  |  |  |
| PAGE_KERNEL_RO | x |  |  | x | x | x |  | x |  |  |
| PAGE_KERNEL_ROX | x |  |  | x |  | x |  | x |  |  |
| PAGE_KERNEL_NOCACHE | x | x |  | x | x | x |  | x |  | x |
| PAGE_KERNEL_VVAR | x |  | x | x | x | x |  | x |  |  |
| PAGE_KERNEL_LARGE | x | x |  | x | x | x | x | x |  |  |
| PAGE_KERNEL_LARGE_EXEC | x | x |  | x |  | x | x | x |  |  |
| PAGE_KERNEL_WP | x | x |  | x | x | x |  | x | x |  |

### Page size

* By default page size is equal to __4 KiB__ (12 bits of page offset), however x86-64
  supports 1 GiB and 2 MiB page sizes.

* __1 GiB__ pages are enabled by setting the `_PAGE_PSE` bit in the PUD
  entry. This limits page table traversal to the PGD, P4D, PUD levels, leaving
  `12 + 9 + 9 = 30` bits of page offset and thus 1 GiB available per-page.

* __2 MiB__ pages are enabled by setting the `_PAGE_PSE` bit in the PMD
  entry. This limits page table traversal to the PGD, P4D, PUD, PMD levels,
  leaving `12 + 9 = 21` bits of page page offset and thus 2 MiB available
  per-page.

### Page table flag predicates

The predicates are declared in [arch/x86/include/asm/pgtable.h][18]

| Flag | G | 4 | U | M | T | Function |
|------|---|---|---|---|---|----------|
| _PAGE_PRESENT | x | x | x | x | x | `pXX_present()` |
| _PAGE_RW |  |  | x | x | x | `pXX_write()` |
| _PAGE_USER |  |  |  |  |  | N/A |
| _PAGE_PWT |  |  |  |  |  | N/A |
| _PAGE_PCD |  |  |  |  |  | N/A |
| _PAGE_ACCESSED | | | x | x | x | `pXX_young()` |
| _PAGE_DIRTY | | | x | x | x | `pXX_dirty()` |
| _PAGE_PSE | | | x | x |  | `pXX_huge()` or `pXX_large()` \* |
| _PAGE_GLOBAL | | | |  | x | `pte_global()` |
| _PAGE_SPECIAL | | | |  | x | `pte_special()` |
| _PAGE_UFFD_WP | | | | x | x | `pXX_uffd_wp()` |
| _PAGE_SOFT_DIRTY | | | x | x | x | `pXX_soft_dirty()` |
| _PAGE_DEVMAP | | | x | x | x | `pXX_devmap()` |
| _PAGE_PKEY_BIT0 |  |  |  |  |  | N/A |
| _PAGE_PKEY_BIT1 |  |  |  |  |  | N/A |
| _PAGE_PKEY_BIT2 |  |  |  |  |  | N/A |
| _PAGE_PKEY_BIT3 |  |  |  |  |  | N/A |
| _PAGE_NX |  |  |  |  | x | `pte_exec()` |

\* Note that [pud_huge][19] and [pmd_huge][20] are declared in
`arch/x86/mm/hugetlbpage.h` and implemented in `hugetlbpage.c` and have
hugetlb-specific semantics (present flag can't be set for PMD). The
`pud_large()` predicate also asserts the page is present.

Typically these flags can be set via `pXX_mkYYY()` e.g. `pte_mkyoung()`.

There are also a few other predicates that don't rely on specific flags:

| G | 4 | U | M | T | Function |
|---|---|---|---|---|----------|
| x | x | x | x | x | `pXX_none()` - Determines if entry is empty |
| x | x | x | x | x | `pXX_bad()` - Determines if the entry is corrupted |
|   | x | x | x | x | `pXX_pgprot()` - Returns flags cast to [pgprot_t][21] |

## Available address space

* In 4-level mode there are `512 * 1 * 512 * 512 * 512` = 68.7bn 4KiB pages =
  __256 TiB__ of address space. Linux divides this into __128 TiB__ of userland
  address space and __128 TiB__ of kernel address space. A direct memory mapping of
  all physical memory of 64 TiB is present in the kernel address space so the
  practical maximum memory capacity is __64 TiB__.

* In 5-level mode there are `512 * 512 * 512 * 512 * 512` = 35.2tn 4KiB pages =
  __128 PiB__ of address space. Linux divides this into __64 PiB__ of userland
  address space and __64 PiB__ of kernel address space.  A direct memory mapping
  of all physical memory of 32 PiB is present in the kernel address space so the
  practical maximum memory capacity is __32 PiB__.

## Address space layout

See the [kernel documentation][22] for full details.

Note that with `CONFIG_RANDOMIZE_MEMORY` enabled the actual base of the direct
memory mapping, vmalloc/ioremap space and the virtual memory map are randomised
by [arch/x86/mm/kaslr.c][23]. The layout remains the same but the base of these
regions is offset.

### 4-level

```
Userland (128 TiB)
                        0000000000000000 -> |---------------| ^
                                            |    Process    | |
                                            |    address    | | 128 TiB
                                            |     space     | |
                        0000800000000000 -> |---------------| v
                     .        ` .     -                 `-       ./   _
                              _    .`   -   The netherworld of  `/   `
                    -     `  _        |  /      unavailable sign-extended -/ .
                     ` -        .   `  48-bit address space  -     \  /    -
                   \-                - . . . .             \      /       -
Kernel (128 TiB)
                        ffff800000000000 -> |----------------| ^
                                            |   Hypervisor   | |
                                            |    reserved    | | 8 TiB
                                            |      space     | |
                        ffff880000000000 -> |----------------| x
                                            | LDT remap for  | | 0.5 TiB
                                            |       PTI      | |
[kaslr]   PAGE_OFFSET = ffff888000000000 -> |----------------| x
                                            | Direct mapping | |
                                            |  of all phys.  | | 64 TiB
                                            |     memory     | |
                        ffffc88000000000 -> |----------------| v
                                            /                /
                                            \     unused     \
                                            /      hole      /
                                            \                \
[kaslr] VMALLOC_START = ffffc90000000000 -> |----------------| ^
                                            |    vmalloc/    | |
                                            |    ioremap     | | 32 TiB
                                            |     space      | |
      VMALLOC_END + 1 = ffffe90000000000 -> |----------------| v
                                            /                /
                                            \     unused     \
                                            /      hole      /
                                            \                \
[kaslr] VMEMMAP_START = ffffea0000000000 -> |----------------| ^
                                            |     Virtual    | |
                                            |   memory map   | | 1 TiB
                                            |  (struct page  | |
                                            |     array)     | |
                        ffffeb0000000000 -> |----------------| v
                                            /                /
                                            \     unused     \
                                            /      hole      /
                                            \                \
                        ffffec0000000000 -> |----------------| ^
                                            |  KASAN shadow  | | 16 TiB
                                            |     memory     | |
                        fffffc0000000000 -> |----------------| v
                                            /                /
                                            \     unused     \
                                            /      hole      /
                                            \                \
                        fffffe0000000000 -> |----------------| ^
                                            | cpu_entry_area | | 0.5 TiB
                                            |     mapping    | |
                        fffffe8000000000 -> |----------------| v
                                            /                /
                                            \     unused     \
                                            /      hole      /
                                            \                \
     ESPFIX_BASE_ADDR = ffffff0000000000 -> |----------------| ^
                                            |   %esp fixup   | | 0.5 TiB
                                            |     stacks     | |
                        ffffff8000000000 -> |----------------| v
                                            /                /
                                            \     unused     \
                                            /      hole      /
                                            \                \
           EFI_VA_END = ffffffef00000000 -> |----------------| ^
                                            |   EFI region   | | 64 GiB
                                            | mapping space  | |
         EFI_VA_START = ffffffff00000000 -> |----------------| v
                                            /                /
                                            \     unused     \
                                            /      hole      /
                                            \                \
   __START_KERNEL_map = ffffffff80000000 -> |----------------| ^
                                            |     Kernel     | |
                                            |      text      | | KERNEL_IMAGE_SIZE = 1 GiB *
                                            |     mapping    | |
        MODULES_VADDR = ffffffffc0000000 -> |----------------| x *
                                            |     Module     | |
                                            |    mapping     | | 1 GiB *
                                            |     space      | |
                        ffffffffff600000 -> |----------------| x
                                            |   vsyscalls    | | 8 MiB
                        ffffffffffe00000 -> |----------------| v
                                            /                /
                                            \     unused     \
                                            /      hole      /
                                            \                \
                                            ------------------
```

(these notes apply to 5-level also)

\* Note that `MODULES_VADDR = __START_KERNEL_map + KERNEL_IMAGE_SIZE` and
`KERNEL_IMAGE_SIZE` varies depending on whether KASLR (via
`CONFIG_RANDOMIZE_BASE`) is enabled - with it enabled it is equal to 1 GiB as
shown in the diagram, otherwise it is equal to 512 MiB and modules get 1.5 GiB
rather than 1 GiB.

The kernel text mapping section is _ostensibly_ mapped to PA 0 (KASLR means that
it usually isn't but page tables are fixed up on early initialisation, see
section below on VA to PA conversions for more details) and due to early kernel
identity mapping requirements it is physically contiguous meaning that the
actual data there begins at `__START_KERNEL_map` + [CONFIG_PHYSICAL_START][36].

### 5-level

```
Userland (64 PiB)
                        0000000000000000 -> |---------------| ^
                                            |    Process    | |
                                            |    address    | | 64 PiB
                                            |     space     | |
                        0100000000000000 -> |---------------| v
                     .        ` .     -                 `-       ./   _
                              _    .`   -   The netherworld of  `/   `
                    -     `  _        |  /      unavailable sign-extended -/ .
                     ` -        .   `  57-bit address space  -     \  /    -
                   \-                - . . . .             \      /       -
Kernel (64 PiB)
                        ff00000000000000 -> |----------------| ^
                                            |   Hypervisor   | |
                                            |    reserved    | | 4 PiB
                                            |      space     | |
                        ff10000000000000 -> |----------------| x
                                            | LDT remap for  | | 0.25 PiB
                                            |       PTI      | |
[kaslr]   PAGE_OFFSET = ff11000000000000 -> |----------------| x
                                            | Direct mapping | |
                                            |  of all phys.  | | 32 PiB
                                            |     memory     | |
                        ff91000000000000 -> |----------------| v
                                            /                /
                                            \     unused     \
                                            /      hole      /
                                            \                \
[kaslr] VMALLOC_START = ffa0000000000000 -> |----------------| ^
                                            |    vmalloc/    | |
                                            |    ioremap     | | 12.5 PiB
                                            |     space      | |
      VMALLOC_END + 1 = ffd2000000000000 -> |----------------| v
                                            /                /
                                            \     unused     \
                                            /      hole      /
                                            \                \
[kaslr] VMEMMAP_START = ffd4000000000000 -> |----------------| ^
                                            |     Virtual    | |
                                            |   memory map   | | 0.5 PiB
                                            |  (struct page  | |
                                            |     array)     | |
                        ffd6000000000000 -> |----------------| v
                                            /                /
                                            \     unused     \
                                            /      hole      /
                                            \                \
                        ffdf000000000000 -> |----------------| ^
                                            |  KASAN shadow  | | 8 PiB
                                            |     memory     | |
                        fffffc0000000000 -> |----------------| v
                                            /                /
                                            \     unused     \
                                            /      hole      /
                                            \                \
                        fffffe0000000000 -> |----------------| ^
                                            | cpu_entry_area | | 0.5 TiB
                                            |     mapping    | |
                        fffffe8000000000 -> |----------------| v
                                            /                /
                                            \     unused     \
                                            /      hole      /
                                            \                \
     ESPFIX_BASE_ADDR = ffffff0000000000 -> |----------------| ^
                                            |   %esp fixup   | | 0.5 TiB
                                            |     stacks     | |
                        ffffff8000000000 -> |----------------| v
                                            /                /
                                            \     unused     \
                                            /      hole      /
                                            \                \
           EFI_VA_END = ffffffef00000000 -> |----------------| ^
                                            |   EFI region   | | 64 GiB
                                            | mapping space  | |
         EFI_VA_START = ffffffff00000000 -> |----------------| v
                                            /                /
                                            \     unused     \
                                            /      hole      /
                                            \                \
   __START_KERNEL_map = ffffffff80000000 -> |----------------| ^
                                            |     Kernel     | |
                                            |      text      | | KERNEL_IMAGE_SIZE = 1 GiB *
                                            |     mapping    | |
        MODULES_VADDR = ffffffffc0000000 -> |----------------| x *
                                            |     Module     | |
                                            |    mapping     | | 1 GiB *
                                            |     space      | |
                        ffffffffff600000 -> |----------------| x
                                            |   vsyscalls    | | 8 MiB
                        ffffffffffe00000 -> |----------------| v
                                            /                /
                                            \     unused     \
                                            /      hole      /
                                            \                \
                                            ------------------
```

## Converting between physical and virtual addresses

### PA to kernel VA

Given the memory map as described above and the fact that the kernel maps in the
entire physical address space (one of the great benefits of such a huge address
space), conversion from a physical address to a virtual one is simple and the
base implementation is provided by [__va()][26]:

```c
#define __va(x)         ((void *)((unsigned long)(x)+PAGE_OFFSET))
```

We simply offset the physical address by the address of the base of the physical
mapping and we're done.

A more palatable version of this might be [phys_to_virt()][27] which makes the
types explicit:

```c
static inline void *phys_to_virt(phys_addr_t address)
{
    return __va(address);
}
```

`phys_addr_t` is simply an alias for `unsigned long` i.e. 64-bit unsigned
integer on x86-64.

### Kernel VA to PA

Conversion in the other direction is trickier as we have multiple ranges of
virtual addresses a VA might be using.

We retain the same naming convertion as above and define the raw implementation
of this as [__pa()][29]:

```c
#define __pa(x)     __phys_addr((unsigned long)(x))
```

Again, a more palatable version of this is [virt_to_phys()][30]:

```c
static inline phys_addr_t virt_to_phys(volatile void *address)
{
    return __pa(address);
}
```

Tracing through, [__phys_addr()][31] is an alias for [__phys_addr_nodebug()][32]:

```c
#define __phys_addr(x)      __phys_addr_nodebug(x)

...

static inline unsigned long __phys_addr_nodebug(unsigned long x)
{
    unsigned long y = x - __START_KERNEL_map;

    /* use the carry flag to determine if x was < __START_KERNEL_map */
    x = y + ((x > y) ? phys_base : (__START_KERNEL_map - PAGE_OFFSET));

    return x;
}
```

[phys_base][33] represents the physical _offset_ from [CONFIG_PHYSICAL_START][36] at
which the kernel text mappings start if the kernel has been relocated.

In x86-64 this defaults to 0, however it is [offset][35] by the [load_delta][34]
between the address the kernel code is linked against and the address it is
actually running at during early initialisation. KASLR in particular means there
will always be a delta here. See the [linux insides][ref1] [early
initialisation][ref2] documentation for more details.

In x86-64 we randomise both the physical offset of the kernel text mapping _and_
the virtual offset (in [handle_relocations()][41]) of the _text section via
[choose_random_location()][37] which is [invoked][38] in [extract_kernel()][39]
at kernel decompression.


This means that when [__startup_64()][40] is called, the `physaddr` parameter
and `_text` can be offset from one another when [load_delta][34] is calculated:

```c
    /*
     * Compute the delta between the address I am compiled to run at
     * and the address I am actually running at.
     */
    load_delta = physaddr - (unsigned long)(_text - __START_KERNEL_map);
```

As a result `load_delta` can be effectively negative (underflowed as unsigned)
and thus `phys_base` [can be set][35] to an underflowed negative uint64.

Coming back to the `__phys_addr_nodebug()` function, the purpose of this is to
determine whether the kernel address referenced is either part of the physical
memory mapping or the kernel text segment.

The code then either computes `addr + phys_base - __START_KERNEL_map` for a
kernel text section address or `addr - PAGE_OFFSET` for a physical memory
mapping address.

Note that we therefore cannot determine the PA of any address from another
source using this method.

We also have [__pa_symbol()][__pa_symbol] available for VAs that we know for a
fact live in the kernel text mappings, which in turn invokes
[__phys_addr_symbol()][__phys_addr_symbol] which is simply defined as:

```c
#define __phys_addr_symbol(x) \
    ((unsigned long)(x) - __START_KERNEL_map + phys_base)
```

## Direct physical memory mapping

In linux all processes share kernel page table mappings between them (or if
[Page table isolation][pti] ([kernel docs][pti_kernel_docs]) is available and
enabled via `CONFIG_PAGE_TABLE_ISOLATION`, can make use of PTI functionality in
order to reduce the impact of swapping out the PGD) meaning that no [TLB][tlb]
flush is required when transitioning from userland to kernel space. Additionally
kernel page table mappings don't get flushed as a result of the `_PAGE_GLOBAL`
page table flag being set.

As a result the initial page table layout (that will be used as a template for
all processes on the system) needs to be setup early in the initialisation
process so any kernel process can access any part of physical memory.

As part of this work we need to create the direct mapping of all physical memory
that begins at `PAGE_OFFSET` in virtual memory.

The initialisation is performed at a top level via
[start_kernel()][start_kernel] -> [setup_arch()][setup_arch] ->
[init_mem_mapping()][init_mem_mapping].

A simplified version of that function:

```c
void __init init_mem_mapping(void)
{
    unsigned long end;

    end = max_pfn << PAGE_SHIFT;

    /* the ISA range is always mapped regardless of memory holes */
    init_memory_mapping(0, ISA_END_ADDRESS, PAGE_KERNEL);

    /* Init the trampoline, possibly with KASLR memory offset */
    init_trampoline();

    /*
     * If the allocation is in bottom-up direction, we setup direct mapping
     * in bottom-up, otherwise we setup direct mapping in top-down.
     */
    if (memblock_bottom_up()) {
        unsigned long kernel_end = __pa_symbol(_end);

        /*
         * we need two separate calls here. This is because we want to
         * allocate page tables above the kernel. So we first map
         * [kernel_end, end) to make memory above the kernel be mapped
         * as soon as possible. And then use page tables allocated above
         * the kernel to map [ISA_END_ADDRESS, kernel_end).
         */
        memory_map_bottom_up(kernel_end, end);
        memory_map_bottom_up(ISA_END_ADDRESS, kernel_end);
    } else {
        memory_map_top_down(ISA_END_ADDRESS, end);
    }

    load_cr3(swapper_pg_dir);
    __flush_tlb_all();
}
```

We map up to the [max_pfn][max_pfn] physical Page Frame Number (PFN is the
index of a 4 KiB page, e.g. PFN 1 = PA 0x1000, etc.) a value determined during
memory hotplug discovery.

We start by mapping the x86 'ISA range' which is the first megabyte of memory,
i.e. the maximum that real-mode x86 can access.

After that we set the PGD entry used by the x86 [real mode][real-mode]
[trampoline][trampoline] used to initialise other cores.

Once this is done we're ready to perform the bulk of the memory mappings. Early
kernel uses the [memblock][memblock] simplified memory management system before
we are able to get the full-fledged mm running. This can allocate memory from
the bottom up or the top down, depending whether there are concerns around use
of hot-pluggable memory that could be removed during the process.

If we are in bottom-up mode we start by mapping post-kernel image memory which
will be where we're allocated page tables so naturally we need this first,
before mapping the kernel image itself. We determine the end of the kernel via
converting the (KASLR randomised) VA of the end of the text section to its PA.

Finally, once this memory is mapped we place [swapper_pg_dir][swapper_pg_dir]
(the PA of the first defined PGD which is part of the data section of the kernel
image) into the x86 register CR3 which is the register tracking PGD before
flushing the [TLB][tlb] to ensure all mappings are correctly picked up.

[memory_map_top_down()][memory_map_top_down] and
[memory_map_bottom_up()][memory_map_bottom_up] do roughly the same thing with
some slight finessing in the top-down case.

Taking a look at `memory_map_bottom_up()`:

```c
static void __init memory_map_bottom_up(unsigned long map_start,
                    unsigned long map_end)
{
    unsigned long next, start;
    unsigned long mapped_ram_size = 0;
    /* step_size need to be small so pgt_buf from BRK could cover it */
    unsigned long step_size = PMD_SIZE;

    start = map_start;
    min_pfn_mapped = start >> PAGE_SHIFT;

    /*
     * We start from the bottom (@map_start) and go to the top (@map_end).
     * The memblock_find_in_range() gets us a block of RAM from the
     * end of RAM in [min_pfn_mapped, max_pfn_mapped) used as new pages
     * for page table.
     */
    while (start < map_end) {
        if (step_size && map_end - start > step_size) {
            next = round_up(start + 1, step_size);
            if (next > map_end)
                next = map_end;
        } else {
            next = map_end;
        }

        mapped_ram_size += init_range_memory_mapping(start, next);
        start = next;

        if (mapped_ram_size >= step_size)
            step_size = get_new_step_size(step_size);
    }
}
```

We want to allocate page tables above the kernel image as lower memory is often
reserved or otherwise unavailable.

We start by assigning a minimum step size (i.e. the range of memory we are
mapping in one go) at `PMD_SIZE` (i.e. 2 MiB) in order that the early memory
mapping provides sufficient memory for the worst-case allocation. See the below
section on [alloc_low_pages()][alloc_low_pages] for more details but this is
necessary as we have a very small amount of memory available for page tables to
begin with and need to directly map more in order to obtain further page
tables. At first we have up to 2 MiB of address space available for mapping.

The `round_up()` helper function simply finds the next value aligned to
`step_size` so the range can vary from a single 4 KiB page to the full step
size.

The [get_new_step_size()][get_new_step_size] function attempts to increase the
step size at each while ensuring that we always have sufficient space available
for page table allocations. This [was changed][step_size_diff] in 2014 to
increase the step size by 1 bit less than the number of bits available at each
page table level, e.g. multiplying by 2^8 (256). We only do this if we have
mapped RAM available that equals or exceeds the step value guaranteeing that we
always have enough memory to proceed.

In order to perform the memory mapping we invoke
[init_range_memory_mapping()][init_range_memory_mapping]:

```c
/*
 * We need to iterate through the E820 memory map and create direct mappings
 * for only E820_TYPE_RAM and E820_KERN_RESERVED regions. We cannot simply
 * create direct mappings for all pfns from [0 to max_low_pfn) and
 * [4GB to max_pfn) because of possible memory holes in high addresses
 * that cannot be marked as UC by fixed/variable range MTRRs.
 * Depending on the alignment of E820 ranges, this may possibly result
 * in using smaller size (i.e. 4K instead of 2M or 1G) page tables.
 *
 * init_mem_mapping() calls init_range_memory_mapping() with big range.
 * That range would have hole in the middle or ends, and only ram parts
 * will be mapped in init_range_memory_mapping().
 */
static unsigned long __init init_range_memory_mapping(
                       unsigned long r_start,
                       unsigned long r_end)
{
    unsigned long start_pfn, end_pfn;
    unsigned long mapped_ram_size = 0;
    int i;

    for_each_mem_pfn_range(i, MAX_NUMNODES, &start_pfn, &end_pfn, NULL) {
        u64 start = clamp_val(PFN_PHYS(start_pfn), r_start, r_end);
        u64 end = clamp_val(PFN_PHYS(end_pfn), r_start, r_end);
        if (start >= end)
            continue;

        /*
         * if it is overlapping with brk pgt, we need to
         * alloc pgt buf from memblock instead.
         */
        can_use_brk_pgt = max(start, (u64)pgt_buf_end<<PAGE_SHIFT) >=
                    min(end, (u64)pgt_buf_top<<PAGE_SHIFT);
        init_memory_mapping(start, end, PAGE_KERNEL);
        mapped_ram_size += end - start;
        can_use_brk_pgt = true;
    }

    return mapped_ram_size;
}
```

This function makes use of the fact that the the early boot [memblock][memblock]
memory manager has mapped memory excluding memory that is either reserved (via
the system's [e820][e820] BIOS memory map information), device memory or
otherwise unavailable. We don't try to map these but rather only that memory
which is available as RAM.

### init_memory_mapping()

[init_memory_mapping()][init_memory_mapping] is the point at which blocks of
memory are actually allocated:


```c
/*
 * Setup the direct mapping of the physical memory at PAGE_OFFSET.
 * This runs before bootmem is initialized and gets pages directly from
 * the physical memory. To access them they are temporarily mapped.
 */
unsigned long __ref init_memory_mapping(unsigned long start,
                    unsigned long end, pgprot_t prot)
{
    struct map_range mr[NR_RANGE_MR];
    unsigned long ret = 0;
    int nr_range, i;

    pr_debug("init_memory_mapping: [mem %#010lx-%#010lx]\n",
            start, end - 1);

    memset(mr, 0, sizeof(mr));
    nr_range = split_mem_range(mr, 0, start, end);

    for (i = 0; i < nr_range; i++)
        ret = kernel_physical_mapping_init(mr[i].start, mr[i].end,
                           mr[i].page_size_mask,
                           prot);

    add_pfn_range_mapped(start >> PAGE_SHIFT, ret >> PAGE_SHIFT);

    return ret >> PAGE_SHIFT;
}
```

This function starts by dividing the input range into ranges based on page
size. x86-64 can support 4 KiB, 2 MiB and 1 GiB page sizes and in order to
minimise memory used for page tables as much as possible we try to use as large
a page size as we can.

The maximum number of ranges we can divide a range into for x86-64
(`NR_RANGE_MR`) is 5. To see why consider mapping an example range:

```
map 0x001ff000 -
    0x80201000:

0x001ff000 - 0x001fffff : 4 KiB page size
0x00200000 - 0x3fffffff : 2 MiB page size
0x40000000 - 0x7fffffff : 1 GiB page size
0x80000000 - 0x801fffff : 2 MiB page size
0x80200000 - 0x80200fff : 4 KiB page size
```

We can only use the larger page sizes for addresses that are virtually aligned
(and given `PAGE_OFFSET` is aligned to at least 1 GiB physically aligned) to
their page size. Therefore we can only map pages above and below the aligned
maximum range with the maximum available page we can be aligned to. At most this
will be 4 KiB -> 2 MiB -> 1 GiB -> 2 MiB -> 4 KiB as show in the example above.

[split_mem_range()][split_mem_range] does the work to divide up the input range
accordingly, dividing the memory into `struct map_range` entries in `mr`:

```c
struct map_range {
    unsigned long start;
    unsigned long end;
    unsigned page_size_mask;
};
```

After this has been determined we jump into
[kernel_physical_mapping_init()][kernel_physical_mapping_init] to perform the
actual mapping which invokes
[__kernel_physical_mapping_init()][__kernel_physical_mapping_init] directly:

```c
static unsigned long __meminit
__kernel_physical_mapping_init(unsigned long paddr_start,
                   unsigned long paddr_end,
                   unsigned long page_size_mask,
                   pgprot_t prot, bool init)
{
    bool pgd_changed = false;
    unsigned long vaddr, vaddr_start, vaddr_end, vaddr_next, paddr_last;

    paddr_last = paddr_end;
    vaddr = (unsigned long)__va(paddr_start);
    vaddr_end = (unsigned long)__va(paddr_end);
    vaddr_start = vaddr;

    for (; vaddr < vaddr_end; vaddr = vaddr_next) {
        pgd_t *pgd = pgd_offset_k(vaddr);
        p4d_t *p4d;

        vaddr_next = (vaddr & PGDIR_MASK) + PGDIR_SIZE;

        if (pgd_val(*pgd)) {
            p4d = (p4d_t *)pgd_page_vaddr(*pgd);
            paddr_last = phys_p4d_init(p4d, __pa(vaddr),
                           __pa(vaddr_end),
                           page_size_mask,
                           prot, init);
            continue;
        }

        p4d = alloc_low_page();
        paddr_last = phys_p4d_init(p4d, __pa(vaddr), __pa(vaddr_end),
                       page_size_mask, prot, init);

        spin_lock(&init_mm.page_table_lock);
        if (pgtable_l5_enabled())
            pgd_populate_init(&init_mm, pgd, p4d, init);
        else
            p4d_populate_init(&init_mm, p4d_offset(pgd, vaddr),
                      (pud_t *) p4d, init);

        spin_unlock(&init_mm.page_table_lock);
        pgd_changed = true;
    }

    if (pgd_changed)
        sync_global_pgds(vaddr_start, vaddr_end - 1);

    return paddr_last;
}
```

This starts at a PGD level and invokes functions to perform the page table
layout at each page table layer below returning the last valid physical address
contained in the mapped range.

If we already have a PGD mapping then we do not need to add one. If we do need
to add one then we invoke [alloc_low_page()][alloc_low_page] to alloc a P4D then
after initiasing it we hold the [init_mm][init_mm] page table lock and update
the PGD.

In both instances [phys_p4d_init()][phys_p4d_init] is invoked to initialise the
P4D (in the 4-level case this simply forwards the call to
[phys_pud_init()][phys_pud_init]).

[sync_global_pgds()][sync_global_pgds] copies PGD data between all copies of
kernel PGDs.

Each of the `phys_pXX_init()` functions follow the same kind of pattern - using
`alloc_low_page()` to allocate the actual page tables before invoking the next
page table level `phys_pXX()_init()` function. Taking
[phys_pud_init()][phys_pud_init] as a representative example:

```c
/*
 * Create PUD level page table mapping for physical addresses. The virtual
 * and physical address do not have to be aligned at this level. KASLR can
 * randomize virtual addresses up to this level.
 * It returns the last physical address mapped.
 */
static unsigned long __meminit
phys_pud_init(pud_t *pud_page, unsigned long paddr, unsigned long paddr_end,
          unsigned long page_size_mask, pgprot_t _prot, bool init)
{
    unsigned long pages = 0, paddr_next;
    unsigned long paddr_last = paddr_end;
    unsigned long vaddr = (unsigned long)__va(paddr);
    int i = pud_index(vaddr);

    for (; i < PTRS_PER_PUD; i++, paddr = paddr_next) {
        pud_t *pud;
        pmd_t *pmd;
        pgprot_t prot = _prot;

        vaddr = (unsigned long)__va(paddr);
        pud = pud_page + pud_index(vaddr);
        paddr_next = (paddr & PUD_MASK) + PUD_SIZE;

        if (paddr >= paddr_end) {
            if (!after_bootmem &&
                !e820__mapped_any(paddr & PUD_MASK, paddr_next,
                         E820_TYPE_RAM) &&
                !e820__mapped_any(paddr & PUD_MASK, paddr_next,
                         E820_TYPE_RESERVED_KERN))
                set_pud_init(pud, __pud(0), init);
            continue;
        }

        if (!pud_none(*pud)) {
            if (!pud_large(*pud)) {
                pmd = pmd_offset(pud, 0);
                paddr_last = phys_pmd_init(pmd, paddr,
                               paddr_end,
                               page_size_mask,
                               prot, init);
                continue;
            }
            /*
             * If we are ok with PG_LEVEL_1G mapping, then we will
             * use the existing mapping.
             *
             * Otherwise, we will split the gbpage mapping but use
             * the same existing protection  bits except for large
             * page, so that we don't violate Intel's TLB
             * Application note (317080) which says, while changing
             * the page sizes, new and old translations should
             * not differ with respect to page frame and
             * attributes.
             */
            if (page_size_mask & (1 << PG_LEVEL_1G)) {
                if (!after_bootmem)
                    pages++;
                paddr_last = paddr_next;
                continue;
            }
            prot = pte_pgprot(pte_clrhuge(*(pte_t *)pud));
        }

        if (page_size_mask & (1<<PG_LEVEL_1G)) {
            pages++;
            spin_lock(&init_mm.page_table_lock);

            prot = __pgprot(pgprot_val(prot) | __PAGE_KERNEL_LARGE);

            set_pte_init((pte_t *)pud,
                     pfn_pte((paddr & PUD_MASK) >> PAGE_SHIFT,
                         prot),
                     init);
            spin_unlock(&init_mm.page_table_lock);
            paddr_last = paddr_next;
            continue;
        }

        pmd = alloc_low_page();
        paddr_last = phys_pmd_init(pmd, paddr, paddr_end,
                       page_size_mask, prot, init);

        spin_lock(&init_mm.page_table_lock);
        pud_populate_init(&init_mm, pud, pmd, init);
        spin_unlock(&init_mm.page_table_lock);
    }

    update_page_count(PG_LEVEL_1G, pages);

    return paddr_last;
}
```

A couple of interesting points to note - we investigate the reported physical
memory layout (via [e820][e820]) since obviously `after_bootmem` is set and if
the memory is neither reported as RAM (e.g. it's memory-mapped device registers)
or is reserved then we clear the entry and abort. This is checked at every level
*note that when this is ultimately called from
[init_range_memory_mapping()][init_range_memory_mapping] we should already be
skipping these areas).

Secondly, we check to see whether this page is a 1 GiB (if the mapping already
exists and we don't want that, we reduce the page size) and skip out remaining
page levels if so.

It's important to note that as we perform these mappings we are updating the
current live page mappings meaning the direct mappings become ever more
available as we progress.

Finally, the [init_memory_mapping()][init_memory_mapping] function calls
[add_pfn_range_mapped()][add_pfn_range_mapped] to update PFN mapping data in
global state.

### alloc_low_page

The fundamental means of allocating page tables for our direct memory mapping is
[alloc_low_page()][alloc_low_page] which invokes
[alloc_low_pages()][alloc_low_pages] with num = 1:

```c
/*
 * Pages returned are already directly mapped.
 *
 * Changing that is likely to break Xen, see commit:
 *
 *    279b706 x86,xen: introduce x86_init.mapping.pagetable_reserve
 *
 * for detailed information.
 */
__ref void *alloc_low_pages(unsigned int num)
{
    unsigned long pfn;
    int i;

    if (after_bootmem) {
        unsigned int order;

        order = get_order((unsigned long)num << PAGE_SHIFT);
        return (void *)__get_free_pages(GFP_ATOMIC | __GFP_ZERO, order);
    }

    if ((pgt_buf_end + num) > pgt_buf_top || !can_use_brk_pgt) {
        unsigned long ret = 0;

        if (min_pfn_mapped < max_pfn_mapped) {
            ret = memblock_find_in_range(
                    min_pfn_mapped << PAGE_SHIFT,
                    max_pfn_mapped << PAGE_SHIFT,
                    PAGE_SIZE * num , PAGE_SIZE);
        }
        if (ret)
            memblock_reserve(ret, PAGE_SIZE * num);
        else if (can_use_brk_pgt)
            ret = __pa(extend_brk(PAGE_SIZE * num, PAGE_SIZE));

        if (!ret)
            panic("alloc_low_pages: can not alloc memory");

        pfn = ret >> PAGE_SHIFT;
    } else {
        pfn = pgt_buf_end;
        pgt_buf_end += num;
    }

    for (i = 0; i < num; i++) {
        void *adr;

        adr = __va((pfn + i) << PAGE_SHIFT);
        clear_page(adr);
    }

    return __va(pfn << PAGE_SHIFT);
}
```

Throughout the direct memory mapping we update the global variables
`min_pfn_mapped` and `max_pfn_mapped` via
[add_pfn_range_mapped()][add_pfn_range_mapped] which is invoked by
[init_memory_mapping()][init_memory_mapping] which is called for every direct
mapping we add.

These keep track of the range of physical memory pages mapped and thus memory
potentially available for page table allocations. In this function it allows us
to determine what range of memory is potentially available for page table
allocation so we can use the direct memory mapping to access this memory.

There is a bootstrapping problem with this however - what can we do when need to
allocate page tables in order to get access to the memory we want to put page
tables in? The solution is the `pgt_buf` - a block of memory allocated in the
BSS section of the kernel ELF image (i.e. right at the end) which is limited to
`__brk_limit` and extended by [extend_brk()][extend_brk].

During the direct memory mapping process the whole kernel image is of course by
necessity mapped which includes this region, meaning we can bootstrap page table
allocations before we have direct memory mappings above the kernel to allocate
there.

The `pgt_buf_end` variable tracks the current end of used `pgt_buf` memory,
while `pgt_buf_top` indicates the available capacity. These values are expressed
in pages.

We will have 6 or 12 pages available in the `pgt_buf` depending on whether KASLR
is enabled, providing space for the ISA mapping and the first `PMD_SIZE` mapping
(PUD, PMD, PTE tables for 4-level) providing 2 MiB of address space or 512 4 KiB
page tables.

Even if these are exceeded, we attempt to invoke [extend_brk()][extend_brk] to
get more memory.

If we need to allocate memory from our now-mapped ranges we use
[memblock][memblock] which will make available memory that is marked as free and
available to the system. This will be what is used for the majority of the page
table allocations.

Later, memblock allocations get put into the buddy allocator as part of
transitioning from the early boot memory stage so each allocated page like this
will be properly accounted for.

[0]:https://github.com/torvalds/linux/blob/0fa8ee0d9ab95c9350b8b84574824d9a384a9f7d/arch/x86/include/asm/pgtable_64_types.h#L14-L19
[1]:https://github.com/torvalds/linux/blob/0fa8ee0d9ab95c9350b8b84574824d9a384a9f7d/arch/x86/include/asm/pgtable_types.h#L285
[2]:https://github.com/torvalds/linux/blob/0fa8ee0d9ab95c9350b8b84574824d9a384a9f7d/arch/x86/include/asm/pgtable_types.h#L332
[3]:https://github.com/torvalds/linux/blob/0fa8ee0d9ab95c9350b8b84574824d9a384a9f7d/arch/x86/include/asm/pgtable_types.h#L358
[4]:https://github.com/torvalds/linux/blob/0fa8ee0d9ab95c9350b8b84574824d9a384a9f7d/arch/x86/include/asm/pgtable_types.h#L384
[5]:https://github.com/torvalds/linux/blob/0fa8ee0d9ab95c9350b8b84574824d9a384a9f7d/arch/x86/include/asm/pgtable_64_types.h#L21
[6]:https://github.com/torvalds/linux/blob/0fa8ee0d9ab95c9350b8b84574824d9a384a9f7d/arch/x86/include/asm/pgtable_64_types.h#L56
[7]:https://github.com/torvalds/linux/blob/0fa8ee0d9ab95c9350b8b84574824d9a384a9f7d/arch/x86/include/asm/pgtable_64_types.h#L63
[8]:https://github.com/torvalds/linux/blob/c2e7554e1b85935d962127efa3c2a76483b0b3b6/arch/x86/kernel/head64.c#L56
[9]:https://github.com/torvalds/linux/blob/c2e7554e1b85935d962127efa3c2a76483b0b3b6/arch/x86/kernel/head64.c#L105-L122
[10]:https://github.com/torvalds/linux/blob/0fa8ee0d9ab95c9350b8b84574824d9a384a9f7d/arch/x86/include/asm/pgtable_64_types.h#8L4
[11]:https://github.com/torvalds/linux/blob/0fa8ee0d9ab95c9350b8b84574824d9a384a9f7d/arch/x86/include/asm/pgtable_64_types.h#L91
[12]:https://github.com/torvalds/linux/blob/0fa8ee0d9ab95c9350b8b84574824d9a384a9f7d/arch/x86/include/asm/pgtable_64_types.h#L96
[13]:https://github.com/torvalds/linux/blob/0fa8ee0d9ab95c9350b8b84574824d9a384a9f7d/arch/x86/include/asm/pgtable_64_types.h#L55
[14]:https://github.com/torvalds/linux/blob/0fa8ee0d9ab95c9350b8b84574824d9a384a9f7d/arch/x86/include/asm/pgtable_64_types.h#L61
[15]:https://github.com/torvalds/linux/blob/0fa8ee0d9ab95c9350b8b84574824d9a384a9f7d/arch/x86/include/asm/pgtable_64_types.h#L90
[16]:https://github.com/torvalds/linux/blob/c2e7554e1b85935d962127efa3c2a76483b0b3b6/arch/x86/include/asm/page_types.h#L10
[17]:https://github.com/torvalds/linux/blob/2dcd0af568b0cf583645c8a317dd12e344b1c72a/arch/x86/include/asm/pgtable_types.h
[18]:https://github.com/torvalds/linux/blob/c2e7554e1b85935d962127efa3c2a76483b0b3b6/arch/x86/include/asm/pgtable.h
[19]:https://github.com/torvalds/linux/blob/c2e7554e1b85935d962127efa3c2a76483b0b3b6/arch/x86/mm/hugetlbpage.c#L71
[20]:https://github.com/torvalds/linux/blob/c2e7554e1b85935d962127efa3c2a76483b0b3b6/arch/x86/mm/hugetlbpage.c#L65
[21]:https://github.com/torvalds/linux/blob/c2e7554e1b85935d962127efa3c2a76483b0b3b6/arch/x86/include/asm/pgtable_types.h#L283
[22]:https://github.com/torvalds/linux/blob/master/Documentation/x86/x86_64/mm.rst
[23]:https://github.com/torvalds/linux/blob/3494d58865ad4a47611dbb427b214cc5227fa5eb/arch/x86/mm/kaslr.c
[24]:https://github.com/torvalds/linux/blob/0fa8ee0d9ab95c9350b8b84574824d9a384a9f7d/arch/x86/include/asm/page_types.h#L20
[25]:https://github.com/torvalds/linux/blob/0fa8ee0d9ab95c9350b8b84574824d9a384a9f7d/arch/x86/include/asm/page_64_types.h#L57
[26]:https://github.com/torvalds/linux/blob/fa02fcd94b0c8dff6cc65714510cf25ad194b90d/arch/x86/include/asm/page.h#L59
[27]:https://github.com/torvalds/linux/blob/fa02fcd94b0c8dff6cc65714510cf25ad194b90d/arch/x86/include/asm/io.h#L148
[28]:https://github.com/torvalds/linux/blob/fa02fcd94b0c8dff6cc65714510cf25ad194b90d/arch/x86/include/asm/page.h#L59
[29]:https://github.com/torvalds/linux/blob/fa02fcd94b0c8dff6cc65714510cf25ad194b90d/arch/x86/include/asm/page.h#L42
[30]:https://github.com/torvalds/linux/blob/fa02fcd94b0c8dff6cc65714510cf25ad194b90d/arch/x86/include/asm/io.h#L129
[31]:https://github.com/torvalds/linux/blob/fa02fcd94b0c8dff6cc65714510cf25ad194b90d/arch/x86/include/asm/page_64.h#L32
[32]:https://github.com/torvalds/linux/blob/fa02fcd94b0c8dff6cc65714510cf25ad194b90d/arch/x86/include/asm/page_64.h#L18
[33]:https://github.com/torvalds/linux/blob/fa02fcd94b0c8dff6cc65714510cf25ad194b90d/arch/x86/kernel/head_64.S#L588-L589
[34]:https://github.com/torvalds/linux/blob/fa02fcd94b0c8dff6cc65714510cf25ad194b90d/arch/x86/kernel/head64.c#L161
[35]:https://github.com/torvalds/linux/blob/fa02fcd94b0c8dff6cc65714510cf25ad194b90d/arch/x86/kernel/head64.c#L278
[36]:https://cateee.net/lkddb/web-lkddb/PHYSICAL_START.html
[37]:https://github.com/torvalds/linux/blob/0adb32858b0bddf4ada5f364a84ed60b196dbcda/arch/x86/boot/compressed/kaslr.c#L713
[38]:https://github.com/torvalds/linux/blob/0adb32858b0bddf4ada5f364a84ed60b196dbcda/arch/x86/boot/compressed/misc.c#L400
[39]:https://github.com/torvalds/linux/blob/0adb32858b0bddf4ada5f364a84ed60b196dbcda/arch/x86/boot/compressed/misc.c#L348
[40]:https://github.com/torvalds/linux/blob/0adb32858b0bddf4ada5f364a84ed60b196dbcda/arch/x86/kernel/head64.c#L49
[41]:https://github.com/torvalds/linux/blob/0adb32858b0bddf4ada5f364a84ed60b196dbcda/arch/x86/boot/compressed/misc.c#L211-L212

[start_kernel]:https://github.com/torvalds/linux/blob/3bb61aa61828499a7d0f5e560051625fd02ae7e4/init/main.c#L848
[setup_arch]:https://github.com/torvalds/linux/blob/3bb61aa61828499a7d0f5e560051625fd02ae7e4/arch/x86/kernel/setup.c#L771
[init_mem_mapping]:https://github.com/torvalds/linux/blob/3bb61aa61828499a7d0f5e560051625fd02ae7e4/arch/x86/mm/init.c#L706
[max_pfn]:https://github.com/torvalds/linux/blob/3bb61aa61828499a7d0f5e560051625fd02ae7e4/arch/x86/include/asm/page_64.h#L11
[swapper_pg_dir]:https://github.com/torvalds/linux/blob/3bb61aa61828499a7d0f5e560051625fd02ae7e4/arch/x86/include/asm/pgtable_64.h#L29
[__pa_symbol]:https://github.com/torvalds/linux/blob/3bb61aa61828499a7d0f5e560051625fd02ae7e4/arch/x86/include/asm/page.h#L55
[__phys_addr_symbol]:https://github.com/torvalds/linux/blob/3bb61aa61828499a7d0f5e560051625fd02ae7e4/arch/x86/include/asm/page_64.h#L33-L34
[init_memory_mapping]:https://github.com/torvalds/linux/blob/3bb61aa61828499a7d0f5e560051625fd02ae7e4/arch/x86/mm/init.c#L503
[memory_map_top_down]:https://github.com/torvalds/linux/blob/7f376f1917d7461e05b648983e8d2aea9d0712b2/arch/x86/mm/init.c#L596
[memory_map_bottom_up]:https://github.com/torvalds/linux/blob/7f376f1917d7461e05b648983e8d2aea9d0712b2/arch/x86/mm/init.c#L650
 [split_mem_range]:https://github.com/torvalds/linux/blob/7b1b868e1d9156484ccce9bf11122c053de82617/arch/x86/mm/init.c#L370
[kernel_physical_mapping_init]:https://github.com/torvalds/linux/blob/6bff9bb8a292668e7da3e740394b061e5201f683/arch/x86/mm/init_64.c#L782
[__kernel_physical_mapping_init]:https://github.com/torvalds/linux/blob/6bff9bb8a292668e7da3e740394b061e5201f683/arch/x86/mm/init_64.c#L725
[init_mm]:https://github.com/torvalds/linux/blob/6bff9bb8a292668e7da3e740394b061e5201f683/mm/init-mm.c#L29
[phys_p4d_init]:https://github.com/torvalds/linux/blob/6bff9bb8a292668e7da3e740394b061e5201f683/arch/x86/mm/init_64.c#L674
[phys_pud_init]:https://github.com/torvalds/linux/blob/6bff9bb8a292668e7da3e740394b061e5201f683/arch/x86/mm/init_64.c#L587
[sync_global_pgds]:https://github.com/torvalds/linux/blob/6bff9bb8a292668e7da3e740394b061e5201f683/arch/x86/mm/init_64.c#L212
[alloc_low_page]:https://github.com/torvalds/linux/blob/6bff9bb8a292668e7da3e740394b061e5201f683/arch/x86/mm/mm_internal.h#L6
[alloc_low_pages]:https://github.com/torvalds/linux/blob/7b1b868e1d9156484ccce9bf11122c053de82617/arch/x86/mm/init.c#L114
[add_pfn_range_mapped]:https://github.com/torvalds/linux/blob/7b1b868e1d9156484ccce9bf11122c053de82617/arch/x86/mm/init.c#L473
[early_alloc_pgt_buf]:https://github.com/torvalds/linux/blob/6bff9bb8a292668e7da3e740394b061e5201f683/arch/x86/mm/init.c#L172
[extend_brk]:https://github.com/torvalds/linux/blob/6bff9bb8a292668e7da3e740394b061e5201f683/arch/x86/kernel/setup.c#L198
[get_new_step_size]:https://github.com/torvalds/linux/blob/7b1b868e1d9156484ccce9bf11122c053de82617/arch/x86/mm/init.c#L567
[init_range_memory_mapping]:https://github.com/torvalds/linux/blob/7b1b868e1d9156484ccce9bf11122c053de82617/arch/x86/mm/init.c#L539

[tlb]:https://en.wikipedia.org/wiki/Translation_lookaside_buffer
[trampoline]:https://en.wikipedia.org/wiki/Trampoline_(computing)
[real-mode]:https://en.wikipedia.org/wiki/Real_mode
[memblock]:https://0xax.gitbooks.io/linux-insides/content/MM/linux-mm-1.html
[page-table]:https://en.wikipedia.org/wiki/Page_table
[pti]:https://en.wikipedia.org/wiki/Kernel_page-table_isolation
[pti_kernel_docs]:https://www.kernel.org/doc/html/latest/x86/pti.html
[e820]:https://en.wikipedia.org/wiki/E820
[step_size_diff]:https://github.com/torvalds/linux/commit/132978b94e66f8ad7d20790f8332f0e9c1426029

[ref0]:https://en.wikipedia.org/wiki/Intel_5-level_paging
[ref1]:https://0xax.gitbooks.io/linux-insides/content/
[ref2]:https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-1.html
