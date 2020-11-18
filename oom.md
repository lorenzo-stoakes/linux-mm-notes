# OOM Killer

* Files - `include/linux/oom.h`, `include/linux/mm.h`, `mm/oom_kill.c`

* Entrypoint - `out_of_memory()`, (also invoked via `pagefault_out_of_memory()`)

Example:

```
[   33.932118] stress-ng-brk invoked oom-killer: gfp_mask=0x100dca(GFP_HIGHUSER_MOVABLE|__GFP_ZERO), order=0, oom_score_adj=1000
[   33.932127] CPU: 1 PID: 212 Comm: stress-ng-brk Not tainted 5.10.0-rc3-next-20201116-dirty #4
[   33.932130] Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS ArchLinux 1.14.0-1 04/01/2014
[   33.932132] Call Trace:
[   33.932147]  dump_stack+0x57/0x6a
[   33.932153]  dump_header+0x4c/0x32e
[   33.932160]  ? do_try_to_free_pages+0x263/0x330
[   33.932165]  oom_kill_process.cold+0x4e/0xa9
[   33.932171]  ? find_lock_task_mm+0x4c/0x90
[   33.932176]  out_of_memory+0x192/0x660
[   33.932185]  __alloc_pages_slowpath.constprop.0+0xbef/0xcc0
[   33.932193]  __alloc_pages_nodemask+0x2c3/0x2f0
[   33.932201]  alloc_pages_vma+0x64/0x1a0
[   33.932207]  handle_mm_fault+0x64d/0xe70
[   33.932215]  do_user_addr_fault+0x1ce/0x410
[   33.932223]  exc_page_fault+0x4f/0x110
[   33.932230]  ? asm_exc_page_fault+0x8/0x30
[   33.932234]  asm_exc_page_fault+0x1e/0x30
[   33.932240] RIP: 0033:0x55fde2e8916d
[   33.932251] Code: Unable to access opcode bytes at RIP 0x55fde2e89143.
[   33.932254] RSP: 002b:00007ffd669d9a10 EFLAGS: 00010246
[   33.932259] RAX: 000055fe0d990000 RBX: 000055fe0d990000 RCX: 00007f29ee67fe7b
[   33.932261] RDX: 00007f29ee74f480 RSI: 000055fde2fe2923 RDI: 000055fe0d991000
[   33.932264] RBP: 0000000000001000 R08: 0000000000000000 R09: 00007f29ee582010
[   33.932266] R10: 0000000000000000 R11: 0000000000000206 R12: 000055fde4a91000
[   33.932269] R13: 0000000000000004 R14: 00007ffd669d9b30 R15: 0000000000000000
[   33.932272] Mem-Info:
[   33.932281] active_anon:7174 inactive_anon:484737 isolated_anon:0
                active_file:0 inactive_file:84 isolated_file:0
                unevictable:6 dirty:0 writeback:0
                slab_reclaimable:2894 slab_unreclaimable:2892
                mapped:8 shmem:304 pagetables:4146 bounce:0
                free:3275 free_pcp:356 free_cma:0
[   33.932289] Node 0 active_anon:28696kB inactive_anon:1938948kB active_file:0kB inactive_file:336kB unevictable:24kB isolated(anon):0kB isolated(file):0kB ms
[   33.932291] Node 0 DMA free:7896kB min:44kB low:56kB high:68kB reserved_highatomic:0KB active_anon:0kB inactive_anon:7960kB active_file:0kB inactive_file:0B
[   33.932300] lowmem_reserve[]: 0 1965 1965 1965
[   33.932308] Node 0 DMA32 free:5204kB min:5648kB low:7660kB high:9672kB reserved_highatomic:0KB active_anon:28696kB inactive_anon:1930988kB active_file:0kB B
[   33.932317] lowmem_reserve[]: 0 0 0 0
[   33.932325] Node 0 DMA: 2*4kB (U) 4*8kB (UM) 1*16kB (M) 1*32kB (U) 2*64kB (UM) 2*128kB (UM) 1*256kB (U) 0*512kB 1*1024kB (U) 1*2048kB (M) 1*4096kB (M) = 78B
[   33.932361] Node 0 DMA32: 37*4kB (UE) 18*8kB (ME) 1*16kB (M) 1*32kB (U) 10*64kB (UME) 21*128kB (UM) 2*256kB (E) 0*512kB 1*1024kB (M) 0*2048kB 0*4096kB = 52B
[   33.932395] Node 0 hugepages_total=0 hugepages_free=0 hugepages_surp=0 hugepages_size=1048576kB
[   33.932398] Node 0 hugepages_total=0 hugepages_free=0 hugepages_surp=0 hugepages_size=2048kB
[   33.932400] 832 total pagecache pages
[   33.932403] 440 pages in swap cache
[   33.932405] Swap cache stats: add 357112, delete 356625, find 48/100
[   33.932407] Free swap  = 0kB
[   33.932409] Total swap = 1048572kB
[   33.932411] 524155 pages RAM
[   33.932412] 0 pages HighMem/MovableOnly
[   33.932414] 15903 pages reserved
[   33.932415] 0 pages cma reserved
[   33.932417] Tasks state (memory values in pages):
[   33.932419] [  pid  ]   uid  tgid total_vm      rss pgtables_bytes swapents oom_score_adj name
[   33.932429] [     99]     0    99    19541       62   188416      229          -250 systemd-journal
[   33.932435] [    109]     0   109     7413        1    81920      378         -1000 systemd-udevd
[   33.932439] [    134]    81   134     1820        1    49152      140          -900 dbus-daemon
[   33.932443] [    136]     0   136     4727        0    77824      282             0 systemd-logind
[   33.932447] [    138]   977   138      750        0    45056       88             0 dhcpcd
[   33.932451] [    139]     0   139      824       11    49152       75             0 dhcpcd
[   33.932455] [    140]   977   140      703       10    45056       57             0 dhcpcd
[   33.932458] [    141]   977   141      703       10    45056       57             0 dhcpcd
[   33.932461] [    151]     0   151      618        0    45056       32             0 agetty
[   33.932465] [    152]     0   152     1692        0    49152      172             0 login
[   33.932469] [    156]   977   156      824        9    49152       75             0 dhcpcd
[   33.932473] [    159]  1000   159     5054        0    81920      391             0 systemd
[   33.932477] [    160]  1000   160     6172       10    86016      631             0 (sd-pam)
[   33.932480] [    166]  1000   166     1831        2    61440      487             0 zsh
[   33.932484] [    204]  1000   204     9334       18    69632       67             0 stress-ng
[   33.932488] [    205]  1000   205     9334       23    69632       64             0 stress-ng-brk
[   33.932491] [    206]  1000   206     9334       23    69632       64             0 stress-ng-brk
[   33.932494] [    207]  1000   207     9334       34    69632       55             0 stress-ng-stack
[   33.932498] [    208]  1000   208     9334       23    69632       64             0 stress-ng-stack
[   33.932502] [    209]  1000   209   192035   122496  1531904    60287          1000 stress-ng-brk
[   33.932506] [    211]  1000   211  1322987   189734 10596352   139070          1000 stress-ng-stack
[   33.932510] [    212]  1000   212   177014   108359  1409024    59406          1000 stress-ng-brk
[   33.932513] [    213]  1000   213   291120    70500  2326528       51          1000 stress-ng-stack
[   33.932517] oom-kill:constraint=CONSTRAINT_NONE,nodemask=(null),cpuset=/,mems_allowed=0,task=stress-ng-stack,pid=211,uid=1000
[   33.932534] Out of memory: Killed process 211 (stress-ng-stack) total-vm:5291948kB, anon-rss:758932kB, file-rss:0kB, shmem-rss:4kB, UID:1000 pgtables:103480
[   33.992414] oom_reaper: reaped process 211 (stress-ng-stack), now anon-rss:0kB, file-rss:0kB, shmem-rss:4kB
```
