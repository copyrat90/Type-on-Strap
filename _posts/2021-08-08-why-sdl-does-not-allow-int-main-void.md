---
layout: post
title: Why SDL doesn't allow `int main(void)`
tags: [SDL]
author: copyrat90
last_modified_at: 2021-08-08T16:41:00+09:00
---


Probably the most common mistake SDL beginners do is this:

```cpp
// Error: Fails to link.
#include <stdio.h>
#include <SDL.h>

int main(void)
{
    printf("Hello, SDL!\n");
    return 0;
}
```

... Hang on a second, it's just another `Hello, world!` program with the `SDL.h` header included!\
What could possibly go wrong?

**Surprisingly, this code fails to link,** even if you set up the compiler and the linker correctly.\
The error's gonna say something similar to this:
```
.../lib/libSDL2main.a(SDL_windows_main.o): In function `main_getcmdline':
.../src/main/windows/SDL_windows_main.c:71: undefined reference to `SDL_main'
collect2.exe: error: ld returned 1 exit status
```

**To fix this, you have to put `int main(int argc, char** argv)` instead of `int main(void)`.**
```cpp
// OK
#include <stdio.h>
#include <SDL.h>

int main(int argc, char** argv)
{
    printf("Hello, SDL!\n");
    return 0;
}
```

