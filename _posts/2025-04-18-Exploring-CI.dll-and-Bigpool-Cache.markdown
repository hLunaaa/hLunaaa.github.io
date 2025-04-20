---
layout: post
title:  "Exploring CI.dll and Bigpool Cache"
date:   2025-04-18 03:12:41 +0200
---

The `CI.dll` integrity module is responsible for authenticating and verifying the reliability of kernel modules. The kernel initializes it by calling `CiInitialize()`, which returns the `g_CiCallbacks` array. This list of callbacks is then used elsewhere in the kernel. For example, `CiValidateImageHeader()` is called when a driver is loaded to verify its signature.

![CiValidateImageHeader Call Stack](/assets/{FD14D6FB-2AEC-4018-9882-8ABCDAAA56CF}.png)

We navigate to `CiValidateImageHeader()` and follow the call into `CiInitializePhase2()`.

![CiInitializePhase2 Call](/assets/{7E18B9B6-F4C0-4701-A08C-B2A6CF97B269}.png)

Here, in `CiInitializePhase2()`, `CI.dll` creates the `g_CiEaCacheLookasideList` with a call to `ExInitializePagedLookasideList()` which has type `PAGED_LOOKASIDE_LIST`. See [documentation.](https://www.vergiliusproject.com/kernels/x64/windows-11/24h2/_PAGED_LOOKASIDE_LIST) This lookaside list tracks paged big pool (heap) allocations.

Insiders are already aware that there are two (well, three) pool allocators: regular and big allocator. The regular allocator is used for any allocation less than or equal to a page in size, including the 32 byte pool header and initial free block. Big pool allocator is used when the size is more than one page, or when the pool type is `CacheAligned`. Crucially, big pool allocations dont have room for headers, and are instead tracked in the `PoolBigTable`.

![ExInitializePagedLookasideList Usage](/assets/image.png)

In the docs, we see field `ListEntry` which we use to iterate over the list. Interestingly, we also see `PoolType`, `Tag` and `Size`. We generate the signature `48 8D 0D ?? ?? ?? ?? C7 44 24` and walk.

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
  auto ci_cache = uti::ida_sig<ci_lookaside_t*>(ci_base, "48 8D 0D ?? ?? ?? ?? C7 44 24").rva(3,7);
  if (!ci_cache) // oopsie                                             	^^^^^^^^^^^
    return;
  
  for (auto it = ci_cache->link.f; it != &ci_cache->link; it = it->f)
  {
    auto it_entry = (ci_lookaside_t*)((u8*)(it) - 0x40); // link is +0x40
    if (!it_entry)
      continue;

    auto it_tag = (u8*)(&it_entry->tag);
    DbgPrintEx(0, 0, "-- size: %6X type: %4X tag: %c%c%c%c\n",
      it_entry->size, it_entry->type, it_tag[0], it_tag[1], it_tag[2], it_tag[3]);
  }
}
{% endhighlight %}

We load the driver and run it. The type is `POOL_TYPE`, which is [documented.](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/wdm/ne-wdm-_pool_type) The output shows many `PagedPool` entries with varying sizes. `CI.dll` keeps track of a lot of interesting telemetry. There are many lists like `g_BootDriverList`, `g_CiValidationLookasideList`, `g_KernelHashBucketList` and `g_KernelHashBucketList`.

![Output](/assets/{83E0EFE6-C7EE-420A-92C1-590E6A2EE129}.png)

The traditional way of dealing dealing with these lists is locking, deleting and creating them with `ExDeleteLookasideListEx()` and `ExInitializeLookasideListEx()`. But clearing lists which would otherwise include reference to legitimate drivers is suspicious. Indeed, the best approach is to walk every relevant list and unlinking references to your vulnerable driver.

In addition, `ntoskrnl` tracks additional telemetry like `PiDDBCacheTable`, `MmLastUnloadedDriver`, `MmUnloadedDrivers`. The more you dig, the more you uncover. All these lists, caches and trees are potential detection vectors. Advanced malware and cheats should consider all of them and more.