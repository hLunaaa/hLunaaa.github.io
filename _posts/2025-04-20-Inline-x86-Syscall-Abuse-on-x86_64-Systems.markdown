---
layout: post
title:  "Inline x86 Syscall Abuse on x86_64 Systems"
date:   2025-04-20 05:12:41 +0200
---

<br>
### Rationale

Malware and cheats often call system API to do things. Calls like `VirtualAlloc` and `VirtualProtect` are routed through `NtAllocateVirtualMemory` and `NtProtectVirtualMemory` after some user mode checks in `Kernel32.dll`. However, using the highest level API can fall victim to user mode hooks.

For example, instrumentation callbacks are set using `NtSetInformationProcess`. By using the info class `ProcessInstrumentationCallback` & passing a `PROCESS_INSTRUMENTATION_CALLBACK_INFORMATION` structure containing a callback function, we handle all syscalls coming from all threads. We can also use `TLS` and examine the return address and arguments of all syscalls, storing context as we do so.


<br>
### Briefly on Syscalls
When we perform a `syscall`, we request that the OS do something. On Windows version `NT6+`, these system calls are implemented in `ntoskrnl` and `win32k` in exported tables [`KiServiceTables`](https://hfiref0x.github.io/X86_64/NT6_syscalls.html) and [`W32pServiceTables`](https://hfiref0x.github.io/X86_64/NT6_w32ksyscalls.html) respectively.

We can use syscalls to bridge the user-kernel gap not covered by public APIs, though many opaque routines are mirrored renames of syscalls, often with some error checking and so on. The majority of syscalls are trampolined via `ntdll`. We disassemble any x86 syscall.

![Syscall Disasm](/assets/{07E01347-FB93-43D5-BB8E-3880096AEA4B}.png)

The routine first places the syscall index into `EAX`. It then does something peculiar. Because the x86 call is not native, the syscall is instead routed through a translation layer `Wow64SystemServiceCall`. Instead of containing the expected `SYSENTER` instruction, it jumps into 64-bit mode and issues a `SYSCALL`.

![Wow64 Translation](/assets/{1C4BCA20-5531-4364-B3D0-B45FF6EB2670}.png)

We want to retrieve the address of this function, place the syscall id in `EAX`, dynamically fix the stack and return. Instead of calling the `ntdll` export, we avoid any hooks, filters and callbacks and instead ship the call straight to kernel. We can do this by allocating executable memory, setting up the function and calling it on the go.

{% highlight C++ %}
u64 nt_wow64_transition(u8* nt_base)
{
  // 1. vv vv vv vv
  // BA A0 A0 31 4B       mov     edx, 4B31A0A0h
  // FF D2                call    edx

  // 2.    vv vv vv vv
  // FF 25 24 22 3B 4B    jmp      ds:_Wow64Transition
  return *uti::ida_sig<u64*>("BA ?? ?? ?? ?? FF D2 C2 34 00").ref(1).ref(2);
}

template<u32 id, typename... arg_t>
u32 nt_make_call(arg_t... args)
{
  u8 call_stub[15] = {
    0xb8, 0x00, 0x00, 0x00, 0x00,  // mov eax, ?? ?? ?? ??
    0xba, 0x00, 0x00, 0x00, 0x00,  // mov edx, ?? ?? ?? ??
    0xff, 0xd2,                    // call edx
    0xc2, 0x00, 0x00               // retn ?? ??
  };

  auto mem_alloc = VirtualAlloc(0, sizeof(call_stub), MEM_RESERVE | MEM_COMMIT, PAGE_EXECUTE_READWRITE);
  if (!mem_alloc)
    return STATUS_UNSUCCESSFUL;

  *(u32*)(&call_stub[0x1]) = id;
  *(u32*)(&call_stub[0x6]) = nt_wow64_transition(LoadLibraryA("ntdll.dll")); // get wow64 translation
  *(u16*)(&call_stub[0xd]) = (sizeof(arg_t) + ...);

  for (size_t i = 0; i < sizeof(call_stub); i++) mem_alloc[i] = call_stub[i]; // write stub to RWX alloc

  auto ret_status = (u32(__stdcall*)(arg_t...))(args...);

  if (!VirtualFree(mem_alloc, sizeof(call_stub), MEM_RELEASE | MEM_DECOMMIT))
    return STATUS_UNSUCCESSFUL;

  return ret_status;
}
{% endhighlight %}

<br>
### Conclusion
With this method, we are able to call syscalls on the go without calling the `ntdll` trampoline methods. I leave further analysis of `Wow64Transition` as an exercise to the reader. The bottom line is that we are able to implement syscalling trivially, effectively getting around many of the checks and balances that would otherwise catch us - such as those previously mentioned.

Similar methods are available in 64-bit, but the technique must be somewhat adapted. I leave this, too, as an exercise to the reader. Syscalls are a very interesting mechanic of operating system design.