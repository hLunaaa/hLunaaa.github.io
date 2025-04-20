---
layout: post
title:  "Inline x86 Syscall Abuse on x86_64 Systems"
date:   2025-04-20 05:12:41 +0200
---

When we perform a `syscall`, we request that the OS do something. On Windows version `NT6+`, these system calls are implemented in `ntoskrnl` and `win32k` in exported tables [`KiServiceTables`](https://hfiref0x.github.io/X86_64/NT6_syscalls.html) and [`W32pServiceTables`](https://hfiref0x.github.io/X86_64/NT6_w32ksyscalls.html) respectively.

We can use syscalls to bridge the user-kernel gap not covered by public APIs, though many opaque routines are mirrored renames of syscalls, often with some error checking and so on. The majority of syscalls are trampolined via `ntdll`. We disassemble any x86 syscall.

![Syscall Disasm](/assets/{07E01347-FB93-43D5-BB8E-3880096AEA4B}.png)

The routine first places the syscall index into `EAX`. It then does something peculiar. Because the x86 call is not native, the syscall is instead routed through a translation layer `Wow64SystemServiceCall`. Instead of containing the expected `SYSENTER` instruction, it jumps into 64-bit mode and issues a `SYSCALL`.

![Wow64 Translation](/assets/{1C4BCA20-5531-4364-B3D0-B45FF6EB2670}.png)

We want to retrieve the address of this function, place the syscall id in `EAX`, dynamically fix the stack and return. Instead of calling the `ntdll` export, we avoid any hooks, filters and callbacks and instead ship the call straight to kernel. We can do this by allocating executable memory, setting up the function and calling it on the go.

{% highlight C++ %}
__declspec(naked) u64 issue_syscall_stub(u64 wow64)
{
  __asm {
    mov eax, 24
    mov edx, wow64
    call edx
    retn 24
  }
}



{% endhighlight %}