But why is it like that?\
The C language [allows the `int main(void)` as an entry point](https://en.cppreference.com/w/c/language/main_function), doesn't it?

The thing is, **when you include `SDL.h`, your `main` is not the real entry point anymore.**\
**The real entry point now resides somewhere in the SDL library, and it calls your `main` function.**\
... Wait, what?

## How SDL calls your (pseudo) main function

`SDL.h` includes `SDL_main.h`, and this is where the main related conditional defines take place.\
If you look close to the `SDL_main.h`, you'll find [these weird things](https://github.com/libsdl-org/SDL/blob/791d9d3ff61db369008d8590ee87340d7074f3c6/include/SDL_main.h#L109):

```cpp
#if defined(SDL_MAIN_NEEDED) || defined(SDL_MAIN_AVAILABLE)
#define main    SDL_main
#endif
...
/**
 *  The prototype for the application's main() function
 */
typedef int (*SDL_main_func)(int argc, char *argv[]);
extern SDLMAIN_DECLSPEC int SDL_main(int argc, char *argv[]);
```

`#define main SDL_main` tells your preprocessor to substitute identifier `main` into `SDL_main`.\
By including `SDL.h`, you'll also include `SDL_main.h`, **which turns your `int main(int argc, char** argv)` into `int SDL_main(int argc, char** argv)`.**

So, where's the real entry point then?\
It differs from each platform.\
I'll look into [the lines](https://github.com/libsdl-org/SDL/blob/791d9d3ff61db369008d8590ee87340d7074f3c6/src/main/windows/SDL_windows_main.c#L32) which deals with Windows console application.

```cpp
#if defined(_MSC_VER)
/* The VC++ compiler needs main/wmain defined */
# define console_ansi_main main
# if UNICODE
#  define console_wmain wmain
# endif
#endif
...
/* This is where execution begins [console apps, ansi] */
int console_ansi_main(int argc, char *argv[])
{
    return main_getcmdline();
}

#if UNICODE
/* This is where execution begins [console apps, unicode] */
int console_wmain(int argc, wchar_t *wargv[], wchar_t *wenvp)
{
    return main_getcmdline();
}
#endif
```

There they are, the `console_ansi_main` and `console_wmain`, which will turn into `main` and `wmain` respectively by the preprocessor.\
They are calling `main_getcmdline`,
so let's look at [this one](https://github.com/libsdl-org/SDL/blob/791d9d3ff61db369008d8590ee87340d7074f3c6/src/main/windows/SDL_windows_main.c#L40), too.

```cpp
/* Gets the arguments with GetCommandLine, converts them to argc and argv
   and calls SDL_main */
static int main_getcmdline(void)
{
    ...
    SDL_SetMainReady();

    /* Run the application main() code */
    result = SDL_main(argc, argv);
    ...
    return result;
}
```

There!  the `main_getcmline` calls `SDL_main(int argc, char* argv[])` function declared in `SDL_main.h`, and its implementation is your `int main(int argc, char** argv)` function!\
**This is why you can't implement your `main` as `int main(void)`,** as it's not an implementation of `SDL_main(int argc, char* argv[])`, which is called by the SDL entry point.

But why SDL does this?\
SDL got its own [`SDL_Init` function](https://wiki.libsdl.org/SDL_Init) to initialize SDL, after all.\
If it were only for initializing platform specific stuff, it could have been done in `SDL_Init`, with the help of the [conditional inclusion](https://en.cppreference.com/w/cpp/preprocessor/conditional).\
It seems there's no point to mess with the entry point, doesn't it?

## Why SDL mess with entry point

Actually, **some platforms require you to use an _entry point other than `main` function._**\
A well known example of this is [`WinMain` from the Win32 API](https://docs.microsoft.com/en-us/windows/win32/learnwin32/winmain--the-application-entry-point).\
So, normally you have to detect the platform you are working, and [conditionally include](https://en.cppreference.com/w/cpp/preprocessor/conditional) the necessary entry point.

But thinking that the point of using SDL is to deal with cross-platform easily, this is a complete nonsense.\
So the **SDL wraps up your `main` function as `SDL_main` in its _platform-specific entry points_, effectively allowing you to just write `main` on the application level.**\
(e.g. WinMain is dealt in [here](https://github.com/libsdl-org/SDL/blob/3a4b217d6c26df04f4afe05d5f0eec5edd14afef/src/main/windows/SDL_windows_main.c#L112))

So this approach certainly has a reason.\
**BUT, if you also want to use other libraries that mess with `main`, it is GUARENTEED TO BE BROKEN.**\
So, how can you use those libraries with SDL?

## Tell SDL that you'll handle `main` manually (NOT recommended)

Let's look at these lines in `SDL_main.h` again.
```cpp
#if defined(SDL_MAIN_NEEDED) || defined(SDL_MAIN_AVAILABLE)
#define main    SDL_main
#endif
```

Hey, it's covered with `#if defined(SDL_MAIN_NEEDED) || defined(SDL_MAIN_AVAILABLE)` conditional inclusion!\
Looking through the `SDL_main.h` again, you can find [the other lines that defines `SDL_MAIN_NEEDED` and `SDL_MAIN_AVAILABLE`](https://github.com/libsdl-org/SDL/blob/791d9d3ff61db369008d8590ee87340d7074f3c6/include/SDL_main.h#L33).

```cpp
#ifndef SDL_MAIN_HANDLED
#if defined(__WIN32__)
...
#define SDL_MAIN_AVAILABLE

#elif defined(__WINRT__)
...
#define SDL_MAIN_NEEDED
...
#endif
#endif /* SDL_MAIN_HANDLED */
```

When `SDL_MAIN_HANDLED` is defined, `SDL_MAIN_AVAILABLE` or `SDL_MAIN_NEEDED` gets defined.\
So, **by defining `SDL_MAIN_HANDLED` BEFORE including `SDL_main.h`, you can prevent SDL from messing with the entry point.**

<https://wiki.libsdl.org/SDL_SetMainReady#code_examples>
```cpp
#define SDL_MAIN_HANDLED
#include "SDL.h"

int main(int argc, char *argv[])
{
    SDL_SetMainReady();
    SDL_Init(SDL_INIT_VIDEO);
    ...
    SDL_Quit();

    return 0;
}
```

> In this case, your `main` function is the real entry point.\
> And you have to deal with some platform-specific stuff by yourself.

> Also, you have to call `SDL_SetMainReady` before `SDL_Init` in this case,\
> as [on some operating system, SDL_Init() will fail if SDL_main() has not been defined as the entry point for the program.](https://wiki.libsdl.org/CategoryInit)\
> But unfortunately [this will disable SDL's error handling.](https://stackoverflow.com/a/34079413/12875525)

As a small side effect, you can use `int main(void)` in this case.\
**But defining `SDL_MAIN_HANDLED` is NOT recommended, as it lacks of platform-specific initializations.**\
You should not define it whenever possible, and should just stick to `int main(int argc, char** argv)`.

* * *

#### References not mentioned in the article
1. <https://stackoverflow.com/a/33051499/12875525>

*Last modified at : {{ page.last_modified_at }}*
