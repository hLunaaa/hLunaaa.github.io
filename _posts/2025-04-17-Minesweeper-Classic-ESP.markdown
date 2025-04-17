---
layout: post
title:  "Minesweeper Classic ESP"
date:   2025-04-17 09:19:42 +0200
---

Minesweeper is a fun little game about clicking squares and avoiding bombs. But I've always wondered how it works under the hood. A grid game. There's probably a bitmap somewhere. 

In the context menu we can create a custom came by setting width, height and number of bombs. I attach a debugger and see where these are accessed. They bring me to the function responsible for showing the dialog `winmine.hlp`. We rename the variables.

![](/assets/{AB29194B-780E-4E52-A8D0-86B679B54278}.png)

By cross referencing where these are used we find multiple functions. Some store the custom game values to registry keys, others load defaults on startup. We also find an interesting function that loops, reducing the bomb count on each iteration.

![](/assets/{FB211914-8E66-435F-9BAD-01036C06A600}.png)

At this point it is quite clear what this function does. We loop over the bitmap, setting the bomb flag `0x80` at random using `rand`. We also see the bitmap stride `32 * Y + X`, with `X` and `Y` starting at `1`. With this information, we are able to read the bitmap and check for bomb flags.

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

It really is that shrimple. We inject and play. This is the result.

![](/assets/{EAB758A4-2E16-4FE7-929F-726D7CB065AB}.png)
