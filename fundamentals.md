# Fundamentals

## Page tables

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
                        fffffe000000000  -> |----------------| ^
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
                                            |  Kernel text   | |
                                            |    mapping     | | KERNEL_IMAGE_SIZE = 1 GiB *
                                            |(starts at PA 0)| |
       MODULES_VADDR *= ffffffffc0000000 -> |----------------| x
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

\* Note that `MODULES_VADDR = __START_KERNEL_map + KERNEL_IMAGE_SIZE` and
`KERNEL_IMAGE_SIZE` varies depending on whether KASLR (via
`CONFIG_RANDOMIZE_BASE`) is enabled - with it enabled it is equal to 1 GiB as
shown in the diagram, otherwise it is equal to 512 MiB and modules get 1.5 GiB
rather than 1 GiB.

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
                        fffffe000000000  -> |----------------| ^
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
                                            |  Kernel text   | |
                                            |    mapping     | | KERNEL_IMAGE_SIZE = 1 GiB *
                                            |(starts at PA 0)| |
       MODULES_VADDR *= ffffffffc0000000 -> |----------------| x
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

[ref0]:https://en.wikipedia.org/wiki/Intel_5-level_paging
