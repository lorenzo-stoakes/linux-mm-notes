# Memory Reclaim

Memory reclaim occurs when physical memory needs to be freed in order to
maintain zone watermarks. It can either occur indirectly via `kswapd`, a kernel
process woken up when zone watermarks drop below the 'low' level and which
brings memory up to the 'high' level, or directly on physical memory allocation
to bring memory up above the 'minimum' level.

If direct reclaim fails, the out of memory (OOM) killer is activated which
attempts to kill the process using the largest amount of resident memory.

## __alloc_pages_slowpath()

After the fast path has been attempted and failed,
[__alloc_pages_slowpath()][__alloc_pages_slowpath] is invoked. This is where
slower operations are tried to allocate the memory including reclaim (though
note direct reclaim can occur in the fast path (via
[get_page_from_freelist()][get_page_from_freelist]) if the GFP flags enable it.

## OOM Killer

The OOM killer is activated when a zone's watermark reaches or drops below its
minimum watermark and reclaim is enabled, or in the case of user page allocation
if the allocation fails (which does not otherwise invoke reclaim).

determines the process using the largest amount of RSS (Resident
Set Size, i.e. actually allocated memory, including shared) and terminates
it. Its behaviour can be moderated by assigning processes different OOM killer
'scores' ranging from disabling the OOM entirely for that process to always
selecting that process - this is typically adjusted via
`/proc/$pid/oom_score_adj` and can range from `-1000` (`OOM_SCORE_ADJ_MIN`) to
`1000` (`OOM_SCORE_ADJ_MAX`).

The core OOM killer is invoked from [out_of_memory()][out_of_memory], with
userland page fault OOM being invoked from
[pagefault_out_of_memory()][pagefault_out_of_memory] which ultimately invokes
`out_of_memory()` too.

[oom_score_adj]:https://github.com/torvalds/linux/blob/master/Documentation/filesystems/proc.rst#31-procpidoom_adj--procpidoom_score_adj--adjust-the-oom-killer-score
[out_of_memory]:https://github.com/torvalds/linux/blob/996e435fd401de35df62ac943ab9402cfe85c430/mm/oom_kill.c#L1049
[pagefault_out_of_memory]:https://github.com/torvalds/linux/blob/996e435fd401de35df62ac943ab9402cfe85c430/mm/oom_kill.c#L1127
[__alloc_pages_slowpath]:https://github.com/torvalds/linux/blob/71c061d2443814de15e177489d5cc00a4a253ef3/mm/page_alloc.c#L4654
[get_page_from_freelist]:https://github.com/torvalds/linux/blob/f6e1ea19649216156576aeafa784e3b4cee45549/mm/page_alloc.c#L3826
