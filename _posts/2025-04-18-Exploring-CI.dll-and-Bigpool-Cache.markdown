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

As of `Win11`, this struct has guhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhh

{% highlight C++ %}
void loop_draw_esp(u8* game)
{
  u32 hdc = GetDC(0); // lazy, grab the entire screen
  if (!hdc)
    return;

  if(*(u32*)(game + 0x5164) == 0) // should clock start (is playing?)
    return;

  u32 win_x = *(u32*)(game + 0x56B0) + 18;
  u32 win_y = *(u32*)(game + 0x56B4) + 105; // window pos + offset to tile grid

  for (u32 y = 1; y <= *(u32*)(game + 0x5338); y++)
  for (u32 x = 1; x <= *(u32*)(game + 0x5334); x++) // walk coords
  {
    u32 tile = (game + 0x5340)[32 * y + x];
    if (tile & 0x80u) // has bomb?
      draw_tile(hdc, win_x + (x - 1) * 15, win_y + (y - 1) * 15, RGB(255, 0, 0));
  }

  ReleaseDC(0, hdc);
  Sleep(5);
}
{% endhighlight %}