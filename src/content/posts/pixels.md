---
title: "drawing pixels in c"
published: 2025-08-2
draft: false
description: 'an unnecessary complex operation...'
tags: ['c', 'graphics']
---

## rant

pretty much every single one of my projects requires drawing something on the screen eventually. and for some reason this has always been a source of great frustration.

there seem to be tons of ways of achieving this nowadays, being a perfectionist and minimalist, i always end up spending all my energy considering every option rather than actually working on the project.

i am pretty young, but i've always enjoyed learning about how machines from the past worked. the simplicity of something like the commodore 64 has always had an appeal on me. getting nearly "intimate" with a chip (the [vic](https://en.wikipedia.org/wiki/MOS_Technology_VIC-II)), learning all its nouances... i think i would have definetely enjoyed the experience. i was born in the wrong era...

...one where drawing a pixel requires setting up the entire gpu swapchain and uploading the data as a texture, and only then rendering the texture on a quad. all this mental overhead and setup always causes me to waste tons of hours.

not to mention the fact you then need to choose all your tools... want to use c? perfect this is decided, but then... which libraries? which apis?

first of all: windowing. why is this so complicated?? [sdl](https://www.libsdl.org) is the obvious choice it seems. i have a few issues with it though. it is just too big... i like my binaries to be tiny and just have functionality for what they actually do. if you decide to use sdl, in my experience, you're either forced to use cmake or precompiled binaries. i hate having dynamic dependencies in my executables, might be weird, but it's just the way it is. even if you trim all the functionality you need and statically link sdl you'll never get a binary under 1MB for a program that simply opens a window!!

then there are are (the few) alternatives. [glfw](https://www.glfw.org) and the [sokol](https://github.com/floooh/sokol) headers. i really like glfw, it is stable and just works on all platforms. i also like sokol but it's still pretty immature and missing features, especially regarding windowing. unlike sdl they don't have a simple rendering api. you are forced to use opengl or another full blown 3d rendering api. for drawing some pixels? this adds absolutely unnecessary overhead, cpu and memory requirements.

i have found a few more "contenders", each with their own set of issues:

- [kit](https://github.com/rxi/kit): cute, made by rxi, has some rendering issues, not really stable, will probably never get updates again and will never work for platforms outside windows.
- [fenster](https://github.com/zserge/fenster): made more as a proof of concept, you can read more about it in its blog post, cannot really be considered reliable or feature complete by any means.
- [minifb](https://github.com/emoon/minifb): probably the closest to what i need, but i don't like the code quality, barely maintained and has an unnecessarily complex build process.

## idea

so... this is the plan: i want a header-only, [stb](https://github.com/nothings/stb)-style library, which can open a window and draw pixels inside, is stable, well-written and has a similar api to glfw.

this will be used as a foundation for most of my future projects and will be improved over time. i'll call it **cri** after an important person in my life, it's my base on top of which i'll build everything else.

## windows

ahh... windows. i hate it so much, but if you want your projects to be somewhat relevant you need to develop for it, so we'll start with it.

to open the window we will use the good old [win32](https://learn.microsoft.com/en-us/windows/win32/api) api. to render graphics on the screen we will use the [gdi](https://learn.microsoft.com/en-gb/windows/win32/gdi/windows-gdi) api. this is the simplest and most low level possible way of rendering to a window in windows. our program could most likely compile on windows nt 3.1 using the borland c++ compiler.

enough chat, let's start with windowing. this is the program we want to make work for now:

```c
#define CRI_IMPLEMENTATION
#include "cri.h"

int main() {
  if (!cri_init()) {
    return 1;
  }

  cri_window* window = cri_create_window(640, 480, "hello cri");
  if (!window) {
    cri_terminate();
    return 1;
  }

  while (!cri_window_should_close(window)) {
    cri_poll_events();
  }

  cri_destroy_window(window);
  cri_terminate();

  return 0;
}
```

if you know the glfw api at all, this should look extremely familiar. it is pretty much 1:1 how you would open a window using it. the ```cri.h``` header is used as a standard stb library.

i'll show you the windows implementation of the api. the init function is pretty straight-forward.

```c
bool cri_platform_init(void) {
  SetProcessDPIAware();

  // register window class
  WNDCLASSEXW wc;
  ZeroMemory(&wc, sizeof(wc));
  wc.cbSize = sizeof(wc);
  wc.style = CS_HREDRAW | CS_VREDRAW | CS_OWNDC;
  wc.lpfnWndProc = window_proc;
  wc.hInstance = GetModuleHandleW(NULL);
  wc.hCursor = LoadCursorW(NULL, IDC_ARROW);
  wc.lpszClassName = L"cri";

  wc.hIcon = LoadImageW(GetModuleHandleW(NULL), L"CRI_ICON", IMAGE_ICON, 0, 0, LR_DEFAULTSIZE | LR_SHARED);
  if (!wc.hIcon) {
    wc.hIcon = LoadImageW(NULL, IDI_APPLICATION, IMAGE_ICON, 0, 0, LR_DEFAULTSIZE | LR_SHARED);
  }

  if (!RegisterClassExW(&wc)) {
    return false;
  }

  // setup timer
  QueryPerformanceFrequency((LARGE_INTEGER*)&state.timer_frequency);
  timeBeginPeriod(1);

  return true;
}
```
