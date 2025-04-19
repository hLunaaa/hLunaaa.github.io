---
layout: post
title:  "Exploring CI.dll and Bigpool Cache"
date:   2025-04-18 03:12:41 +0200
---

The `CI.dll` integrity module is responsible for authenticating and verifying the reliability of kernel modules. The kernel initializes it by calling `CiInitialize()`, which returns the `g_CiCallbacks` array. This list of callbacks is then used elsewhere in the kernel. For example, `CiValidateImageHeader()` is called when a driver is loaded to verify its signature.

![CiValidateImageHeader Call Stack](/assets/{FD14D6FB-2AEC-4018-9882-8ABCDAAA56CF}.png)

We navigate to `CiValidateImageHeader()` and follow the call into `CiInitializePhase2()`.

![CiInitializePhase2 Call](/assets/{7E18B9B6-F4C0-4701-A08C-B2A6CF97B269}.png)

Here, in `CiInitializePhase2()`, `CI.dll` creates the `g_CiEaCacheLookasideList` with a call to `ExInitializePagedLookasideList()` which has type `PAGED_LOOKASIDE_LIST`. See [documentation.](https://www.vergiliusproject.com/kernels/x64/windows-11/24h2/_PAGED_LOOKASIDE_LIST)

![ExInitializePagedLookasideList Usage](/assets/image.png)

We generate the signature `48 8D 0D ?? ?? ?? ?? C7 44 24` and scan. In the docs, we see field `ListEntry` which we can walk. Interestingly, we also see `PoolType`, `Tag` and `Size`.

{% highlight C++ %}
struct list_entry_t
{
  list_entry_t* f_link;
  list_entry_t* b_link;
  };
{% endhighlight %}

{% highlight C++ %}
#pragma pack(push, 1)
struct ci_lookaside_t
{
  u8 pad0[0x24];
  u32 type;                 // +0x24
  u32 tag;                  // +0x28
  u32 size;                 // +0x2c
  u8 pad1[0x10];
  list_entry_t link;        // +0x40
};
#pragma pack(pop)
{% endhighlight %}

{% highlight C++ %}
void enum_ci_cache_lookaside(u8* ci_base)
{
  ci_lookaside_t* ci_cache_lookaside = uti::find_ida_sig(ci_base, "48 8D 0D ?? ?? ?? ?? C7 44 24").rva(3,7);
  if (!ci_cache_lookaside) // oopsie
    return;

  do {

  } while ()
  // get src
  // walk until dst
  // sex3?
}
{% endhighlight %}