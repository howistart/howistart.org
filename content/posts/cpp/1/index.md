---
title: C++
author: Jeremy Ong
subheading: CTO and cofounder of PlexChat
tags: ["cpp"]
summary: Jeremy is a full stack engineer, currently working on a <a href='http://plexchat.com'>company</a> specializing in cross-platform real-time coordination and communication. He is a programming language and paradigm polyglot, with professional experience ranging from distributed systems and network protocols, to game engines and renderers for next-gen game consoles.
date: 2015-10-08
---

## Introduction

C++ is one of the few languages that can incite as much debate as an editor holy
war. However, it resolutely holds it's position as a state-of-the-art lingua
franca, losing in ubiquity only to it's predecessor C. While it's easy to
bemoan the warts of the language, it's important to understand that the
modernization of C++ since the advent of C++11 (which continues through
refinements introduced in C++14 and upcoming additions to C++17), has truly
changed the game (ref.
[The Beast is Back](https://www.jetbrains.com/cpp-today-oreilly/books/Cplusplus_Today.pdf)).

Historically, C++ has not been kind to interested developers. The awkward mix of
various Make build utilities, opaque compilation toolchains, obscure flags, and
more provide both a immense configurability for nearly any platform and a
absolute swamp of complexity.

My hope with this article is to provide you, the reader, with a decent foothold
from which to begin. You might be writing shared libraries for mobile apps, high
performance simulations, graphics engines for consoles, or even embedded code
for a remote control car. Regardless, while I may not have time or space to delve into
every nook and cranny now, you should feel comfortable using what you learn here as
a starting point. It's worth noting that this is not meant to be a language
tutorial or language introduction even. Think of it more as a whirlwind tour of
a workflow to making several different parts that interact with each other in a
non-trivial fashion. For learning C++, I have a few references listed at the
end. However, here's a rough outline of what we *will* cover:

- The objective: What are we going to get by the end of this article?
- The executable: How are C++ programs compiled and how do they run?
- The build process: How do we set up our project for multiple platforms and
configurations?
- The implementation: How do we implement the project?
- Some refinements: Touching up our implementation and discussing other improvements.
- Testing and publishing: How do we make our library testable and package it?
- Notes on writing code: How do I typically author C++ code?
- The wrap-up: Where do I go from here?

## The objective

Our end goal is a self-contained project containing two parts. A shared library,
and a runtime we can use to test the library (hosted as the project
[RePlex](https://github.com/PlexChat/RePlex)). The function of the library is to
act as a live code-reloader, which is extremely useful for live development (and
potentially a surprise for people that didn't realize C++ could do this). This
way, we'll cover how to make a reusable library across other executables, link
that library to a sample executable, and cover a cool technique which needs to
account for the host platform. Neat, let's get started!

## The executable

First, we need to understand the basic building blocks of a C++
executable (also known as a binary executable or just binary for short). While
this might seem overly pedantic, it actually isn't as complicated as you might
think (if you skip the less interesting parts). Knowledge about these building
blocks will give us a common pool of terminology and concepts we'll use
throughout the rest of the article. If this is something you're already familiar
with, feel free to skip ahead to the next section.

C++ code is usually organized into two types of files, header files and source
files, usually with a `.h` or `.cpp` file suffix respectively. The smallest
useful granularity of compiled code is an *object file*, which is the output when
compiling a single source file. Thus, a set of `N` source files will emit `N`
associated compiled object files. These object files can then be combined into a
library or standalone-executable in a process known as *linking*. In addition to
linking object files, the *linker* can also link other libraries (which were
themselves created by linking object files together). Here's a
flow chart showing some of the possibilities:


     Source Data     Compilation              Linking
     -----------     -----------              -------

    +-------------+              +----------+
    |Source file 1|  ----------> |Obj file 1| -
    +-------------+              +----------+  \
    +-------------+              +----------+   \   +---------------------+
    |Source file 2|  ----------> |Obj file 2| ----> |Library or executable|
    +-------------+              +----------+   /   +---------------------+
                                 +----------+  /
                                 |Static Lib| -
                                 +----------+

Notice that I referred to the external library in the above flow chart as a
*static* library. There are actually two types of libraries, *static* and
*dynamic* (also known as *shared*). Static libraries are bundled into the
executable at compile time as you've just seen. Dynamics libraries are loaded on
demand by the executable at runtime.

As you may have guessed, our reloader is only going to work on dynamic
libraries. Reloading a static library would require a relink of the entire
executable and doing this on the fly would be quite a challenge indeed (left as
a sadistic exercise to the reader).

# The build process

If you haven't guessed already, builds for C++ programs can get complicated
fast. Decisions like link order, what to link, what the linker should produce
have to be made by the programmer. The compilation of the object files is quite
configurable as well. For example, we can specify how much the compiler should
optimize or if we need debug symbols. Couple this with the fact that there are
many compilers, each with their own feature sets and options (which may or may
not be platform specific) and we have a real doozy on our hands. Fortunately,
there are a lot more solutions for managing this today than there were years
ago. The solution I'm going to cover here leverages
[premake5](https://premake.github.io/index.html). Another piece of good news is
that once you have some of the boilerplate in place, it's easy to fork a new
project off of it. There exist other build solutions
which I'll list non-exhaustively at the end of the section, but without further
ado, let's start scaffolding our project.

    RePlex
    |-- premake5.lua
    |-- lib
    |   |-- RePlex.cpp
    |   +-- pub
    |       +-- RePlex.h
    +-- runtime
        +-- Main.cpp

There are countless ways one can organize files within a C++ project.
Personally, I like having public headers separated other files within the same
module and in an easily identifiable folder name (like "pub"). I also ensure
that modules themselves (standalone libraries and executables) are separated.
Occasionally, I introduce more folder nesting within a module to provide even
more structure where it makes sense (e.g. grouping platform specific code, test
harness code, etc). The contents of lib will be used to make our library, and
the contents of runtime will make our executable. The file
`RePlex/lib/pub/RePlex.h` will be the public interface of the library. Note that
we'll add files as we need them throughout this article; this is just a starting
point.

What you're actually curious about though, is that `lua` file. The Premake build
configuration system actually consumes Lua files in order to generate project
files or makefiles appropriate to your target platform. For example, on Windows,
you could invoke `premake5 vs2015` in the `RePlex` directory to generate Visual
Studio 2015 solution files to then edit and compile the code. Alternatively, on
OSX you could generate Xcode project files, or GNU makefiles on Linux. Many more
backends to premake
[exist](https://github.com/premake/premake-core/wiki/Using-Premake) and you can
even
[make your own](https://github.com/premake/premake-core/wiki/Adding-a-New-Action).
The Lua files Premake consumes are declarative in nature, but you can use the
entire Lua runtime at your disposal to do more complex build tasks if you wish
(like preprocessing, or what have you). Here are the contents of `premake5.lua`
file.

```lua
workspace "RePlex"
  configurations {"Debug", "Release"}
  -- Use C++ as the target language for all builds
  language "C++"
  targetdir "bin/%{cfg.buildcfg}"

  -- Get that C++14 goodness
  flags { "C++14" }

  filter "configurations:Debug"
    -- Add the preprocessor definition DEBUG to debug builds
    defines { "DEBUG" }
    -- Ensure symbols are bundled with debug builds
    flags { "Symbols" }

  filter "configurations:Release"
    -- Add the preprocessor definition RELEASE to debug builds
    defines { "RELEASE" }
    -- Turn on compiler optimizations for release builds
    optimize "On"

  -- RePlex Library
  project "RePlex"
    kind "SharedLib"
    -- recursively glob .h and .cpp files in the lib directory
    files { "lib/**.h", "lib/**.cpp" }

  -- RePlex Runtime
  project "RePlexRuntime"
    kind "ConsoleApp"
    -- recursively glob .h and .cpp files in the runtime directory
    files { "runtime/**.h", "runtime/**.cpp" }
    -- link the RePlexLib library at runtime
    links { "RePlex" }
    includedirs { "lib/pub" }
```

I don't want to belabor the mechanics of this in too much detail, but hopefully
the structure makes sense. At the top level, we have a workspace called "RePlex"
which is purely semantic; it has no meaning as far as C++ is concerned but acts
as a container for all our build targets (i.e. libraries and executables). This
is useful for signalling what should be grouped to IDEs like Visual Studio,
Xcode. Premake files are structured with lexical scoping, where keywords like
`workspace` and `project` increase the depth of the scope tree. Thus all the
properties defined underneath line 1 apply to all projects in that workspace.
Here, we define two possible build configurations, the workspace language,
target directory, and configuration specific compiler flags and preprocessor
definitions.

Then, we define two projects for our shared library and console application
respectively. Note that RePlexLib is a dependency of RePlexRuntime;
RePlexRuntime has access to all headers in `lib/pub` and also links RePlexLib.
Note that we could have overridden or specified any of global workspace
properties on a per project basis.

Let's make some stub files too:

```c++
// lib/pub/RePlex.h

#pragma once

class Foo
{
public:
    int GetTheAnswer() const;
private:
    int m_answer = 42;
};
```

```c++
// lib/RePlex.cpp

#include "pub/RePlex.h"

int Foo::GetTheAnswer() const
{
    return m_answer;
}
```

```c++
// runtime/Main.cpp

#include <RePlex.h>
#include <iostream>

int main()
{
    Foo foo;
    std::cout << "The answer is " << foo.GetTheAnswer() << std::endl;
    return 0;
}
```

As a side note, my preference for editing code varies based on the platform I am
using. When developing on Windows, I prefer Visual Studio (which has improved
dramatically since 2010, the version I first learned on). When developing on
OSX, I use Xcode due to its integration with the various simulators for iOS
devices. On Linux, I use Emacs with Vim emulation, or Eclipse when I need a
visual debugger (I've had issues using Eclipse as an editor due to stability,
although this may have been addressed in more recent builds since my last
experience with it in 2014). When writing this article, I opted to use a simple
text editor and the make system, as this is the most ubiquitous build system and
the reader should be able to reproduce all the work with the IDE of their
choosing relatively easily.

At this point, you should install Premake if you haven't already for the
operating system of your choice. With this, you should be able to invoke
`premake5 gmake` (substitute `gmake` with whatever action you like) in the root
directory of the app should create the corresponding Makefiles or project files
depending on what action you choose. Subsequently, building the workspace should
emit `bin/Debug/RePlexRuntime` and `bin/Debug/libRePlex.dylib`. Running
`bin/Debug/RePlexRuntime` should give us the output we expect.

    The answer is 42

Woohoo, progress!

# The implementation

As mentioned before, we want to author a library that will handle
"hot-reloading" a different library on the fly. Let's first imagine what the
interface to this library might look like. Obviously, the calling site needs to
supply the name of the library they want to link. In addition, they need to
specify the *symbols* in the library they wish to use. A symbol is, roughly
speaking, the name given by the compiler to a variable name or function in your
program. "Why can't they just use the name I gave it?" you might ask. The reason
is because of features like function overloading, namespacing, templating, and a
number of other features that make the given visible name insufficient for
unique identification purposes. The symbol is generally used by the linker at
compile time to determine where in memory the data or function exists. For our
purposes, we need to make this association at runtime, but fortunately, an API
for doing this exists in all major operating systems. We're going to focus on
UNIX-based operating systems here. The functions we need are:

- `dlopen`: Given a file name, reads the library from disk into memory
- `dlsym`: Given a symbol, returns the address of that symbol
- `dlerror`: Returns an error message describing the last thing that went wrong
- `dlclose`: Releases a reference to the specified library. If the reference
  count drops to zero, the library is removed from the address space.

Let's do a quick and dirty demo to test the mechanics of these functions. The
structure of the program will look like the following:

    +------+
    |RePlex| Library for performing loading and unloading
    +------+
     |       \  Is a dependency of
     |        \
     |         *---*
     |              \     +-------------+
     |               *--> |RePlexRuntime|
     |                    +-------------+
     | Loads               /
     |                    / Uses symbols in RePlexTest
      \                  /
       \     +----------+
        *--> |RePlexTest| Library that gets reloaded
             +----------+

RePlexRuntime is the executable that will be running in our test. RePlexTest
will be the library that we eventually want to hotload. Replex will be our
library for encapsulating the various functions to interact with the dynamic
library. To accommodate this structure, let's add the test library to our
Premake file.

```lua
project "RePlexTest"
  kind "SharedLib"
  files { "test/**.h", "test/**.cpp", "test/pub/*.h" }
```

Easy enough. We'll also add a header and source file to our test library.

```c++
// test/pub/Test.h

#pragma once

// This line prevents C++ name mangling which would prevent dlsym from retrieving
// the correct symbols.
extern "C"
{
    void foo();

    // The extern keyword here exports a global variable that will be defined in Test.cpp
    extern int bar;
}
```
```c++
// test/Test.cpp
#include "pub/Test.h"
#include <cstdio>

void foo()
{
    printf("Hi\n");
}

int bar = 4;
```

Our RePlex library for now will just have a very thin wrapper to the various
`dl*` functions.

```c++
// lib/pub/RePlex.h

#pragma once

#include <dlfcn.h>

void* Load(const char* filepath);

void* LoadSymbol(void* library, const char* symbol);

void Reload(void* &library, const char* filepath);

void PrintError();
```

```c++
// lib/RePlex.cpp

#include "pub/RePlex.h"

#include <cstdio>

void* Load(const char* filepath)
{
  return dlopen(filepath, RTLD_NOW);
}

void* LoadSymbol(void* library, const char* symbol)
{
  return dlsym(library, symbol);
}

void Reload(void* &library, const char* filepath)
{
  dlclose(library);
  library = dlopen(filepath, RTLD_NOW);
}

void PrintError()
{
  printf("Error: %s\n", dlerror());
}
```
Finally, we'll modify our runtime to use these new functions and test run a dll hot-load.

```c++
#include <RePlex.h>
#include <iostream>

#ifdef DEBUG
const char* g_libPath = "bin/Debug/libRePlexTest.dylib";
#else
const char* g_libPath = "bin/Release/libRePlexTest.dylib";
#endif

void (*foo)();

int main()
{
  void* handle = Load(g_libPath);
  if (handle)
  {
    // Set the memory address of the function foo from the library to our foo.
    foo = reinterpret_cast<void (*)()>(LoadSymbol(handle, "foo"));
    // Call foo
    foo();

    // Read the data from the global variable bar in the test library
    int bar = *reinterpret_cast<int*>(LoadSymbol(handle, "bar"));
    std::cout << "bar == " << bar << std::endl;

    // Wait for input to give us a chance to recompile the library
    std::cout << "Make some changes, recompile, and press enter." << std::flush;
    while(std::cin.get() != '\n') {}

    // Reload the library!
    Reload(handle, g_libPath);

    // We need to refetch the symbol because it's location in memory may have changed
    foo = reinterpret_cast<void (*)()>(LoadSymbol(handle, "foo"));
    foo();

    // Do the same for bar
    bar = *reinterpret_cast<int*>(LoadSymbol(handle, "bar"));
    std::cout << "bar == " << bar << std::endl;
  }
  else
  {
    PrintError();
  }
  return 0;
}
```

Running this on my machine generates output like the following:

    Hi
    bar == 4
    Make some changes, recompile, and press enter.
    Can i haz hot-reloading
    bar == 314159

Note that after the second line of that output, I changed the contents of
Test.cpp and reinvoked `make` to recompile the library. So far so good! Now that
we have some grasp of the strange incantations of this program, we can start to
think about a better way to structure it. One thing worth noting is our use of
`extern "C"`. This has a special meaning in C++ and informs the compiler to not
use name-mangling on the symbols defined in its scope (in our case, the `foo`
function and `bar` global). This makes those symbols callable from C code, and
also allows functions like `dlsym` to locate them using a simple C-string
lookup. Symbol lookup for C++ functions and variables is a bit more complex due
to the various decorators that can be attached to functions and variables. More
importantly, the way in which the compiler assigns names to these decorated
functions and variables is not standardized and can vary from compiler to
compiler.

The existing program has a number of problems. First, we need to manually load
the symbol ourselves. This will certainly become tedious if a module exports a
lot of functions and variables. In addition, the test library where the actual
code resides doesn't actually specify it's own exports which is a little odd.
What we really want is a way to package all exports in a pretty package. This
means that RePlex will need to expose two public interfaces, one for publishing
a hot-loadable library, and one for loading and reloading those hot-loadable
libraries. To do this, we'll make a class called `RePlexModule` in Replex.h. The
intended usage of this module is to be inherited by the test library and
specialized so it can load all the correct symbols and expose a cleaner
interface to the end user of the library. Let's start with just the public
interface:

```c++
// lib/pub/RePlex.h
#pragma once

#include <array>
#include <unordered_map>
#include <string>
#include <dlfcn.h>

template <typename E, size_t NumSymbols>
class RePlexModule
{
public:
  static void LoadLibrary() { GetInstance().Load(); }
  static void ReloadLibrary() { GetInstance().Reload(); }

protected:
  static E& GetInstance()
  {
    static E instance;
    return instance;
  }

  //...
  //... continued later
};
```

The first thing you'll notice are the template parameters attached to our class.
That's right, this is a template class! If you haven't seen them before (or saw
them and didn't understand them), the following example should give you the
basic gist.

```c++
template <typename T>
class Foo
{
public:
    T GetT() { return t; }
    T t;
}
```

`Foo` is a class template (you might have heard the term "template class"
before, but honestly, I don't think that makes any sense; just know that they're
interchangeable and "template class" is a bit more common) and has one template
parameter. We can't make an object of type `Foo` since we don't know what `T`
is. However, later we might *instantiate* the class template like so:

```c++
Foo<double> foo;
```

This makes an instance of the class template `Foo` with `T = double`. The
compiler essentially writes out the code like so:

```c++
class Foo<double>
{
public:
    double GetT() { return t; }
    double t;
}
```

The compiler simply did a substitution of the unqualified type `T` for the type
`double`. If you conceptually think of templates in this way and do mental
substitutions, you'll have a good mental model for what's going on. In addition
to types, template arguments can be countable numbers (like `NumSymbols`).

Going back to `RePlexModule`, the only two public functions
RePlexModule will expose to our runtime is `LoadLibrary` and `ReloadLibrary`
which both depend on `GetInstance`. Notice that these are all static functions
that operate on a singleton. Singletons are often considered an antipattern,
however, in this case we actually want to enforce that only one copy of this
class exists in memory. It really doesn't make sense to have multiple instances
(if we wanted, say, two separate versions of the library to coexist, we would
associate them with entirely different types, not two instances of the same
type). Why doesn't `GetInstance` return a reference to `RePlexModule`? Because
remember, we want our test library to inherit from this class to specialize
behavior. Thus, we expect it to supply the value of the template parameter `E`
as itself. If this is confusing now, don't worry. It will get clarified better
later on. We also need to remember to remove the `RePlex` library as a Premake
project since it now is a header only file that doesn't require standalone
compilation (this includes removing it as a link dependency of the runtime).

Now, let's look at the functions exposed to the test library that will inherit
`RePlexModule`:

```c++
  // Start of RePlexModule declaration above
  // ...
protected:

  // Should return the path to the library on disk
  virtual const char* GetPath() const = 0;

  // Should return a reference to an array of C-strings of size NumSymbols
  // Used when loading or reloading the library to lookup the address of
  // all exported symbols
  virtual std::array<const char*, NumSymbols>& GetSymbolNames() const = 0;

  template <typename Ret, typename... Args>
  Ret Execute(const char* name, Args... args)
  {
    // Lookup the function address
    auto symbol = m_symbols.find(name);
    if (symbol != m_symbols.end())
    {
      // Cast the address to the appropriate function type and call it,
      // forwarding all arguments
      return reinterpret_cast<Ret(*)(Args...)>(symbol->second)(args...);
    }
    else
    {
      throw std::runtime_error(std::string("Function not found: ") + name);
    }
  }

  template <typename T>
  T* GetVar(const char* name)
  {
    auto symbol = m_symbols.find(name);
    if (symbol != m_symbols.end())
    {
      return static_cast<T*>(symbol->second);
    }
    else
    {
      // We didn't find the variable. Return an empty pointer
      return nullptr;
    }
  }

private:
  void Load()
  {
    m_libHandle = dlopen(GetPath(), RTLD_NOW);
    LoadSymbols();
  }

  void Reload()
  {
    dlclose(m_libHandle);
    m_symbols.clear();
    Load();
  }

  void LoadSymbols()
  {
    for (const char* symbol : GetSymbolNames())
    {
      m_symbols[symbol] = dlsym(m_libHandle, symbol);
    }
  }

  void* m_libHandle;
  std::unordered_map<std::string, void*> m_symbols;
};
```

The data members of the class at the very bottom are a pointer to the library
handle after it gets loaded and an associative container mapping symbol names to
their pointers in memory. Working upwards, we have functions that operate very
similarly to our initial toy implementation. `LoadSymbols` iterates over all
elements returned from `GetSymbols` and populates `m_symbols`. `Load` works as
before but also calls `LoadSymbols`. `Reload` also works as before but clears
the contents of `m_symbols` first to ensure there aren't any invalid symbols
lingering around. `Load` and `Reload` are called by the static functions
`LoadLibrary` and `ReloadLibrary` defined above respectively.

Towards the top, we have pure virtual functions we expect implementers of this
class to override: `GetPath` and `GetSymbols`. We'll override these soon, but
first, let's look at the (possibly terrifying) functions `Execute` and `GetVar`.

```c++
template <typename Ret, typename... Args>
Ret Execute(const char* name, Args... args)
{
  auto symbol = m_symbols.find(name);
  if (symbol != m_symbols.end())
  {
    return reinterpret_cast<Ret(*)(Args...)>(symbol->second)(args...);
  }
  else
  {
    throw std::runtime_error(std::string("Function not found: ") + name);
  }
}
```

`Execute` takes a function name and `Args... args` as arguments. Its return type
is `Ret`. The first argument is unlikely to be contentious but the second is
likely unfamiliar to those who haven't touched C++ since the advent of the C++11
standard. The `...` syntax denotes a parameter pack and is useful for specifying
a variadic number of arguments with varying types. For example, if I called this
function like:

```c++
Execute<char, int, float>("stuff", 7, 2.4f);

// Compiler turns this into something like:
// char Execute(const char* name, int arg1, float arg2)
// {
//   auto symbol = m_symbols.find(name);
//   if (symbol != m_symbols.end())
//   {
//     return reinterpret_cast<char(*)(int, float)>(symbol->second)(arg1, arg2);
//   }
//   else
//   {
//     throw std::runtime_error(std::string("Function not found: ") + name);
//   }
// }

```

The compiler would interpret `Ret` as a `char`, `args...` would be expanded
to 7 and 2.4f, and `Args...` would be expanded to an int and float type. This
allows us to invoke `Execute` to first lookup the symbol, call it as a function
with the appropriate arguments, and subsequently return the correct return
value. Neat! We throw an exception if the function isn't found because it's hard
to know what to return in this case.

The function `GetVar` is a little simpler. We simply look up the symbol and cast
it as a pointer to specified template type before returning it.

Now, we're ready to specialize this class for our test library.

```c++
// Test.h
#pragma once

#include <RePlex.h>

extern "C"
{
  void foo();
  extern int bar;
}

std::array<const char*, 2> g_exports = { "foo", "bar" };

class TestModule : public RePlexModule<TestModule, g_exports.size()>
{
public:
  static void Foo()
  {
    // Execute might throw, but we don't bother catching the exception here for brevity
    GetInstance().Execute<void>("foo");
  }

  static int GetBar()
  {
    // decltype is a relatively new operator. decltype(bar) resolves to int
    // Note that this function does not protect against retrieving nullptr
    return *GetInstance().GetVar<decltype(bar)>("bar");
  }

protected:
  virtual const char* GetPath() const override
  {
#ifdef DEBUG
    return "bin/Debug/libRePlexTest.dylib";
#else
    return "bin/Release/libRePlexTest.dylib";
#endif
  }

  virtual std::array<const char*, g_exports.size()>& GetSymbols() const override
  {
    return g_exports;
  }
};
```

In addition to the things we actually want to export from before (`foo` and
`bar`), we make an array of exports of size 2 containing the correct string
names. When we inherit from `RePlexModule`, we are careful to fully qualify all
of its template arguments so the compiler can properly substitute all the
template arguments where they are necessary. Thus, `GetInstance` will return a
reference to a `TestModule` singleton, and `GetSymbolNames` will return an array of
2 strings. We override the methods that were declared pure virtual, `GetPath`,
and `GetSymbolNames` in a straightforward manner. Finally, we provide convenient
static functions `Foo` and `GetBar` for calling `foo` and retrieving `bar`.
Notice that the body of `Foo` contains

```c++
GetInstance().Execute<void>("foo");
```

because `foo` returns void and takes no arguments. We're finally ready to modify
our main program to take advantage of the new `TestModule` class.

```c++
// runtime/Main.cpp

#include <RePlex.h>
#include <Test.h>
#include <iostream>

int main()
{
  TestModule::LoadLibrary();
  TestModule::Foo();
  std::cout << "bar == " << TestModule::GetBar() << std::endl;

  std::cout << "Make some changes, recompile, and press enter." << std::flush;
  while(std::cin.get() != '\n') {}

  TestModule::ReloadLibrary();
  TestModule::Foo();
  std::cout << "bar == " << TestModule::GetBar() << std::endl;
  return 0;
}
```

Much better. Now the user of the test module doesn't need to think about the
specifics regarding symbol names, reloading each symbol, or their types. We can
just look at the public interface of `TestModule` to get it all down pat. If
you're following along in code, remember to add the correct include paths to the
test library and runtime projects in `premake5.lua` so the preprocessor includes
work before compiling.

# Some refinements

What we have now is probably usable as an internal library for the purpose of
iterating on a C++ module that exposes a public interface of variables and
functions. By coupling this with enums, interfaces, structs, and classes in a
public header, we can pretty quickly imagine how we might integrate this small
library into our workflow. There are a number of refinements possibly worth
making to the library which I'll mention briefly in this section.

First, the library as is will only work on Linux and OSX platforms. The way
symbols get mapped and unmapped in memory is operating system dependent, and
Windows exposes its own set of functions for doing so: `LoadLibrary`,
`GetProcAddress`, and `FreeLibrary`. They are analogous to `dlopen`, `dlsym`,
and `dlclose` respectively, and I will leave it as an exercise to the reader to
implement this. There are at least two ways to accomplish this. One way is to
add a Premake [filter](https://github.com/premake/premake-core/wiki/filter) on
the platform name and create preprocessor definitions that can be used to ensure
the correct function is called. Alternatively, you can split the interface to
the operating system in files based on operating system and exclude files that
were meant for different platforms in Premake.

A second problem is one of performance. If we are calling `Execute` or `GetVar`
many times in an inner loop, we have to repeatedly hash the symbol name to do a
lookup for the symbol address. To avoid this, we could cache the result of the
lookup. Even easier though, is to avoid using the map in the first place and
store the symbols in the same order as the symbol names. This might make it
harder to change what symbols get loaded between loads, but it's unlikely that
this would be a useful feature anyways.

```c++
  // Before we stored a map
  // std::unordered_map<std::string, void*> m_symbols;
  //
  // Now we'll use a reference to the array that was passed in
  using SymbolArray = std::array<std::pair<const char*, void*>>;
  SymbolArray& m_symbols;
```

Also, we'll change our `LoadSymbols` function to populate this array:

```c++
  void LoadSymbols()
  {
    for (auto&& symbol : m_symbols)
    {
      symbol.second = dlsym(m_libHandle, symbol.first);
    }
  }
```

Note the `auto&&` here which shorthand for saying the variable `symbol` can bind
to any type regardless of const-ness or
[value category](http://en.cppreference.com/w/cpp/language/value_category). In
this case, there is only one possibility of `auto&&` and the compiler will treat
it as a `std::pair<const char*, void*>&`.

Next, we modify our `Execute` and `GetVar` functions to take an index instead of
a string name, in addition to adding a constructor which accepts a reference to
the symbol array.

```c++
  RePlexModule(SymbolArray& symbols) : m_symbols(symbols) {}

  template <typename Ret, typename... Args>
  Ret Execute(const int index, Args... args)
  {
    auto symbol = m_symbols.at(index);
    return reinterpret_cast<Ret(*)(Args...)>(symbol.second)(args...);
  }

  template <typename T>
  T* GetVar(const int index)
  {
    auto symbol = m_symbols.at(index);
    return static_cast<T*>(symbol.second);
  }
```

This is now quite a bit simpler than before because we leave it to the method
`std::array::at` to do bounds checking for us. However, is it likely that this
index will be dynamically determined at runtime? Not really. Instead, if we made
the index a function template parameter, we can enforce that the correct address
is retrieved without a bounds check. Doing this for the `GetVar` function for
example:

```c++
  template <typename T, size_t index>
  T* GetVar()
  {
    static_assert(Index >= 0 && Index < NumSymbols, "Out of bounds symbol index");
    auto symbol = m_symbols[index];
    return static_cast<T*>(symbol.second);
  }
```

Doing `m_symbols[index]` doesn't do a bounds check like `m_symbols.at(index)`
does, but we still get the bounds checking via the `static_assert` which means
it's enforced at compile time instead of runtime. Great! There need to be a few
changes to the `TestModule` to accommodate this new interface:

```c++
std::array<std::pair<const char*, void*>, 2> g_exports = {
  std::make_pair("foo", nullptr),
  std::make_pair("bar", nullptr)
};


class TestModule : public RePlexModule<TestModule, g_exports.size()>
{
public:
  static void Foo(int input)
  {
    GetInstance().Execute<0, void>();
  }

  static int GetBar()
  {
    return *GetInstance().GetVar<1, decltype(bar)>();
  }

  TestModule() : RePlexModule(g_exports) {}

  // Rest of class identical except GetSymbolNames was removed
};
```

Now, we can expect that `Foo` and `GetBar` will be very fast since they no
longer require the string based map lookup. At most, they will need an offset
from the array address, but it's likely that the optimizer will even elide that
instruction.

This library is far from complete. It's not thread-safe. It doesn't protect you
from reloading if you're in the middle of executing a function that might do a
symbol lookup shortly after the previous library is unloaded. It requires the
programmer to repeat himself or herself with regard to function return types and
arguments when binding `Execute`. It doesn't handle errors well (library not
found, symbol missing, etc). It's also missing a number of nice features,
like automatic reload if the file changes. The goal of this article wasn't to
provide a perfect implementation, but hopefully convey a since of how software
like this might be written and structured.

One thing that's important to understand, is that the template programming we
are doing here should not define one's programming style. In this case, we are
using generics simply because we are defining a generic interface, which is
where the template really shines. In particular, templates coupled with static
assertions can go a long way in enforcing the type safety and correctness of an
application. It's worth noting that all the templating is not
exposed to the main runtime executable, who has the luxury of an easy-to-use
interface. Indeed, abstracting away common generic behavior can be a good tool
to reduce complexity and code duplication if done correctly.

# Testing and publishing

Making manual changes to the library and recompiling it, followed by a keystroke
to see if things worked visually is a pretty terrible workflow as far as
detecting regressions goes. In this section, we'll make things a little neater
and repeatable. We'll also discuss the mechanics of publishing our code so
others can use it.

For the purposes of this article, we will use
[googletest](https://github.com/google/googletest) which is a batteries-included
test suite which contains the necessities (assertions, test framework) and other
amenities like test report generation and an optional
[frontend](https://github.com/ospector/gtest-gbar). To make our repository
self-contained, let's add `googletest` as a git submodule, and also add a
Premake project for it.

```bash
git submodule add git@github.com:google/googletest.git
```

```lua
// premake5.lua

# ...

  project "GoogleTest"
    kind "StaticLib"
    files { "googletest/googletest/src/gtest-all.cc" }
    includedirs { "googletest/googletest/include", "googletest/googletest" }

  project "RePlexRuntime"
    kind "ConsoleApp"
    files { "runtime/**.h", "runtime/**.cpp" }
    includedirs { "lib/pub", "test/pub", "googletest/googletest/include" }
    links { "GoogleTest" }
```

Now, the Google test framework is bundled in the repository and invoking
`premake5 gmake` will compile it and link it to `RePlexRuntime`. Now to actually
author the tests themselves. Let's first do a simple test to see how this all
works.

```c++
// runtime/Main.cpp

#include <gtest/gtest.h>

TEST(SillyTest, IsFourPositive)
{
  EXPECT_GT(4, 0);
}

TEST(SillyTest, IsFourTimesFourSixteen)
{
  int x = 4;
  EXPECT_EQ(x * x, 16);
}

int main(int argc, char** argv)
{
  // This allows us to call this executable with various command line arguments
  // which get parsed in InitGoogleTest
  ::testing::InitGoogleTest(&argc, argv);
  return RUN_ALL_TESTS();
}
```

Compiling and invoking `RePlexRuntime` generates the following output:

    ./bin/Debug/RePlexRuntime
    [==========] Running 2 tests from 1 test case.
    [----------] Global test environment set-up.
    [----------] 2 tests from SillyTest
    [ RUN      ] SillyTest.IsFourPositive
    [       OK ] SillyTest.IsFourPositive (0 ms)
    [ RUN      ] SillyTest.IsFourTimesFourSixteen
    [       OK ] SillyTest.IsFourTimesFourSixteen (0 ms)
    [----------] 2 tests from SillyTest (0 ms total)

    [----------] Global test environment tear-down
    [==========] 2 tests from 1 test case ran. (0 ms total)
    [  PASSED  ] 2 tests.

Great! To learn the feature set of the Google test framework more completely, I
recommend reading it's documentation, starting with the
[primer](https://github.com/google/googletest/blob/master/googletest/docs/Primer.md).
As an exercise, I recommend recompiling the above with a failing test to see
what happens before continuing. What we need is to provide a way for the tests
to emit code as text, recompile, and continue at runtime. To do this, we'll
author a fixture class.

```c++
// runtime/Main.cpp

#include <RePlex.h>
#include <Test.h>
#include <cstdlib>
#include <fstream>
#include <gtest/gtest.h>

const char* g_Test_v1 =
  "#include \"pub/Test.h\"\n"
  "int bar = 3;\n"
  "int foo(int x)\n"
  "{\n"
  "  return x + 5;\n"
  "}";

const char* g_Test_v2 =
  "#include \"pub/Test.h\"\n"
  "int bar = -2;\n"
  "int foo(int x)\n"
  "{\n"
  "  return x - 5;\n"
  "}";

class RePlexTest : public ::testing::Test
{
public:
  // Called automatically at the start of each test case.
  virtual void SetUp()
  {
    WriteFile("test/Test.cpp", g_Test_v1);
    Compile(1);
    TestModule::LoadLibrary();
  }

  // We'll invoke this function manually in the middle of each test case
  void ChangeAndReload()
  {
    WriteFile("test/Test.cpp", g_Test_v2);
    Compile(2);
    TestModule::ReloadLibrary();
  }

  // Called automatically at the end of each test case.
  virtual void TearDown()
  {
    TestModule::UnloadLibrary();
    WriteFile("test/Test.cpp", g_Test_v1);
    Compile(1);
  }

private:
  void WriteFile(const char* path, const char* text)
  {
    // Open an output filetream, deleting existing contents
    std::ofstream out(path, std::ios_base::trunc | std::ios_base::out);
    out << text;
  }

  void Compile(int version)
  {
    if (version == m_version)
    {
      return;
    }

    m_version = version;
    EXPECT_EQ(std::system("make"), 0);

    // Super unfortunate sleep due to the result of make not being fully flushed
    // by the time the command returns (there are more elegant ways to solve this)
    sleep(1);
  }

  int m_version = 1;
};
```

The fixture class should be fairly self explanatory. There are three primary
methods for setup, teardown, and reloading the library. We keep track of the
currently loaded library in a member variable `m_version` so we avoid
recompiling the library if the one we want is already loaded (note that
`m_version` defaults to 1 at the beginning). We also have two versions of
`Test.cpp` that we will write out and compile at runtime. You'll have to change
the function signatures of `Foo` in `Test.h` so things compile properly. To use
the fixture, we use the `TEST_F` macro instead of the `TEST` macro like so:

```c++
TEST_F(RePlexTest, VariableReload)
{
  EXPECT_EQ(TestModule::GetBar(), 3);
  ChangeAndReload();
  EXPECT_EQ(TestModule::GetBar(), -2);
}

TEST_F(RePlexTest, FunctionReload)
{
  EXPECT_EQ(TestModule::Foo(4), 9);
  ChangeAndReload();
  EXPECT_EQ(TestModule::Foo(4), -1);
}

int main(int argc, char** argv)
{
  ::testing::InitGoogleTest(&argc, argv);
  return RUN_ALL_TESTS();
}
```

Running this will generate a fair bit of output due to the runtime compilation
but we should have all our tests passing. A *bad* thing we did in order to make
this work was the sleep in our `Compile` function. Even though the `system` call
is synchronous, there is a race condition when reading the file from the disk
which is being flushed by the `make` command. The sleep here is unfortunate
because it makes the tests slower, and also makes the test non-deterministic.
The proper way to implement these tests is to install a handler for a file
change notification. The implementation of this will vary based on the platform,
and is left as an exercise to the reader.

At this point, we might decide that the library is good enough for others to
use. The only file a 3rd-party user would need to leverage our library is
`RePlex.h` (in other words, this is a "header-only" library). Thus, distribution
is just a matter of copying `RePlex.h` into the include path of the target
project. If we had needed to export compiled code in the form of a static or
shared library, we have two options. First, we may opt to compile the library
for all the various combinations of OS and architecture (x86/x64/etc) we might
support. This library would then be distributable as a binary file.
Alternatively, we can simply publish the code with the Premake script we
authored and let the end user compile the code themselves and link the result to
their own library or executable. These are the two primary options at the
moment, and sadly, no unified "package manager" has been authored in the C++
community (although the author of this article is very interested in efforts to
do so).


# Notes on writing code

Without getting into editor/IDE battles, I am posting below a survey of the
various tools I use. If your favorite tool isn't listed, it is either because I
haven't tried it, or have reason to use an alternative. This isn't meant to be exhaustive or
definitive. Rather it should be used as a starting point for someone just
starting to experiment with the language.

- Microsoft Visual Studio 2015: The IDE has come quite a long way, and is by and
  large among the most fully featured IDEs in existence. The debugger, watch
  windows, conditional breakpoints, immediate window, peek windows, graphics
  debugger, and performance analyzers are tools I leverage frequently.
- Xcode: For developing on OSX. It has a similar feature-set as VS and compiles
  using the Clang compiler instead (possibly better error messages, stricter
  warnings).
- Eclipse: Open source IDE for use on Linux that I've used (disclaimer: on Linux
  I tend to fall back to text editors and command line debuggers)
- [cppcheck](https://github.com/danmar/cppcheck): A static analysis tool which
can help you catch bugs
- [Spacemacs](https://github.com/syl20bnr/spacemacs): An Emacs distribution with
batteries included that uses Vim modal
editing as its default mode (this is going to make me popular /s).
- [unittest-cpp](https://github.com/unittest-cpp/unittest-cpp): A nice
lightweight framework for authoring unit tests if you don't need something as heavy duty as gtest.
- [googletest](https://github.com/google/googletest): The Google Test framework used in this article.
- [Premake](https://premake.github.io/download.html): Build configuration tool
  leveraged throughout this article (alternatives to consider include
  [bazel](http://bazel.io/), [CMake](https://cmake.org/),
  [FastBuild](http://www.fastbuild.org/docs/home.html), or
  [Scons](http://www.scons.org/)). Meta-make systems like Premake and CMake have
  supplanted simple Makefiles for more complex projects, and I personally favor
  Premake due to its simplicity and modularized architecture (it also helps that
  it isn't built on top of a strange DSL).
- lldb/gdb: Command line debuggers for quick-and-dirty debugging if the code was
compiled with Clang or GCC respectively
- [Clang tools](http://clang.llvm.org/docs/ClangTools.html): A suite of tools
  that should be part of any robust build pipeline including formatters,
  linters, static analysis, and more.
- [cppreference](http://en.cppreference.com/w/): Invaluable online reference for
  the C and C++ language and standard library

There is also an entire gamut of profilers, heap analyzers, leak detectors, and
more which vary based on operating system and task, which I will leave for
perhaps another time. My recommendation regarding tools is, pick an IDE and
learn it very well (seriously!). Your skills will likely translate to other IDEs. It is a
little difficult to recommend using only an editor (unless you have access to a
plethora of editor scripts and features) when programming in C++, primarily
because on-the-fly static analysis can save a lot of time by detecting compile
errors early (e.g. missing symbols, type mismatches, const-correctness
violations, etc). In addition, debugging runtime problems like thread
deadlocking, memory stomping, and the like are a bit tricky without a robust
debugger. Other languages might do away with these types of problems, but they
won't offer as much control either, so it's a two-way street.

# The wrap up

At this point, we have a project that, while not necessarily in its final form,
sufficiently accomplishes what we set out to do. We also made three
inter-dependent modules, with one providing a shared interface to the other two,
and configured it so that the code could be compiled on any platform (with some
tweaking). We optimized and streamlined the code a little bit using some new
language facilities to make the code faster and less error-prone. Finally, we
also used some functionality provided directly by the operating system, and
potentially learned something about code gets compiled, loaded, and executed. If
you would like to compile the code and run it yourself or make modifications,
please visit the project [github page](https://github.com/PlexChat/RePlex).
This was intended to be an educational project, but improvements are welcome!

If you are a newcomer to the language, you will find that C++, being a fully
multiparadigm language, can be unfriendly at first but is ultimately
unopinionated with regards to how you wish to interact with the hardware. With
the advent of C++11 and C++14 (both of which are supported by all major
compilers), there are now better and easier-to-use abstractions than ever
before. Old-timers of the language, stand to improve the style and
understandability of their code without sacrificing power. New users will find
the language much more accessible compared to previous attempts to learn the
language. I recommend the **C++ Primer** (Lippman, Lajoie, Moo), **Effective Modern
C++** (Meyers), and **Programming: Principles and Practice Using C++**
(Stroustroup), all in their latest incarnations for studying the language.
