---
layout: post
title:  "Bypassing CR3 Abuse"
date:   2025-04-25 09:12:29 +0200
---

Some kernel mode anti-cheat solutions have now begun to obscure `KPROCESS.DirectoryTableBase` by effectively moving the address space via page table manipulation on startup. This prevents attaching to the real process space and makes it slightly more difficult for cheats and malware to to read/write the protected memory.

The problem? User processes were realistically already not able to interact with the protected memory, and kernel drivers and hypervisors have access to the underlying physical memory. Currently `EAC` and `VKG` implement some form of `CR3` abuse, but using monkey bruteforce™ we can walk all physical ranges and parse the `PML4`.

In theory, this can also be achieved from user mode as page tables are mapped to virtual memory. We can even force a `TLB` flush using `SwitchToThread()`. The method relies on finding the self-referencing `PML4` entry, so the caller must either have access to the underlying physical memory or know the `CR3`.

{% highlight C++ %}
u8* guh(EPROCESS* proc, virt_addr_t va)
{

}
{% endhighlight %}
