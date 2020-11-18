# Fundamentals

## Page tables

Each page table entry is defined as an `unsigned long` (declared as
[pxxval_t][0] types) i.e. `uint64_t` values.

Some makeshift type safety is enforced by wrapping each of these types in a
struct via e.g. `typedef struct { pgdval_t pgd; } pgd_t`.

## Page table levels

* PGD [pgd_t][1] - Page Global Directory - [PTRS_PER_PGD][6] (512) entries,
  [PGDIR_SHIFT][13] (48 if 5-level enabled, 39 otherwise) offset into VA.

* P4D [p4d_t][2] - Page 4 Directory - [PTRS_PER_P4D][7] entries (1 or 512) - the
  number ofentries varies depending on whether 5-level enabled via
  `CONFIG_X86_5LEVEL` and whether the hardware supports it, stored in the global
  [ptrs_per_p4d][8] value and determined in [check_la57_support()][9] on
  boot. Defaults to 1 if not enabled (and is not used in non-5-level mode),
  [P4D_SHIFT][14] (39) offset into VA.

* PUD [pud_t][3] - Page Upper Directory - [PTRS_PER_PUD][10] (512) entries,
  [PUD_SHIFT][14] (30) offset into VA.

* PMD [pmd_t][4] - Page Middle Directory - [PTRS_PER_PMD][11] (512) entries
  (skipped if PUD level marked huge), [PMD_SHIFT][15] (21) offset into VA.

* PTE [pte_t][5] - Page Table Entry directory - [PTRS_PER_PTE][12] (512) entries
  (skipped if PUD/PMD level marked huge), [PAGE_SHIFT][16]offset into VA.

## Virtual Address layout

Note that the MSB to the bit immediately after the last used bit must all be
set the same otherwise

### 5-level

```
xxxxxxx                        57 bits
<-----><------------------------------------------------------->
   6         5         4         3         2         1
---|---------|---------|---------|---------|---------|----------
3210987654321098765432109876543210987654321098765432109876543210
       [  PGD  ][  P4D  ][  PUD  ][  PMD  ][  PTE  ][  OFFSET  ]
               |        |        |        |        |
               |        |        |        |        |
               |        |        |        |        |-------------  PAGE_SHIFT=12
               |        |        |        |----------------------   PMD_SHIFT=21
               |        |        |-------------------------------   PUD_SHIFT=30
               |        |----------------------------------------   P4D_SHIFT=39
               |------------------------------------------------- PGDIR_SHIFT=48
```

### 4-level

```
xxxxxxxxxxxxxxxx                    48 bits
<--------------><---------------------------------------------->
   6         5         4         3         2         1
---|---------|---------|---------|---------|---------|----------
3210987654321098765432109876543210987654321098765432109876543210
                [  PGD  ][  PUD  ][  PMD  ][  PTE  ][  OFFSET  ]
                        |        |        |        |
                        |        |        |        |
                        |        |        |        |-------------  PAGE_SHIFT=12
                        |        |        |----------------------   PMD_SHIFT=21
                        |        |-------------------------------   PUD_SHIFT=30
                        |---------------------------------------- PGDIR_SHIFT=39
```


## Available memory

* In 4-level mode there are 512*1*512*512*512 = 68.7bn 4KiB pages = 256TiB of address space.

* In 5-level mode there are 512*512*512*512*512 = 35,184bn 4KiB pages = 128PiB of address space.

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
[10]:https://github.com/torvalds/linux/blob/0fa8ee0d9ab95c9350b8b84574824d9a384a9f7d/arch/x86/include/asm/pgtable_64_types.h#L84
[11]:https://github.com/torvalds/linux/blob/0fa8ee0d9ab95c9350b8b84574824d9a384a9f7d/arch/x86/include/asm/pgtable_64_types.h#L91
[12]:https://github.com/torvalds/linux/blob/0fa8ee0d9ab95c9350b8b84574824d9a384a9f7d/arch/x86/include/asm/pgtable_64_types.h#L96
[13]:https://github.com/torvalds/linux/blob/0fa8ee0d9ab95c9350b8b84574824d9a384a9f7d/arch/x86/include/asm/pgtable_64_types.h#L55
[14]:https://github.com/torvalds/linux/blob/0fa8ee0d9ab95c9350b8b84574824d9a384a9f7d/arch/x86/include/asm/pgtable_64_types.h#L61
[15]:https://github.com/torvalds/linux/blob/0fa8ee0d9ab95c9350b8b84574824d9a384a9f7d/arch/x86/include/asm/pgtable_64_types.h#L90
[16]:https://github.com/torvalds/linux/blob/c2e7554e1b85935d962127efa3c2a76483b0b3b6/arch/x86/include/asm/page_types.h#L10
