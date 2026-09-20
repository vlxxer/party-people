---
layout: post
title: "Hello blog"
date: 2026-09-20
description: "whats up party people"
tags: [c, graphics, linux]
---

The Linux framebuffer is the smallest interesting graphics target still
shipping. There is no context to create, no surface to acquire, and no
extension to query. You open a file, map it, and write bytes.

## Mapping the device

```c
#include <linux/fb.h>
#include <sys/ioctl.h>
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>

int fd = open("/dev/fb0", O_RDWR);

struct fb_var_screeninfo vinfo;
struct fb_fix_screeninfo finfo;

ioctl(fd, FBIOGET_VSCREENINFO, &vinfo);
ioctl(fd, FBIOGET_FSCREENINFO, &finfo);

size_t len = finfo.line_length * vinfo.yres;
uint8_t *fb = mmap(NULL, len, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
```

Two structures matter. `fb_var_screeninfo` describes what can change —
resolution, bit depth, the bit position of each color channel.
`fb_fix_screeninfo` describes what cannot, and its `line_length` field is the
one people get wrong.

## The stride trap

The obvious way to compute a pixel offset is width times bytes per pixel.
This is wrong on most hardware. Drivers pad each row to a convenient
boundary, so the distance between vertically adjacent pixels is
`line_length`, not `xres * bytes_per_pixel`.

```c
static void put_pixel(uint8_t *fb, const struct fb_var_screeninfo *v,
                      const struct fb_fix_screeninfo *f,
                      int x, int y, uint32_t color)
{
    size_t bpp = v->bits_per_pixel / 8;
    size_t off = (size_t)y * f->line_length + (size_t)x * bpp;

    *(uint32_t *)(fb + off) = color;
}
```

Use the width and your image shears diagonally, a little more with each row.
It is a distinctive enough artifact that you learn to recognize it on sight.

## The color is still wrong

Assuming `0xRRGGBB` is also wrong. The channel layout lives in `vinfo`, and
the driver is entitled to any arrangement it likes:

```c
static uint32_t pack(const struct fb_var_screeninfo *v,
                     uint8_t r, uint8_t g, uint8_t b)
{
    return ((uint32_t)r << v->red.offset)
         | ((uint32_t)g << v->green.offset)
         | ((uint32_t)b << v->blue.offset);
}
```

On a typical 32-bit console this yields BGRX rather than the RGBX you
expected, which is why the first thing everyone draws is a blue square they
meant to be red.

## Why bother

Nothing here is useful for shipping software. It is useful for understanding
what every layer above is doing on your behalf: a rectangle of memory, a
stride, and a packing convention. Everything else is scheduling and
synchronization.
