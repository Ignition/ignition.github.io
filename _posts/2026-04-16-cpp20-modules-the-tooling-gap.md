---
title: "C++20 Modules: The Tooling Gap"
date: 2026-04-16 16:38:00 +0100
categories: [C++]
tags: [c++, modules, clangd, tooling]
---

We've been incrementally adopting C++20 modules at [Memgraph](https://memgraph.com/) ([source](https://github.com/memgraph/memgraph)) since late 2025. The compiler side has been surprisingly smooth. The tooling side, less so.

While working on partially modularised code, I hit a series of issues with clangd, ccache, CMake, and clang-tidy. I use CLion day-to-day, where most of these issues either don't surface or are masked enough that they didn't impact my work. But for anyone using VS Code or Neovim with clangd (or any of the other tools), here's what I found and how to work around it.

> This is pieced together from conversations with colleagues, Mastodon posts, and issues/PRs I've filed or engaged with. Some details may be slightly off. If you spot an error, let me know.
{: .prompt-info }

> We're on clang 20 (specifically 20.1.7) until we can upgrade. Some of what I describe may already be fixed in newer versions. I've linked the relevant upstream fixes where I know them.
{: .prompt-warning }

Before diving in, a few prerequisites:

1. **Pin to a reasonable clangd binary**. Older versions don't handle modules well and may crash. If you're on your distro's clangd, try a newer one.
2. **Make sure clangd can find `compile_commands.json`** by adding a `.clangd` file in your project root:

   ```yaml
   CompileFlags:
     CompilationDatabase: build/
   ```
3. **Enable `--experimental-modules-support`**. Without it, clangd's module support is limited.

With those out of the way, here are the module-specific problems.

## Problem 1: clangd doesn't build PCM files

~~clangd doesn't trigger any builds to produce the precompiled module (`.pcm`) files it needs to understand module imports. It expects them to already exist. If they don't, every `import` is an error.~~

~~The workaround is to pre-build the module targets yourself before opening your editor, so clangd has the PCMs ready and doesn't try to build them. If you're using ninja, you can ask it which build outputs depend on module files, then build only those:~~

```bash
ninja -C build -t inputs \
  | grep -E '\.cppm\.o$|\.o\.modmap$' \
  | xargs -r ninja -C build
```

