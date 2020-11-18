# Fundamentals

## Page tables

Each page table entry is defined as an `unsigned long` (declared as
[pxxval_t][0] types) i.e. `uint64_t` values.

Some makeshift type safety is enforced by wrapping each of these types in a
struct via e.g. `typedef struct { pgdval_t pgd; } pgd_t`.

## Page table levels

| Level | Type | Count | Shift | Description |
|-------|------|-------|-------|-------------|
| PGD | [pgd_t][1] | [PTRS_PER_PGD][6] (512) | [PGDIR_SHIFT][13] (48 if 5-level enabled, 39 otherwise) | Page Global Directory |
| P4D | [p4d_t][2] | [PTRS_PER_P4D][7] (512 or 1) | [P4D_SHIFT][14] (39) | Page 4 Directory\* |
| PUD | [pud_t][3] | [PTRS_PER_PUD][10] (512) | [PUD_SHIFT][14] (30) | Page Upper Directory \*\* |
| PMD | [pmd_t][4] | [PTRS_PER_PMD][11] (512) | [PMD_SHIFT][15] (21) | Page Middle Directory\*\* |
| PTE | [pte_t][5] | [PTRS_PER_PTE][12] (512) | [PAGE_SHIFT][16] (12) | Page Table Entry directory |

\* Number of entries depends on whether `CONFIG_X86_5LEVEL` is set (it's
enabled) and whether the hardware supports it, stored in the global
[ptrs_per_p4d][8] value and determined in [check_la57_support()][9] on
boot. Defaults to 1 if not enabled (and is not used in non-5-level mode).

\*\* PMD and PTE directory entries are skipped (and the PUD treated as a PTE) if
the PUD is marked huge (1 GiB page size), the PTE directory entry is skipped if
the PMD is marked huge (2 MiB page size).

## Virtual Address layout

Note that the MSB to the bit immediately after the last used bit must all be set
the same otherwise the processor will fault (denoted as 'xxxx...' below).

### 5-level

```
xxxxxxx                        57 bits
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

### 4-level

```
xxxxxxxxxxxxxxxx                    48 bits
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


## Available memory

* In 4-level mode there are `512 * 1 * 512 * 512 * 512` = 68.7bn 4KiB pages = __256 TiB__ of address space.

* In 5-level mode there are `512 * 512 * 512 * 512 * 512` = 35.2tn 4KiB pages = __128 PiB__ of address space.

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
