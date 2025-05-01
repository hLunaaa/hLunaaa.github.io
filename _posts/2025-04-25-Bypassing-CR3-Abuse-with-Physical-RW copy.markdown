---
layout: post
title:  "Bypassing CR3 Abuse with Physical R/W"
date:   2025-04-25 09:12:29 +0200
---

<br>
### Rationale

Some kernel mode anti-cheat solutions have now begun to obscure `KPROCESS.DirectoryTableBase` by effectively moving the address space via page table manipulation on startup. This prevents attaching to the real process space and makes it slightly more difficult for cheats and malware to to read/write the protected memory.

The problem? User processes were realistically already not able to interact with the protected memory, and kernel drivers and hypervisors have access to the underlying physical memory. Currently `EAC` and `VKG` implement some form of `CR3` abuse, but using monkey bruteforce™ we can walk all physical ranges and parse the `PML4`.

In theory, this can also be achieved from user mode as page tables are mapped to virtual memory with some assistance. We can even force a `TLB` flush using `SwitchToThread()`. The method relies on finding the self-referencing `PML4` entry, so the caller must have some level of access to the underlying physical memory, or leak non-exported names like `MmPteBase`, `MmPdeBase` and `MmPfnDatabase` which are used internally. More closely studying `MmGetVirtualForPhysical()` is recommended.

<br>
### Technique to Find Directory Table

We first cache the physical memory ranges. Here we can either call `MmGetPhysicalMemoryRanges()` or implement the same behavior ourselves. The [ReactOS](https://doxygen.reactos.org/d1/d6d/dynamic_8c.html#a4d2191536acfdbcab710579f81193527) source can help provide context, but obviously look at the disassembly of `MiReferencePageRuns()` as well.

When walking the pages, we assume the page is the correct `PML4` and attempt a translation using a known user address like `EPROCESS.SectionBaseAddress`.

{% highlight C++ %}
#define _pfn_to_page(pfn) (pfn << _page_shift)
#define _page_to_pfn(pfn) (pfn >> _page_shift)

u64 mm_find_process_cr3(EPROCESS* proc)
{
  auto physical_range = MmGetPhysicalMemoryRanges(); // free with ExFreePool
  if (!physical_range)
    return 0;

  // for every physical range
  // for every page
  // for every page table entry -- self-ref can have any idx  

  virt_addr_t mz_base;
  mz_base.val = (u64)proc->SectionBaseAddress;

  do {
    u64 src = (u64)(physical_range->BaseAddress);
    u64 dst = (u64)(physical_range->BaseAddress) + (u64)(physical_range->NumberOfBytes);

    for (u64 it = src; it < dst; it += PAGE_SIZE) 
    for (u32 pml4e_idx = 0; pml4e_idx < 512; pml4e_idx++)
    {
      auto self_ref = read<pml4e_t>(it + pml4e_idx * sizeof(pml4e_t));
      if (it != _pfn_to_page(self_ref.pfn)) // self-ref?
        continue;

      // ok - attempt translation
      auto pml4e = read<pml4e_t>(it + mz_base.pml4_idx * sizeof(pml4e_t));
      if (!pml4e.present)
        break; // bad cr3, check next page...

      auto pml3e = read<pml3e_t>(_pfn_to_page(pml4e.pfn) + mz_base.pml3_idx * sizeof(pml3e_t));
      if (!pml3e.present)
        break;

      auto pml2e = read<pml2e_t>(_pfn_to_page(pml3e.pfn) + mz_base.pml2_idx * sizeof(pml2e_t));
      if (!pml2e.present)
        break;

      auto pml1e = read<pml1e_t>(_pfn_to_page(pml2e.pfn) + mz_base.pml1_idx * sizeof(pml1e_t));
      if (!pml1e.present)
        break;
      
      if (read<u16>(_pfn_to_page(pml1e.pfn) + mz_base.offset) != 'ZM') // check PE magic
        break;
      
      return _page_to_pfn(it); // found cr3!
    }
    physical_range++;
  } while (physical_range->BaseAddress);
}
{% endhighlight %}

<br>
### The Result

There's a lot more you can do in the way of filtering out bad pages before running expensive read operations, but I leave this as an exercise to the reader. In any case, this serves as a good proof of concept. Keep in mind that the latest `Virtualization Based Security` or `VBS` enforces read-only state at the hypervisor level via `Second Level Address Translation` or `SLAT`. However, other values like page frame number remain unprotected. This will be discussed in a separate post in the future.

![Driver Output](/assets/image2.png)