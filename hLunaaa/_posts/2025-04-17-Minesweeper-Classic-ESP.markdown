---
layout: post
title:  "Minesweeper Classic ESP"
date:   2025-04-17 09:19:42 +0200
---

Sex2

![IA32e Paging Overview](/assets/4_level_paging.png)

{% highlight C++ %}
u8* vtop(EPROCESS* proc, va_t va)
{
  // read from proc, but if it is currentproc we can obv __readcr3
  // procs can also obscure the DirectoryTableBase with tricks
  pml4_t pml4 = read<ph>(proc->DirectoryTableBase);
  pml3_t pml3 = read<ph>(pml4[va.pml4_idx].pfn);
  pml2_t pml2 = read<ph>(pml3[va.pml3_idx].pfn);
  pml1_t pml1 = read<ph>(pml2[va.pml2_idx].pfn);

  // we also assume all pages are paged-in (present=1)...
  return (u8*)read<ph>(pml1[va.pml1_idx].pfn + va.offset);
}
{% endhighlight %}

Other image