> **Correction:** I got this wrong. Thanks to [not_a_novel_account on Reddit](https://www.reddit.com/r/cpp/comments/1snx6r8/comment/ogq3efj) for pointing out that clangd _does_ attempt to build PCMs when `--experimental-modules-support` is enabled. I recall 100% CPU usage and non-responsiveness, yet I am no longer sure if we fully understood the root cause. It may have been clangd running without `--experimental-modules-support`, or something else entirely. Additional [related fixes](https://github.com/llvm/llvm-project/pull/187654) coming in Clang 23.
{: .prompt-warning }

## Problem 2: Header flag heuristics

`compile_commands.json` only contains entries for translation units (`.cpp` and `.cppm` files). It has no entries for headers. clangd works around this with a heuristic: it finds a nearby translation unit with a similar name and borrows its flags.

This is usually fine for traditional code. For modules, it's a problem. Each translation unit can have different `-fmodule-file=` arguments that tell the compiler where to find precompiled modules. If clangd picks the wrong translation unit, it gets the wrong module file mappings. You'll see red squiggles about unknown modules or missing symbols in your editor, even though `clang` compiles everything without errors.

The workaround: make sure each header has a corresponding `.cpp` file, even if all it does is include the header. This gives clangd a reliable flag source. The compiler doesn't need this, it's purely to guide the heuristic.

## Problem 3: import through #include doesn't propagate

This is a [bug in clangd](https://github.com/llvm/llvm-project/issues/181770), fixed in [llvm/llvm-project#189284](https://github.com/llvm/llvm-project/pull/189284). Consider this setup:

A module:

```cpp
// M.cppm
export module M;
export struct MyType {};
```

A header that imports it:

```cpp
// Header.h
import M;
```

A translation unit that includes the header and uses the imported type:

```cpp
// Use.cpp
#include "Header.h"
void use() {
  MyType t;  // clangd errors here
}
```

This builds fine with clang:

```bash
clang -std=c++20 M.cppm --precompile -o M.pcm
clang -fprebuilt-module-path=. -std=c++20 -I . -c Use.cpp  # OK
```

But clangd reports:

```
declaration of 'MyType' must be imported from module 'M' before it is required
```

The root cause is clangd's preamble build. clangd precompiles the preamble (the `#include` and `import` directives at the top of the file) into a PCH for performance. But precompiled headers and C++20 modules don't play well together, and `import` directives inside `#include`d headers get lost during preamble construction. The result is that clangd acts as if the import never happened for that translation unit.

[llvm/llvm-project#189284](https://github.com/llvm/llvm-project/pull/189284) should fix this. Until it lands in your clangd version, the workaround is to add a redundant `import` in the file that does the `#include`:

```cpp
// Use.cpp
#include "Header.h"
import M;  // redundant, but keeps clangd happy
void use() {
  MyType t;  // no more false error
}
```

## Problem 4: ccache doesn't track PCM content

This one isn't clangd, it's [ccache](https://github.com/ccache/ccache). If you're using ccache (and you probably are if you care about build times), it doesn't correctly invalidate caches when a precompiled module changes.

Here's why. ccache hashes the source content and all compiler arguments to build a cache key. Arguments like `-fmodule-file=mod=mod.pcm` are hashed as strings, so the _path_ to the PCM is part of the key. But `import` is a language-level construct, not a preprocessor directive, so ccache has no way to discover the PCM as an input file. The _content_ of the PCM is never hashed. When you modify a module interface and rebuild the PCM, ccache still returns the old cached result.

In our case, the result was stale object files with wrong struct layouts. The consumer `.o` had offsets from the old module interface but linked against the new module object, causing silent memory corruption at runtime. Your mileage may vary, you might get linker errors, wrong behaviour, or nothing obvious until much later. It's the kind of bug where `make clean && make` _doesn't_ fix it (because the stale entry is in ccache, not the build directory), and a colleague with a different cache state can't reproduce it. Peak "works on my machine."

For now, the workaround is to disable ccache for translation units that consume modules, or to always clean module consumers when a module interface changes. Neither is great.

At first glance the fix seems simple: recognise `-fmodule-file=` arguments, extract the PCM path, and hash the file's content. But proper C++20 module support in ccache is a much bigger problem. There's an [open PR](https://github.com/ccache/ccache/pull/1523) that handles P1689 dependency scanning, BMI serialization, and support across GCC, Clang, and MSVC. It's a significant effort and not yet merged.

## Problem 5: clang-tidy needs module artifacts to exist

Unlike clangd, clang-tidy expects the module artifacts to already be on disk. It won't build them for you. Without them, clang-tidy fails to resolve `import` statements.

You need to build the `.cppm.o`/`.modmap` files before running clang-tidy. I believe BMI (Binary Module Interface) files are also needed, though I haven't tracked down the exact root cause. With ninja:

```bash
# Build .cppm.o and .modmap files (listed as inputs in the ninja graph)
ninja -C build -t inputs \
  | grep -E '\.cppm\.o$|\.o\.modmap$' \
  | xargs -r ninja -C build

# Build .bmi files (build targets that clang-tidy needs to resolve imports)
ninja -C build -t targets all \
  | grep -oP '^[^:]*\.bmi(?=:)' \
  | xargs -r ninja -C build
```

## Problem 6: clang-tidy fix conflicts with modules

Even once clang-tidy can resolve modules, there's another issue. When a header is included from both a regular translation unit and a module's global module fragment, clang-tidy can see the same header twice in the same run. If a check wants to apply a fix (like `performance-trivially-destructible` suggesting `= default`), it generates two conflicting replacements for the same location. The result:

```
Fix conflicts with existing fix! The new replacement overlaps with an existing replacement.
```

Here's a minimal reproducer. A header with a trivially-destructible type:

```cpp
// counter.hpp
#pragma once
#include <atomic>
struct X {
  X() : a(0) {}
  std::atomic<int> a;
};
```

A module that includes it in the global module fragment:

```cpp
// mymodule.cppm
module;
#include "counter.hpp"
export module mymodule;
```

And a translation unit that includes the header _and_ imports the module:

```cpp
// main.cpp
#include "counter.hpp"
import mymodule;
int main() { return 0; }
```

Running `clang-tidy -checks='-*,performance-trivially-destructible' -p build main.cpp` triggers the conflict. I filed this as [llvm/llvm-project#178102](https://github.com/llvm/llvm-project/issues/178102) and it's been fixed in [llvm/llvm-project#178471](https://github.com/llvm/llvm-project/pull/178471).

The workaround before the fix lands in your toolchain: suppress individual checks that produce conflicting fixes, or avoid mixing `#include` and `import` for the same header in one translation unit (which isn't always possible).

## Problem 7: CMake 4.x dependency cycles with module targets

CMake allows dependency cycles between `STATIC_LIBRARY` targets, but when you add `FILE_SET CXX_MODULES`, CMake creates a synthetic `INTERFACE_LIBRARY` target that can't participate in cycles. Older versions incorrectly allowed this. CMake 4.x ([CMP0189](https://cmake.org/cmake/help/latest/policy/CMP0189.html)) correctly rejects it.

If your codebase has static library cycles, adding modules to any target in those cycles will force you to untangle them first.

## The bigger picture

None of these are compiler bugs. Clang handles C++20 modules correctly. The pain is in the tooling layer, and it's getting better. Every issue I've filed has had engaged, helpful responses from maintainers. The clang-tidy fix is already merged. The clangd preamble fix is landing. The CMake team is aware of the module metadata gap.

C++20 modules are worth trying. But the ecosystem needs people willing to use them in real codebases, hit the rough edges, and report them. Minimal reproducers go a long way. If you're considering modules, don't wait for everything to be perfect. Try it, file issues, and help move things forward.

If you've hit other module tooling problems, or found better workarounds, I'd love to hear about them. Find me on [Mastodon](https://fosstodon.org/@glloyd).
